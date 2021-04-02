---
description: 'https://github.com/forem/forem/issues/13226 captures the follow up to this.'
---

# Travis CI Segfaults

We've been seeing a large number of segmentation faults in travis CI when running the test suite.

[https://travis-ci.com/github/forem/forem/jobs/492228371](https://travis-ci.com/github/forem/forem/jobs/492228371)

```text
bundle exec rspec --format progress  --default-path spec "spec/models/user_spec.rb" "spec/requests/notifications_spec.rb" "spec/services/feeds/import_spec.rb" "spec/system/comments/user_fills_out_comment_spec.rb" "spec/services/articles/feeds/large_forem_experimental_spec.rb" "spec/models/article_spec.rb"


$ bin/knapsack_pro_rspec

I, [2021-03-19T06:41:01.310495 #8488]  INFO -- : [knapsack_pro] To retry in development the subset of tests fetched from API queue please run below command on your machine. If you use --order random then remember to add proper --seed 123 that you will find at the end of rspec command.

I, [2021-03-19T06:41:01.310577 #8488]  INFO -- : [knapsack_pro] bundle exec rspec --format progress  --default-path spec "spec/models/user_spec.rb" "spec/requests/notifications_spec.rb" "spec/services/feeds/import_spec.rb" "spec/system/comments/user_fills_out_comment_spec.rb" "spec/services/articles/feeds/large_forem_experimental_spec.rb" "spec/models/article_spec.rb"

DEPRECATION WARNING: Devise::Models::Authenticatable::BLACKLIST_FOR_SERIALIZATION is deprecated! Use Devise::Models::Authenticatable::UNSAFE_ATTRIBUTES_FOR_SERIALIZATION instead. (called from const_get at /home/travis/build/forem/forem/vendor/cache/devise-0cd72a56f984/lib/devise/models.rb:90)

[Zonebie] Setting timezone: ZONEBIE_TZ="Arizona"

...................................................................................................................................................................................................................................................................................................../home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/actionpack-6.0.3.5/lib/action_controller/log_subscriber.rb:16: [BUG] Segmentation fault at 0x0000000000000000

ruby 2.7.2p137 (2020-10-01 revision 5445e04352) [x86_64-linux]

-- Control frame information -----------------------------------------------

c:0107 p:---- s:0764 e:000763 CFUNC  :inspect

c:0106 p:---- s:0761 e:000760 CFUNC  :inspect

c:0105 p:0137 s:0757 e:000754 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/actionpack-6.0.3.5/lib/action_controller/log_subscriber.rb:16

c:0104 p:0038 s:0747 e:000746 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.0.3.5/lib/active_support/subscriber.rb:145

c:0103 p:0015 s:0738 e:000737 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.0.3.5/lib/active_support/log_subscriber.rb:107

c:0102 p:0011 s:0730 e:000729 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.0.3.5/lib/active_support/notifications/fanout.rb:1

c:0101 p:0011 s:0723 e:000722 BLOCK  /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.0.3.5/lib/active_support/notifications/fanout.rb:6 [FINISH]

c:0100 p:---- s:0719 e:000718 CFUNC  :each

c:0099 p:0012 s:0715 e:000714 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.0.3.5/lib/active_support/notifications/fanout.rb:6

c:0098 p:0014 s:0707 e:000706 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.0.3.5/lib/active_support/notifications/instrumente

c:0097 p:0035 s:0700 e:000698 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.0.3.5/lib/active_support/notifications/instrumente

c:0096 p:0023 s:0691 e:000690 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.0.3.5/lib/active_support/notifications.rb:180

c:0095 p:0074 s:0685 e:000684 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/actionpack-6.0.3.5/lib/action_controller/metal/instrumentation.rb:

c:0094 p:0017 s:0679 e:000678 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/actionpack-6.0.3.5/lib/action_controller/metal/params_wrapper.rb:2

c:0093 p:0026 s:0674 e:000673 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/activerecord-6.0.3.5/lib/active_record/railties/controller_runtime

c:0092 p:0077 s:0668 e:000667 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/actionpack-6.0.3.5/lib/abstract_controller/base.rb:136

c:0091 p:0062 s:0661 e:000660 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/actionview-6.0.3.5/lib/action_view/rendering.rb:39

c:0090 p:0017 s:0655 e:000654 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/actionpack-6.0.3.5/lib/action_controller/metal.rb:190

c:0089 p:0034 s:0648 e:000647 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/actionpack-6.0.3.5/lib/action_controller/metal.rb:254

c:0088 p:0010 s:0641 e:000640 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/actionpack-6.0.3.5/lib/action_dispatch/routing/route_set.rb:50

c:0087 p:0036 s:0633 e:000632 METHOD /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/actionpack-6.0.3.5/lib/action_dispatch/routing/route_set.rb:33

c:0086 p:0111 s:0625 e:000624 BLOCK  /home/travis/build/forem/forem/vendor/bundle/ruby/2.7.0/gems/actionpack-6.0.3.5/lib/action_dispatch/journey/router.rb:49 [FINISH]

c:0085 p:---- s:0613 e:000612 CFUNC  :each
```

Counting the dots \(successful test cases\) puts us 293 tests into the rspec run \(out of 516\)

Step 1 - I know at some point I'm going to need to get closer to travis's stack to inspect this - but locally I might be able to put a breakpoint in the logger where the failures are occuring \(they always happen at line 16 calling inspect\) - I'm cutting the list of specs run down to see where it is - first guess is split between import spec and user fills out comment spec \(system test would be exercising puma\).

```text

djuber@laptop:~/src/forem$ bundle exec rspec --format progress  --default-path spec "spec/models/user_spec.rb" "spec/requests/notifications_spec.rb" "spec/services/feeds/import_spec.rb"
DEPRECATION WARNING: Devise::Models::Authenticatable::BLACKLIST_FOR_SERIALIZATION is deprecated! Use Devise::Models::Authenticatable::UNSAFE_ATTRIBUTES_FOR_SERIALIZATION instead. (called from const_get at /data/src/forem/vendor/cache/devise-0cd72a56f984/lib/devise/models.rb:90)
[Zonebie] Setting timezone: ZONEBIE_TZ="Vienna"
...........................................................................................
...........................................................................................
...........................................................................................
....................                                                                      

Finished in 2 minutes 7.3 seconds (files took 3.59 seconds to load)
293 examples, 0 failures
```

Perfect! 293 gets me to the _end_ of the feeds import spec, or the first test in the `user_fills_out_comment_spec.rb`

This means the first time this instance runs a system test \(using the full stack, puma http server and chrome headless browser\) we hit this error. Sadly I don't expect to be able to reproduce this locally...







