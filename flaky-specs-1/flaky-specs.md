---
description: Interaction between dbcleaner and seeded values in subsequent specs
---

# Flaky Specs \#1

I'm looking at test suite reliability \(one of my many hats\).

One came up yesterday in the context of a build \(and a rebuild cleared it suggesting the issue was isolated to the test ordering and interaction\).

[https://travis-ci.com/github/forem/forem/jobs/491785877\#L953](https://travis-ci.com/github/forem/forem/jobs/491785877#L953) was the failing component

Seeing this I tried to reproduce it locally \(I just copied the whole chunk of files onto the command line and ran it\) - I got the same two failures as the reporter.

At that point, I was able to open an issue [https://github.com/forem/forem/issues/13030](https://github.com/forem/forem/issues/13030) capturing this - and the less than stimulating bisection quest began.

RSpec's `--bisect` option is pretty clever - given some failing spec list, it first runs the specs completely to verify the issue is reproduced \(and separates the failures from the successes\)

It runs a second test to see if this appears to be order dependent \(this failed for me a few times while I was working, and I had to do the first few bisections manually\). If that works, it does subsequent halving \(bisection search\) of the problem until it can offer a minimum failing ordering. It works on the example, not file level, so the output is often a little _too_ granular, and in my case it came in the reverse order, however it did produce the right set:

```text
bundle exec rspec spec/requests/video_states_update_spec.rb  spec/models/display_ad_spec.rb spec/serializers/search/comment_serializer_spec.rb spec/lib/data_update_scripts/cleanup_orphan_git_hub_repositories_spec.rb  spec/services/html/parser_spec.rb  spec/services/webhook/dispatch_event_spec.rb spec/requests/api_secrets_create_spec.rb spec/liquid_tags/listing_tag_spec.rb  spec/services/edge_cache/bust_comment_spec.rb spec/policies/follow_policy_spec.rb spec/services/profile_fields/import_from_csv_spec.rb spec/services/edge_cache/bust_organization_spec.rb spec/system/admin/admin_creates_new_page_spec.rb spec/lib/app_secrets_spec.rb spec/routing/all_routes_spec.rb spec/requests/sidebars_spec.rb spec/tasks/fastly_spec.rb spec/system/feedback_message_spec.rb spec/models/ahoy/visit_spec.rb spec/views/layouts/signup_modal.html.erb_spec.rb spec/liquid_tags/podcast_tag_spec.rb spec/policies/api_secret_policy_spec.rb spec/requests/user/user_profile_spec.rb spec/system/authentication/user_logs_in_with_apple_spec.rb spec/mailers/notify_mailer_spec.rb spec/services/analytics_service_spec.rb spec/requests/articles/articles_spec.rb  spec/requests/comments_create_spec.rb spec/services/search/feed_content_spec.rb  spec/system/homepage/user_visits_homepage_articles_spec.rb  spec/system/dashboards/user_sorts_dashboard_articles_spec.rb spec/system/homepage/user_visits_homepage_with_announcement_spec.rb spec/workers/users/record_field_test_event_worker_spec.rb  spec/system/homepage/user_visits_homepage_spec.rb spec/system/user_views_a_reading_list_spec.rb  spec/system/authentication/user_logs_in_with_email_spec.rb  spec/requests/api/v0/organizations_spec.rb  spec/decorators/user_decorator_spec.rb  spec/models/profile_pin_spec.rb  spec/lib/data_update_scripts/fix_profile_field_edge_cases_spec.rb --bisect
Running suite to find failures... (2 minutes 3.1 seconds)

Starting bisect with 1 failing example and 505 non-failing examples.

Checking that failure(s) are order-dependent... failure appears to be order-dependent
Round 1: bisecting over non-failing examples 1-505 .. ignoring examples 1-253 (1 minute 40.72 seconds)
Round 2: bisecting over non-failing examples 254-505 .. ignoring examples 254-379 (1 minute 28.21 seconds)
Round 3: bisecting over non-failing examples 380-505 .. ignoring examples 443-505 (1 minute 48.83 seconds)
Round 4: bisecting over non-failing examples 380-442 .. ignoring examples 380-411 (35.28 seconds)
Round 5: bisecting over non-failing examples 412-442 .. ignoring examples 428-442 (52.8 seconds)
Round 6: bisecting over non-failing examples 412-427 ..
```

One thing to factor in is time - since this is not running in parallel from what I can see - you'll want to capture as small a set up front as you can \(in my case, I was able to omit all of the files listed after the first failing test, since I knew these ran in order instead of randomized\), as you'll run every example multiple times.



### Outcome

it turns out the underlying problem was an interaction between some seed data that's imported during `spec_helper` or `rails_helper` being loaded \(this happens twice, once in the end to end setup, once in the general unit test setup\) from a csv file \(outside of the normal "seed data" format\), and being cleared by one of two tests using a `:truncation` db cleaner strategy. A little looking around led me to find work done last June to introduce db cleaner and use truncation \(instead of transactional guarantees\) due to a system test hanging in a create statement during CI test runs. 

I'm in the process of removing that since it might actually be worse than an occassional hang - I'm also running the import feed spec locally 100 times to observe it completes in the same time consistently without hanging, so far so good, but I think this may only manifest in travis when running the spec suite.

[https://github.com/forem/forem/pull/13035](https://github.com/forem/forem/pull/13035)

