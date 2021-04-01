---
description: Dealing with Flaky specs in an efficient manner
---

# Tracking down spec failures

Since we're using knapsack pro in queue mode - the reproduction set from a failed build is actually just the individual step size \(rather than the reported total set at the end\) which makes bisection a lot more tractable.

For example, one developer reported a test failure here [https://travis-ci.com/github/forem/forem/jobs/489033830\#L1016](https://travis-ci.com/github/forem/forem/jobs/489033830#L1016) and looking at the travis output I can see the last block \(line 1035 after the failures\) showing _all_ of the tests run in this container, however the sequential queue provided test chunks run rspec independently \(the full setup/seed/takedown occurs per batch, so the suite run in sequence is more or less unimportant\)

I scrolled further up into the batch executions to [https://travis-ci.com/github/forem/forem/jobs/489033830\#L963-L967](https://travis-ci.com/github/forem/forem/jobs/489033830#L963-L967) to find the small set of tests that were run that did fail \(looking for an F in place of a `.` , `*` is a pending/skipped test and is not a failing case so okay to ignore those if you see them\)

I copied _that_ list to a local console and tried to reproduce \(but this did not fail reliably for me\), it's possible the underlying condition has since been fixed 

![](../.gitbook/assets/image.png)

```text


djuber@laptop:~/src/forem$ bundle exec rspec --format progress  --default-path spec "spec/services/mentions/create_all_spec.rb" "spec/requests/api/v0/webhooks_spec.rb" "spec/system/search/display_jobs_banner_spec.rb" "spec/requests/admin/reactions_spec.rb" "spec/requests/response_templates_spec.rb" "spec/services/rate_limit_checker_spec.rb" "spec/queries/admin/users_query_spec.rb" "spec/helpers/authentication_helper_spec.rb" "spec/requests/admin/sponsorships_spec.rb" "spec/services/notifications/new_follower/send_spec.rb" "spec/requests/articles/articles_create_spec.rb" "spec/requests/admin/articles_spec.rb" "spec/models/follow_spec.rb" "spec/requests/admin/mods_spec.rb" "spec/system/podcasts/user_visits_podcasts_root_page_spec.rb" "spec/services/search/user_spec.rb" "spec/requests/user_blocks_spec.rb" "spec/system/authentication/creator_config_edit_spec.rb" "spec/services/articles/suggest_spec.rb" "spec/requests/admin/response_templates_spec.rb" "spec/models/message_spec.rb" "spec/models/identity_spec.rb" "spec/requests/user/user_show_spec.rb"
DEPRECATION WARNING: Devise::Models::Authenticatable::BLACKLIST_FOR_SERIALIZATION is deprecated! Use Devise::Models::Authenticatable::UNSAFE_ATTRIBUTES_FOR_SERIALIZATION instead. (called from const_get at /data/src/forem/vendor/cache/devise-0cd72a56f984/lib/devise/models.rb:90)
DEPRECATION WARNING: Devise::Models::Authenticatable::BLACKLIST_FOR_SERIALIZATION is deprecated! Use Devise::Models::Authenticatable::UNSAFE_ATTRIBUTES_FOR_SERIALIZATION instead. (called from const_get at /data/src/forem/vendor/cache/devise-0cd72a56f984/lib/devise/models.rb:90)
[Zonebie] Setting timezone: ZONEBIE_TZ="Monrovia"
...........................................................................................
...........................................................................................
.......................................................................................   
Finished in 1 minute 12.91 seconds (files took 11.55 seconds to load)
269 examples, 0 failures

djuber@laptop:~/src/forem$ bundle exec rspec --format progress  --default-path spec "spec/services/mentions/create_all_spec.rb" "spec/requests/api/v0/webhooks_spec.rb" "spec/system/search/display_jobs_banner_spec.rb" "spec/requests/admin/reactions_spec.rb" "spec/requests/response_templates_spec.rb" "spec/services/rate_limit_checker_spec.rb" "spec/queries/admin/users_query_spec.rb" "spec/helpers/authentication_helper_spec.rb" "spec/requests/admin/sponsorships_spec.rb" "spec/services/notifications/new_follower/send_spec.rb" "spec/requests/articles/articles_create_spec.rb" "spec/requests/admin/articles_spec.rb" "spec/models/follow_spec.rb" "spec/requests/admin/mods_spec.rb" "spec/system/podcasts/user_visits_podcasts_root_page_spec.rb" "spec/services/search/user_spec.rb" "spec/requests/user_blocks_spec.rb" "spec/system/authentication/creator_config_edit_spec.rb" "spec/services/articles/suggest_spec.rb" "spec/requests/admin/response_templates_spec.rb" "spec/models/message_spec.rb" "spec/models/identity_spec.rb" "spec/requests/user/user_show_spec.rb"
DEPRECATION WARNING: Devise::Models::Authenticatable::BLACKLIST_FOR_SERIALIZATION is deprecated! Use Devise::Models::Authenticatable::UNSAFE_ATTRIBUTES_FOR_SERIALIZATION instead. (called from const_get at /data/src/forem/vendor/cache/devise-0cd72a56f984/lib/devise/models.rb:90)
[Zonebie] Setting timezone: ZONEBIE_TZ="Zurich"
...........................................................................................
...........................................................................................
.......................................................................................   
Finished in 1 minute 1.85 seconds (files took 3.63 seconds to load)
269 examples, 0 failures
```

Importantly - this cuts the time to validate down to a minute or two and will permit the time to bisect being comparable lower as well. This "optimization" was something I missed the first time I approached this \(in Flaky Specs \#1\)

