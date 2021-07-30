# Tricking KnapsackPro

In "real CI" forem uses knapsack pro's queue mode to receive a balance of tests. In our "local" mode we just bypass this which unfortunately runs all the tests. I want to run specific tests in order.



We make the choice here \(if you don't have the api token, we'll bypass the goodness and just run all the tests using the fallback from knapsacks rspec runner\).

{% embed url="https://github.com/forem/forem/blob/main/bin/knapsack\_pro\_rspec\#L2-L7" %}

The rspec runner we end up using without the token is here

{% embed url="https://github.com/KnapsackPro/knapsack\_pro-ruby/blob/d23365af604e0c2feb5243e738f521818191e88c/lib/knapsack\_pro/runners/rspec\_runner.rb" %}

What I want is to use this Queue::Runner instead - but to fake out the queue to replay the specific tests in order from a travis execution log

{% embed url="https://github.com/KnapsackPro/knapsack\_pro-ruby/blob/d23365af604e0c2feb5243e738f521818191e88c/lib/knapsack\_pro/runners/queue/rspec\_runner.rb" %}

Example log [https://api.travis-ci.com/v3/job/528007127/log.txt](https://api.travis-ci.com/v3/job/528007127/log.txt)



