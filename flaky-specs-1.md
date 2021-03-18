---
description: 'how marking a spec `:flaky` works at forem'
---

# Flaky Specs

It seems the shortcut to automatically permitting retry of a job known to potentially fail is to tag it flaky, as declared at [https://github.com/forem/forem/blob/master/spec/rails\_helper.rb\#L117-L120](https://github.com/forem/forem/blob/master/spec/rails_helper.rb#L117-L120)

```ruby
  config.around(:each, :flaky) do |ex|
    ex.run_with_retry retry: 3
  end

```

Essentially - when a test fails a developer can mark it :flaky, rspec will automatically retry it up to 3 times without raising an error.

To clean this up \(make flaky tests solid once again\) would a valid approach be to remove the flaky tag and run the specs and see what falls out? 

I guess I could just isolate the test suite to run the flaky ones with a tag? I'll start there and see what happens.

```ruby
djuber@laptop:/data/src/forem$ bundle exec rspec --format=documentation --order=random --tag=flaky spec/                                                                              DEPRECATION WARNING: Devise::Models::Authenticatable::BLACKLIST_FOR_SERIALIZATION is deprecated! Use Devise::Models::Authenticatable::UNSAFE_ATTRIBUTES_FOR_SERIALIZATION instead. (called from const_get at /data/src/forem/vendor/cache/devise-0cd72a56f984/lib/devise/models.rb:90)
WARNING: Shared example group 'UserSubscriptionSourceable' has been previously defined at:
  /data/src/forem/spec/models/shared_examples/user_subscription_sourceable_spec.rb:1
...and you are now defining it at:
  /data/src/forem/spec/models/shared_examples/user_subscription_sourceable_spec.rb:1
The new definition will overwrite the original one.
Run options: include {:flaky=>true}

Randomized with seed 43087
[Zonebie] Setting timezone: ZONEBIE_TZ="Baku"

Creating an article with the editor
  creates a new article

Articles::Suggest
  returns the number of articles requested
  returns proper number of articles with post with the same tags
  returns proper number of articles with post with different tags

User visits podcast show page
  they see the content of the hero

Finished in 15.66 seconds (files took 10 seconds to load)
5 examples, 0 failures

Randomized with seed 43087

```

I was underwhelmed when I ran this since my first search covered the installed gems in vendor/ as well, I only want to look for _our_ test suite and that lines up with the run above.

```bash
djuber@laptop:/data/src/forem$ git grep :flaky spec/
spec/rails_helper.rb:  config.around(:each, :flaky) do |ex|
spec/services/articles/suggest_spec.rb:  it "returns proper number of articles with post wi
th the same tags", :flaky do                                                              
spec/services/articles/suggest_spec.rb:  it "returns proper number of articles with post wi
th different tags", :flaky do                                                             
spec/services/articles/suggest_spec.rb:  it "returns the number of articles requested", :fl
aky do                                                                                    
spec/system/articles/user_creates_an_article_spec.rb:  it "creates a new article", :flaky, 
js: true do                                                                               
spec/system/podcasts/user_visits_podcast_episode_spec.rb:  it "they see the content of the 
hero", :flaky, js: true do     
```

To try and simulate the flakiness I'm going to edit the rails helper as follows:

```ruby
  config.around(:each, :flaky) do |ex|
    ex.run
    # ex.run_with_retry retry: 3
  end
```

basically - just ignore this by calling the example directly with no retry count if it's tagged flaky, then I can isolate the failures by re-running the tag scoped examples.

