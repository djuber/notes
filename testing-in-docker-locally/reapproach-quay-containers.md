---
description: >-
  Part 2 in an N part series where I try to get our forem:testing containers to
  run the test suite
---

# Reapproach quay containers

Jotting down a few notes before I get started. Talked earlier this week with Joe about how they overcame volume mount and user permissions issues for the forem self-host instances \(Digital ocean droplet running podman running thes quay.io/forem/forem:latest image plus supporting services\).

{% hint style="info" %}
"quay containers" is a horrible name for this - what I mean are the containers I can download now from quay.io/forem/forem that are built and tagged by buildkite both during PR branches and master builds. This is important because these are also the containers we deploy \(in forem cloud, and likely in self-hosting\). S
{% endhint %}

One thing that needed/needs to happen is removing the root user and using the forem user \(sounds like a docker-compose change- the container file already builds a forem user, we weren't using that\). - Snapped this commit together during the discussion [https://github.com/djuber/forem/commit/36dc3c56a1f18351983bd038f28e49c60ff016be](https://github.com/djuber/forem/commit/36dc3c56a1f18351983bd038f28e49c60ff016be)

My last series of tests would copy the tree into the image during build \(so any code changes locally required a rebuild of the image, which wasn't a problem as long as just running the container was the task at hand, but could be a big hassle for live use/reuse\).

My problems with port vs expose are still valid \(I think I want to change to expose basically for everything in testing - unless there's value in seeing port 3000 I shouldn't _need_ to stop the local dev env to interact with docker\). 

How do I isolate the "testing" stage in the container file?

```text
docker build --target="testing" . 
```

I think this might be correct.

Change 1:

make the entrypoint call rspec instead of rails server - I remember this being a weird decision when I was using docker compose to run the tests because of how the output happened

```text

-CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0", "-p", "3000"]
+CMD ["bundle", "exec", "rspec"]
 
```

Did some minor surgery on the test compose file and I'm able to get errors running tests now successfully

{% embed url="https://github.com/djuber/forem/tree/djuber/quay-images-for-testing" %}

Commented out the profile image from the users factory and I'm down to a few hundred failures \(out of 6000\) so \(1\) carrierwave is still an impediment, mostly tied to user factory, \(2\) it's the main problem to solve here.

```text
$ docker-compose -f test-compose.yml run rails bundle exec rspec --format=documentation

```

My next step will be to add a single carrierwave test case "it saves profile images" and assert I can do that - and use that as the indicator for testing.

![Two feedback loops - lets move into the small one for now](../.gitbook/assets/image%20%282%29.png)

In the meantime - with just a comment on the profile image attribute in the users factory - I can get the specs to run more or less \(some other attached images fail, a few tests actually act on it, there are other problems\) - but we're looking at a few hundred rather than four thousand failures \(when `create(:user)` fails lots of scenarios go belly up\). 

```text
Finished in 13 minutes 3 seconds (files took 7.42 seconds to load)
6030 examples, 965 failures, 28 pending
```

Apart from carrierwave - the same issue as earlier \(chrome binary not present in the image\) presents - we may want to filter out feature tests and run them separately anyway?

```ruby
  describe "profile_image" do
    let(:image_path) { Rails.root.join("spec/support/fixtures/images/image1.jpeg") }
    let(:user) { create(:user) }

    it 'has a profile image' do
      user.profile_image = Rack::Test::UploadedFile.new(image_path, "image/jpeg")
      expect(user.valid?).to be true
    end
  end

```

This fails \(for the same reason you would expect - the `profile_image` is `#<ProfileImageUploader:0x00007f076ab798f0 @model=#<User id: 13362, apple_created_at: nil, apple_usern...=nil, @cache_id=nil, @identifier=nil, @versions=nil, @versions_to_cache=nil, @versions_to_store=nil>` and that's not valid \(image is nil, so image type is nil\)



Suddenly started seeing issues with module autoloading around Notifications \(which makes far less sense as a problem\).



### Part 2

I had some time to reapproach this today. As an experiment, I first tried changing the volume: to a mount: \(bind mount\) by updating the docker compose file. This seems like it was a dead end \(worthwhile experiment, but possibly unimportant\).



Second experiment was to change the `Rack::Test::UploadedFile.new` calls in the factories to `File.open()` calls. This appears to have worked \(notably, I don't see a /tmp file created by rack, and _do_ see the file in `public/uploads/users/profile_image/userid` but this feels like I've sidestepped the issue rather than understanding it.



### Part 3

I'm going to just throw this week at the problem

I removed the edit to the users factory \(restoring the original rack test uploaded file image\).

User.new\(profile\_image: Rack::Test::UploadedFile\) creates a new tempfile per user \(i.e. I pass in an uploaded file that has a tmpfile already, during processing - before anything goes into the upload directory - another tempfile is created - user model fails validation \(initial error was wrong mime type - but removing the minimagick include from base uploader causes the error to change to "must be at least 1 byte" and the `profile_image` has no associated file:

```ruby
image_path = Rails.root.join("spec/support/fixtures/images/image1.jpeg")                                                               
=> #<Pathname:/opt/apps/forem/spec/support/fixtures/images/image1.jpeg>

profile_image = Rack::Test::UploadedFile.new(image_path, "image/jpeg")                                                                           
=> #<Rack::Test::UploadedFile:0x00007f3528912c88
 @content_type="image/jpeg",
 @original_filename="image1.jpeg",
 @tempfile=#<File:/tmp/image120210511-18-ssk9ua.jpeg>>

user = User.new(
  name: Faker::Name.name,
  email: Faker::Internet.email,      
  username: "User0001",              
  profile_image: Rack::Test::UploadedFile.new(Rails.root.join("spec/support/fixtures/images/image1.jpeg"), "image/jpeg"),                                   
  confirmed_at: Time.now,      
  saw_onboarding: true,        
  checked_code_of_conduct: true,      
  registered_at: Time.now)              
=> #<User id: nil, apple_username: nil, articles_count: 0, badge_achievements_count: 0, blocked_by_count: 0, blocking_others_count: 0, checked_code_of_conduct: true, checked_terms_and_conditions: false, comments_count: 0, config_font: "default", config_navbar: "default", config_theme: "default", created_at: nil, credits_count: 0, display_announcements: true, display_sponsors: true, editor_version: "v1", email: "agustin@skiles-hand.biz", email_badge_notifications: true, email_comment_notifications: true, email_community_mod_newsletter: false, email_connect_messages: true, email_digest_periodic: false, email_follower_notifications: true, email_membership_newsletter: false, email_mention_notifications: true, email_newsletter: false, email_tag_mod_newsletter: false, email_unread_notifications: true, experience_level: nil, export_requested: false, exported_at: nil, facebook_username: nil, feed_fetched_at: "2017-01-01 00:00:00.000000000 -0500", feed_mark_canonical: false, feed_referential_link: true, feed_url: nil, following_orgs_count: 0, following_tags_count: 0, following_users_count: 0, github_repos_updated_at: "2017-01-01 00:00:00.000000000 -0500", github_username: nil, inbox_guidelines: nil, inbox_type: "private", last_article_at: "2017-01-01 00:00:00.000000000 -0500", last_comment_at: "2017-01-01 00:00:00.000000000 -0500", last_followed_at: nil, last_moderation_notification: "2017-01-01 00:00:00.000000000 -0500", last_notification_activity: nil, last_onboarding_page: nil, last_reacted_at: nil, latest_article_updated_at: nil, mobile_comment_notifications: true, mod_roundrobin_notifications: true, monthly_dues: 0, name: "Miss Wilmer Stokes", old_old_username: nil, old_username: nil, onboarding_package_requested: false, organization_info_updated_at: nil, payment_pointer: nil, permit_adjacent_sponsors: true, profile_image: nil, profile_updated_at: "2017-01-01 00:00:00.000000000 -0500", rating_votes_count: 0, reaction_notifications: true, reactions_count: 0, registered: true, registered_at: "2021-05-11 10:51:35.331809736 -0400", reputation_modifier: 1.0, saw_onboarding: true, score: 0, secret: nil, signup_cta_variant: nil, spent_credits_count: 0, stripe_id_code: nil, subscribed_to_user_subscriptions_count: 0, twitter_username: nil, unspent_credits_count: 0, updated_at: nil, username: "User0001", welcome_notifications: true, workshop_expiration: nil>

user.valid?                                                                                                                                                                                                               
=> false
user.errors.messages                                      
=> {:profile_image=>["File size should be greater than 1 Byte"]}      

user.profile_image                                                                                       
=> #<ProfileImageUploader:0x00007f3524f9ed08
 @cache_id=nil,
 @file=nil,
 @filename=nil,
 @identifier=nil,
 @model=
  #<User id: nil, apple_username: nil, articles_count: 0, badge_achievements_count: 0, blocked_by_count: 0, blocking_others_count: 0, checked_code_of_conduct: true, checked_terms_and_conditions: false, comments_count: 0, config_font: "default", config_navbar: "default", config_theme: "default", created_at: nil, credits_count: 0, display_announcements: true, display_sponsors: true, editor_version: "v1", email: "agustin@skiles-hand.biz", email_badge_notifications: true, email_comment_notifications: true, email_community_mod_newsletter: false, email_connect_messages: true, email_digest_periodic: false, email_follower_notifications: true, email_membership_newsletter: false, email_mention_notifications: true, email_newsletter: false, email_tag_mod_newsletter: false, email_unread_notifications: true, experience_level: nil, export_requested: false, exported_at: nil, facebook_username: nil, feed_fetched_at: "2017-01-01 00:00:00.000000000 -0500", feed_mark_canonical: false, feed_referential_link: true, feed_url: nil, following_orgs_count: 0, following_tags_count: 0, following_users_count: 0, github_repos_updated_at: "2017-01-01 00:00:00.000000000 -0500", github_username: nil, inbox_guidelines: nil, inbox_type: "private", last_article_at: "2017-01-01 00:00:00.000000000 -0500", last_comment_at: "2017-01-01 00:00:00.000000000 -0500", last_followed_at: nil, last_moderation_notification: "2017-01-01 00:00:00.000000000 -0500", last_notification_activity: nil, last_onboarding_page: nil, last_reacted_at: nil, latest_article_updated_at: nil, mobile_comment_notifications: true, mod_roundrobin_notifications: true, monthly_dues: 0, name: "Miss Wilmer Stokes", old_old_username: nil, old_username: nil, onboarding_package_requested: false, organization_info_updated_at: nil, payment_pointer: nil, permit_adjacent_sponsors: true, profile_image: nil, profile_updated_at: "2017-01-01 00:00:00.000000000 -0500", rating_votes_count: 0, reaction_notifications: true, reactions_count: 0, registered: true, registered_at: "2021-05-11 10:51:35.331809736 -0400", reputation_modifier: 1.0, saw_onboarding: true, score: 0, secret: nil, signup_cta_variant: nil, spent_credits_count: 0, stripe_id_code: nil, subscribed_to_user_subscriptions_count: 0, twitter_username: nil, unspent_credits_count: 0, updated_at: nil, username: "user0001", welcome_notifications: true, workshop_expiration: nil>,
 @mounted_as=:profile_image,
 @staged=false,
 @versions=nil,
 @versions_to_cache=nil,
 @versions_to_store=nil>
 
 
```



