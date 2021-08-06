# Tricking KnapsackPro

In "real CI" forem uses knapsack pro's queue mode to receive a balance of tests. In our "local" mode we just bypass this which unfortunately runs all the tests. I want to run specific tests in order.



We make the choice here \(if you don't have the api token, we'll bypass the goodness and just run all the tests using the fallback from knapsacks rspec runner\).

{% embed url="https://github.com/forem/forem/blob/main/bin/knapsack\_pro\_rspec\#L2-L7" %}

The rspec runner we end up using without the token is here

{% embed url="https://github.com/KnapsackPro/knapsack\_pro-ruby/blob/d23365af604e0c2feb5243e738f521818191e88c/lib/knapsack\_pro/runners/rspec\_runner.rb" %}

What I want is to use this Queue::Runner instead - but to fake out the queue to replay the specific tests in order from a travis execution log

{% embed url="https://github.com/KnapsackPro/knapsack\_pro-ruby/blob/d23365af604e0c2feb5243e738f521818191e88c/lib/knapsack\_pro/runners/queue/rspec\_runner.rb" %}

Example log [https://api.travis-ci.com/v3/job/528007127/log.txt](https://api.travis-ci.com/v3/job/528007127/log.txt)



So what will this look like? Not sure but I'm going to hack at it for a while to see. I know what's roughly going on in our code \(we have modules that define accessors based on database state, and paths in the tests that mutate/reset those accessor bindings, and sometimes, in CI only, hit a place where we have problems resulting in "stack level too deep". While this often represents itself in system tests \(where capybara is viewing a rendered profile partial\) it also happens in model specs \(meaning it's not a front-end issue, definitely something with the class method lookup?\)

I want to get the tests to run in order so that I can then see what's happening when this occurs \(please don't mean I'm attaching a debugger to ruby\). Or to steal @b0rk's graphic from [wizard zines ](https://wizardzines.com/comics/minimal-reproduction/)

![](.gitbook/assets/image%20%283%29.png)

Distill the logs to an action list

Effectively this was just grepping for the `bundle exec rspec` lines that are logged by the test runner \(which are distinct from actually running this as separate process, which is evident when  we check the timing and try to replicate it\)

```text

[0K$ bin/knapsack_pro_rspec
I, [2021-07-30T14:57:59.658192 #9616]  INFO -- : [knapsack_pro] To retry in development the subset of tests fetched from API queue please run below command on your machine. If you use --order random then remember to add proper --seed 123 that you will find at the end of rspec command.
I, [2021-07-30T14:57:59.658280 #9616]  INFO -- : [knapsack_pro] bundle exec rspec --format progress  --default-path spec "spec/controllers/concerns/json_api_sort_param_spec.rb"
[Zonebie] Setting timezone: ZONEBIE_TZ="Marshall Is."
.....
```

From this I want to extract just the following 

```text
bundle exec rspec --format progress  --default-path spec "spec/controllers/concerns/json_api_sort_param_spec.rb"
```

we can do that all at once with `grep "bundle exec rspec " | cut -f 2 -d \]` \(the cut basically skips forward past the `[knapsack_pro]` log item and captures the rest of the line.

 

{% file src=".gitbook/assets/tests.txt" caption="extract test list as rspec commands" %}



Eventually I did something - I'm not sure this was necessary but at the time it made sense. I changed the Client::Connnection class to bypass the normal api behavior in call \(if it's a request to queues/ then return the next set of tests, if it's a request to any other endpoint return an empty body in a 200 response\)

```ruby
module KnapsackPro
  module Client
    class Connection
      def handle_queue
        @@step_number ||= 0
        response = responses[@@step_number] || []
        @@step_number += 1
        {"test_files" => response }
      end

      def responses
        @@responses ||=
        begin
          path = '/home/djuber/Documents/test-lines.txt'
          file = File.read(path); file.size
          tests = file.lines.map {|line| line.delete('"').split(' ') }
          tests.map {|testline| testline.map {|path| {"path" => path} } }
        end
      end

      def call
        if(@action.endpoint_path == '/v1/queues/queue')
          handle_queue
        else
          #send(action.http_method)
          @response_body = ''
          @http_response = Net::HTTPResponse.new("1.1", "200", "OK")
          response_body
        end
      end

      def success?
        return true
      end
    end
  end
end
```

I just modified the file in vendor/cache/ruby/2.7.0/gems/knapsack-pro-2.18.2/lib/knapsackpro/client/connection.rb rather than monkey patching this in a rake task \(which would probably be the smarter way to do this in the future if we had something that took log files or a travis url and replayed it locally.



It turns out I find the issue I was looking for - [https://github.com/forem/forem/pull/14433](https://github.com/forem/forem/pull/14433) - this is the "test-lines.txt" file used to drive this \(extracted from the travis log linked above\).

{% file src=".gitbook/assets/test-lines.txt" %}

I really tried to use the initializer to redefine these methods but it seemed like spring was not re-loading the initializers for me in local testing \(this would likely not be a problem in CI tests but was frustrating locally so I ended up re-pasting the method definitions into the vendor/cache copy of the connection.rb.

