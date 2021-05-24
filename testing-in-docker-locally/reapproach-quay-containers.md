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

Works fine when I pass in an open file object

```ruby
 user = User.new(
   name: Faker::Name.name,
   email: Faker::Internet.email,      
   username: "User0001",      
   profile_image: File.open(image_path) )
=> #<User id: nil, apple_username: nil, articles_count: 0, badge_achievements_count: 0, blocked_by_count: 0, blocking_others_count: 0, checked_code_of_conduct: false, checked_terms_and_conditions: false, comments_count: 0, config_font: "default", config_navbar: "default", config_theme: "default", created_at: nil, credits_count: 0, display_announcements: true, display_sponsors: true, editor_version: "v1", email: "joel_deckow@nicolas.io", email_badge_notifications: true, email_comment_notifications: true, email_community_mod_newsletter: false, email_connect_messages: true, email_digest_periodic: false, email_follower_notifications: true, email_membership_newsletter: false, email_mention_notifications: true, email_newsletter: false, email_tag_mod_newsletter: false, email_unread_notifications: true, experience_level: nil, export_requested: false, exported_at: nil, facebook_username: nil, feed_fetched_at: "2017-01-01 00:00:00.000000000 -0500", feed_mark_canonical: false, feed_referential_link: true, feed_url: nil, following_orgs_count: 0, following_tags_count: 0, following_users_count: 0, github_repos_updated_at: "2017-01-01 00:00:00.000000000 -0500", github_username: nil, inbox_guidelines: nil, inbox_type: "private", last_article_at: "2017-01-01 00:00:00.000000000 -0500", last_comment_at: "2017-01-01 00:00:00.000000000 -0500", last_followed_at: nil, last_moderation_notification: "2017-01-01 00:00:00.000000000 -0500", last_notification_activity: nil, last_onboarding_page: nil, last_reacted_at: nil, latest_article_updated_at: nil, mobile_comment_notifications: true, mod_roundrobin_notifications: true, monthly_dues: 0, name: "Pres. Janette D'Amore", old_old_username: nil, old_username: nil, onboarding_package_requested: false, organization_info_updated_at: nil, payment_pointer: nil, permit_adjacent_sponsors: true, profile_image: nil, profile_updated_at: "2017-01-01 00:00:00.000000000 -0500", rating_votes_count: 0, reaction_notifications: true, reactions_count: 0, registered: true, registered_at: nil, reputation_modifier: 1.0, saw_onboarding: true, score: 0, secret: nil, signup_cta_variant: nil, spent_credits_count: 0, stripe_id_code: nil, subscribed_to_user_subscriptions_count: 0, twitter_username: nil, unspent_credits_count: 0, updated_at: nil, username: "User0001", welcome_notifications: true, workshop_expiration: nil>

user.valid?
=> true

user.profile_image
=> #<ProfileImageUploader:0x00007f3524dd1f98
 @cache_id="1620745833-202884052316791-0002-6862",
 @cache_storage=
  #<CarrierWave::Storage::File:0x00007f35237a63b8
   @cache_called=nil,
   @uploader=#<ProfileImageUploader:0x00007f3524dd1f98 ...>>,
 @file=
  #<CarrierWave::SanitizedFile:0x00007f3524dd0af8
   @content=nil,
   @content_type="image/jpeg",
   @file="/opt/apps/forem/public/uploads/tmp/1620745833-202884052316791-0002-6862/image1.jpeg",
   @original_filename="image1.jpeg">,
 @filename="image1.jpeg",
 @identifier=nil,
 @model=
  #<User id: nil, apple_username: nil, articles_count: 0, badge_achievements_count: 0, blocked_by_count: 0, blocking_others_count: 0, checked_code_of_conduct: false, checked_terms_and_conditions: false, comments_count: 0, config_font: "default", config_navbar: "default", config_theme: "default", created_at: nil, credits_count: 0, display_announcements: true, display_sponsors: true, editor_version: "v1", email: "joel_deckow@nicolas.io", email_badge_notifications: true, email_comment_notifications: true, email_community_mod_newsletter: false, email_connect_messages: true, email_digest_periodic: false, email_follower_notifications: true, email_membership_newsletter: false, email_mention_notifications: true, email_newsletter: false, email_tag_mod_newsletter: false, email_unread_notifications: true, experience_level: nil, export_requested: false, exported_at: nil, facebook_username: nil, feed_fetched_at: "2017-01-01 00:00:00.000000000 -0500", feed_mark_canonical: false, feed_referential_link: true, feed_url: nil, following_orgs_count: 0, following_tags_count: 0, following_users_count: 0, github_repos_updated_at: "2017-01-01 00:00:00.000000000 -0500", github_username: nil, inbox_guidelines: nil, inbox_type: "private", last_article_at: "2017-01-01 00:00:00.000000000 -0500", last_comment_at: "2017-01-01 00:00:00.000000000 -0500", last_followed_at: nil, last_moderation_notification: "2017-01-01 00:00:00.000000000 -0500", last_notification_activity: nil, last_onboarding_page: nil, last_reacted_at: nil, latest_article_updated_at: nil, mobile_comment_notifications: true, mod_roundrobin_notifications: true, monthly_dues: 0, name: "Pres. Janette D'Amore", old_old_username: nil, old_username: nil, onboarding_package_requested: false, organization_info_updated_at: nil, payment_pointer: nil, permit_adjacent_sponsors: true, profile_image: nil, profile_updated_at: "2017-01-01 00:00:00.000000000 -0500", rating_votes_count: 0, reaction_notifications: true, reactions_count: 0, registered: true, registered_at: nil, reputation_modifier: 1.0, saw_onboarding: true, score: 0, secret: nil, signup_cta_variant: nil, spent_credits_count: 0, stripe_id_code: nil, subscribed_to_user_subscriptions_count: 0, twitter_username: nil, unspent_credits_count: 0, updated_at: nil, username: "user0001", welcome_notifications: true, workshop_expiration: nil>,
 @mounted_as=:profile_image,
 @original_filename="image1.jpeg",
 @staged=true,
 @versions={},
 @versions_to_cache=nil,
 @versions_to_store=nil>
```

This also worked fine on Debian \(when I built the app container from rails instead of the forem image\) - so I suspect there's something either with /tmp or /opt/apps/forem/ tied to selinux or some other fedora core specific item.



Let's try to see what the profile image uploader is doing:

```ruby

class Interceptor
  def initialize(delegate)
    @delegate = delegate
  end

  def method_missing(method_name, *args, &block)
    puts [method_name, args].inspect

    @delegate.send(method_name, args, &block)
  end
end

profile_image = Interceptor.new(File.open(image_path))

user = User.new(
		name: Faker::Name.name,
		      email: Faker::Internet.email,
		      username: "User0001",
		      profile_image: profile_image,
		      confirmed_at: Time.now,
		      saw_onboarding: true,
		      checked_code_of_conduct: true,
		      registered_at: Time.now)

```

This doesn't hit the method missing block to puts anything - and trying to use TracePoint didn't log anything until I exited the breakpoint and rspec continued.



I put a breakpoint in check\_size \(in CarrierWave::Uploader::FileSize - since that's one of the validations that's failing\) and wanted to look at `new_file`and see what was happening.

In the never ending shuffle of content from /tmp to /tmp to public/tmp/ to public/uploads/user/profile\_image it looks like there's also a step through tmp/

```ruby
[1] pry(#<RSpec::ExampleGroups::User::Profiles>)> 

From: /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/carrierwave-2.2.1/lib/carrierwave/uploader/file_size.rb:31 CarrierWave::Uploader::FileSize#check_size!:

    29: def check_size!(new_file)
    30:   binding.pry
 => 31:   size = new_file.size
    32:   expected_size_range = size_range
    33:   if expected_size_range.is_a?(::Range)
    34:     if size < expected_size_range.min
    35:       raise CarrierWave::IntegrityError, I18n.translate(:"errors.messages.min_size_error", :min_size => ActiveSupport::NumberHelper.number_to_human_size(expected_size_range.min))                 
    36:     elsif size > expected_size_range.max
    37:       raise CarrierWave::IntegrityError, I18n.translate(:"errors.messages.max_size_error", :max_size => ActiveSupport::NumberHelper.number_to_human_size(expected_size_range.max))                 
    38:     end
    39:   end
    40: end

[1] pry(#<ProfileImageUploader>)> new_file
=> #<CarrierWave::SanitizedFile:0x00007f619c4cf3b0
 @content=nil,
 @content_type="image/jpeg",
 @file="/opt/apps/forem/tmp/1620758519-863715365372413-0001-2311/image1.jpeg",
 @original_filename=nil>
 [2] pry(#<ProfileImageUploader>)> new_file.size
=> 0                                     

# self here - since this is a module - is ProfileImageUploader where original_filename is image1.jpeg and cache_id is the path to the file
```

Somehow _this_ file is being created, assigned to the upload, but it is empty. Having no content in the created file sure sounds like it could _also_ cause issues with validating the content type is allowed.



Copying the content of the jpeg from /tmp into the jpeg in tmp/ does cause the test to pass:

```ruby
File.open(new_file.file, 'wb') do |file|
  file.write(File.read("/tmp/image120210511-17-z5wwpm.jpeg"))  
end                                                            
=> 14946                               

new_file.size
=> 14946                                       
.

Finished in 3 minutes 16.1 seconds (files took 2.99 seconds to load)
1 example, 0 failures
```

Since there's something going wrong _before_ this is happening, let's capture the context and figure out where to look upstream

```ruby
# raise an error just to get the backtrace - 
# is there a smarter way to get this from the current execution context?

begin
  raise StandardError
rescue StandardError => e  
  puts e.backtrace         
  e             
end  
(pry):3:in `check_size!'               
# ... a bunch of lines saying I was in a pry repl when I wrote that code
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/carrierwave-2.2.1/lib/carrierwave/uploader/file_size.rb:31:in `check_size!'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/carrierwave-2.2.1/lib/carrierwave/uploader/callbacks.rb:14:in `block in with_callbacks'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/carrierwave-2.2.1/lib/carrierwave/uploader/callbacks.rb:14:in `each'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/carrierwave-2.2.1/lib/carrierwave/uploader/callbacks.rb:14:in `with_callbacks'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/carrierwave-2.2.1/lib/carrierwave/uploader/cache.rb:144:in `cache!'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/carrierwave-2.2.1/lib/carrierwave/mounter.rb:63:in `block (2 levels) in cache'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/carrierwave-2.2.1/lib/carrierwave/mounter.rb:176:in `handle_error'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/carrierwave-2.2.1/lib/carrierwave/mounter.rb:47:in `block in cache'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/carrierwave-2.2.1/lib/carrierwave/mounter.rb:46:in `map'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/carrierwave-2.2.1/lib/carrierwave/mounter.rb:46:in `cache'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/carrierwave-2.2.1/lib/carrierwave/mount.rb:146:in `profile_image='
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/carrierwave-2.2.1/lib/carrierwave/mount.rb:373:in `profile_image='
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/carrierwave-2.2.1/lib/carrierwave/orm/activerecord.rb:75:in `profile_image='
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/factory_bot-6.1.0/lib/factory_bot/attribute_assigner.rb:16:in `public_send'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/factory_bot-6.1.0/lib/factory_bot/attribute_assigner.rb:16:in `block (2 levels) in object'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/factory_bot-6.1.0/lib/factory_bot/attribute_assigner.rb:15:in `each'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/factory_bot-6.1.0/lib/factory_bot/attribute_assigner.rb:15:in `block in object'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/factory_bot-6.1.0/lib/factory_bot/attribute_assigner.rb:14:in `tap'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/factory_bot-6.1.0/lib/factory_bot/attribute_assigner.rb:14:in `object'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/factory_bot-6.1.0/lib/factory_bot/evaluation.rb:13:in `object'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/factory_bot-6.1.0/lib/factory_bot/strategy/create.rb:9:in `result'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/factory_bot-6.1.0/lib/factory_bot/factory.rb:43:in `run'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/factory_bot-6.1.0/lib/factory_bot/factory_runner.rb:29:in `block in run'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.1.3.2/lib/active_support/notifications.rb:205:in `instrument'
/opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/factory_bot-6.1.0/lib/factory_bot/factory_runner.rb:28:in `run'
```

That looks meaningful - I called profile\_image= from a factory, and carrierwave started doing things via a mounter.

{% hint style="info" %}
Full disclosure - one deadend I did attempt was to update the selection of the storage profile in the carrierwave initializer - fearing that the AWS\_ID being "Optional" would break the test for `present?` and choose the wrong path.
{% endhint %}

So what methods look interesting in this call stack?

We have AR's profile image set calling into the mount's profile image set, calling cache in mount.rb:46 \(def cache is line 43 in that file\): 

```ruby
    def cache(new_files)
      return if !new_files.is_a?(Array) && new_files.blank?
      old_uploaders = uploaders
      @uploaders = new_files.map do |new_file|
        handle_error do
          if new_file.is_a?(String)
            if (uploader = old_uploaders.detect { |uploader| uploader.identifier == new_file })
              uploader.staged = true
              uploader
            else
              begin
                uploader = blank_uploader
                uploader.retrieve_from_cache!(new_file)
                uploader
              rescue CarrierWave::InvalidParameter
                nil
              end
            end
          else
            uploader = blank_uploader
            uploader.cache!(new_file)
            uploader
          end
        end
      end.compact
    end

```

line 63 is `uploader.cache!(new_file)` which is where we leave this file \(having allocated a blank uploader?\) - suggesting "no, upload was not a string, it was a File/IO/Stream type object. This might actually be either too late \(we already have new\_file which might be the wrong object\) or too early \(cache! might be the method creating the file in tmp/\) - this is worth putting a breakpoint at line 62 \(uploader has been created, but not called, and new file is still in scope\). For the meantime, I'll leave the breakpoint after in `check_size` in place as well.

![](../.gitbook/assets/screenshot-from-2021-05-11-14-58-29.png)

Okay, great - I'm on top of the problem \(I have the input and the object I'm giving the input to, and am able to repeatably create the error condition\):

```ruby
[8] pry(#<CarrierWave::Mounter>)> new_file
=> #<Rack::Test::UploadedFile:0x00007fd4c8edf428
 @content_type="image/jpeg",
 @original_filename="image1.jpeg",
 @tempfile=#<File:/tmp/image120210512-17-v1jenc.jpeg>>
[9] pry(#<CarrierWave::Mounter>)> new_file.size
=> 14946                

[11] pry(#<CarrierWave::Mounter>)> uploader = blank_uploader
=> #<ProfileImageUploader:0x00007fd4c97904c8                
 @cache_id=nil,
 @file=nil,
 @filename=nil,
 @identifier=nil,
 @model=
  #<User id: nil, apple_username: nil, articles_count: 0, badge_achievements_count: 0, blocked_by_count: 0, blocking_others_count: 0, checked_code_of_conduct: false, checked_terms_and_conditions: false, comments_count: 0, config_font: "default", config_navbar: "default", config_theme: "default", created_at: nil, credits_count: 0, display_announcements: true, display_sponsors: true, editor_version: "v1", email: "person1@example.com", email_badge_notifications: true, email_comment_notifications: true, email_community_mod_newsletter: false, email_connect_messages: true, email_digest_periodic: false, email_follower_notifications: true, email_membership_newsletter: false, email_mention_notifications: true, email_newsletter: false, email_tag_mod_newsletter: false, email_unread_notifications: true, experience_level: nil, export_requested: false, exported_at: nil, facebook_username: nil, feed_fetched_at: "2017-01-01 16:00:00.000000000 +1100", feed_mark_canonical: false, feed_referential_link: true, feed_url: nil, following_orgs_count: 0, following_tags_count: 0, following_users_count: 0, github_repos_updated_at: "2017-01-01 16:00:00.000000000 +1100", github_username: nil, inbox_guidelines: nil, inbox_type: "private", last_article_at: "2017-01-01 16:00:00.000000000 +1100", last_comment_at: "2017-01-01 16:00:00.000000000 +1100", last_followed_at: nil, last_moderation_notification: "2017-01-01 16:00:00.000000000 +1100", last_notification_activity: nil, last_onboarding_page: nil, last_reacted_at: nil, latest_article_updated_at: nil, mobile_comment_notifications: true, mod_roundrobin_notifications: true, monthly_dues: 0, name: "Elza Huels", old_old_username: nil, old_username: nil, onboarding_package_requested: false, organization_info_updated_at: nil, payment_pointer: nil, permit_adjacent_sponsors: true, profile_image: nil, profile_updated_at: "2017-01-01 16:00:00.000000000 +1100", rating_votes_count: 0, reaction_notifications: true, reactions_count: 0, registered: true, registered_at: nil, reputation_modifier: 1.0, saw_onboarding: true, score: 0, secret: nil, signup_cta_variant: nil, spent_credits_count: 0, stripe_id_code: nil, subscribed_to_user_subscriptions_count: 0, twitter_username: nil, unspent_credits_count: 0, updated_at: nil, username: "username1", welcome_notifications: true, workshop_expiration: nil>,
 @mounted_as=:profile_image,
 @staged=false,
 @versions=nil,
 @versions_to_cache=nil,
 @versions_to_store=nil>
[12] pry(#<CarrierWave::Mounter>)> uploader.cache! new_file                                             
CarrierWave::IntegrityError: File size should be greater than 1 Byte
from /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/carrierwave-2.2.1/lib/carrierwave/uploader/file_size.rb:35:in `check_size!'
[13] pry(#<CarrierWave::Mounter>)> uploader
=> #<ProfileImageUploader:0x00007fd4c97904c8
 @cache_id="1620764971-196491613461237-0002-9396",
 @file=
  #<CarrierWave::SanitizedFile:0x00007fd4c9809698
   @content=nil,
   @content_type="image/jpeg",
   @file="/opt/apps/forem/tmp/1620764971-196491613461237-0002-9396/image1.jpeg",
   @original_filename=nil>,
 @filename="image1.jpeg",
 @identifier=nil,
 @model=
  #<User id: nil, apple_username: nil, articles_count: 0, badge_achievements_count: 0, blocked_by_count: 0, blocking_others_count: 0, checked_code_of_conduct: false, checked_terms_and_conditions: false, comments_count: 0, config_font: "default", config_navbar: "default", config_theme: "default", created_at: nil, credits_count: 0, display_announcements: true, display_sponsors: true, editor_version: "v1", email: "person1@example.com", email_badge_notifications: true, email_comment_notifications: true, email_community_mod_newsletter: false, email_connect_messages: true, email_digest_periodic: false, email_follower_notifications: true, email_membership_newsletter: false, email_mention_notifications: true, email_newsletter: false, email_tag_mod_newsletter: false, email_unread_notifications: true, experience_level: nil, export_requested: false, exported_at: nil, facebook_username: nil, feed_fetched_at: "2017-01-01 16:00:00.000000000 +1100", feed_mark_canonical: false, feed_referential_link: true, feed_url: nil, following_orgs_count: 0, following_tags_count: 0, following_users_count: 0, github_repos_updated_at: "2017-01-01 16:00:00.000000000 +1100", github_username: nil, inbox_guidelines: nil, inbox_type: "private", last_article_at: "2017-01-01 16:00:00.000000000 +1100", last_comment_at: "2017-01-01 16:00:00.000000000 +1100", last_followed_at: nil, last_moderation_notification: "2017-01-01 16:00:00.000000000 +1100", last_notification_activity: nil, last_onboarding_page: nil, last_reacted_at: nil, latest_article_updated_at: nil, mobile_comment_notifications: true, mod_roundrobin_notifications: true, monthly_dues: 0, name: "Elza Huels", old_old_username: nil, old_username: nil, onboarding_package_requested: false, organization_info_updated_at: nil, payment_pointer: nil, permit_adjacent_sponsors: true, profile_image: nil, profile_updated_at: "2017-01-01 16:00:00.000000000 +1100", rating_votes_count: 0, reaction_notifications: true, reactions_count: 0, registered: true, registered_at: nil, reputation_modifier: 1.0, saw_onboarding: true, score: 0, secret: nil, signup_cta_variant: nil, spent_credits_count: 0, stripe_id_code: nil, subscribed_to_user_subscriptions_count: 0, twitter_username: nil, unspent_credits_count: 0, updated_at: nil, username: "username1", welcome_notifications: true, workshop_expiration: nil>,
 @mounted_as=:profile_image,
 @original_filename="image1.jpeg",
 @staged=true,
 @versions=nil,
 @versions_to_cache=nil,
 @versions_to_store=nil>
[15] pry(#<CarrierWave::Mounter>)> uploader.file.size
=> 0                               
```

So let's take a look inside `cache!` which raises the error:

```ruby
From: /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/carrierwave-2.2.1/lib/carrierwave/uploader/cache.rb:109: 
Owner: CarrierWave::Uploader::Cache
Visibility: public
Signature: cache!(new_file=?)
Number of lines: 27

      ##
      # Caches the given file. Calls process! to trigger any process callbacks.
      #
      # By default, cache!() uses copy_to(), which operates by copying the file
      # to the cache, then deleting the original file.  If move_to_cache() is
      # overriden to return true, then cache!() uses move_to(), which simply
      # moves the file to the cache.  Useful for large files.
      #
      # === Parameters
      #
      # [new_file (File, IOString, Tempfile)] any kind of file object
      #
      # === Raises
      #
      # [CarrierWave::FormNotMultipart] if the assigned parameter is a string
      #

def cache!(new_file = file)
  new_file = CarrierWave::SanitizedFile.new(new_file)
  return if new_file.empty?

  raise CarrierWave::FormNotMultipart if new_file.is_path? && ensure_multipart_form

  self.cache_id = CarrierWave.generate_cache_id unless cache_id

  @staged = true
  @filename = new_file.filename
  self.original_filename = new_file.filename

  begin
    # first, create a workfile on which we perform processings
    if move_to_cache
      @file = new_file.move_to(File.expand_path(workfile_path, root), permissions, directory_permissions)                                  
    else
      @file = new_file.copy_to(File.expand_path(workfile_path, root), permissions, directory_permissions)                                  
    end

    with_callbacks(:cache, @file) do
      @file = cache_storage.cache!(@file)
    end
  ensure
    FileUtils.rm_rf(workfile_path(''))
  end
end
```

I am going to pretend it's okay to just reuse the current context \(inside the mounter's cache method\) and not set another breakpoint for the moment.

```ruby
original_file = new_file
=> #<Rack::Test::UploadedFile:0x00007fd4c8edf428               
 @content_type="image/jpeg",
 @original_filename="image1.jpeg",
 @tempfile=#<File:/tmp/image120210512-17-v1jenc.jpeg>>
new_file = CarrierWave::SanitizedFile.new(original_file)
=> #<CarrierWave::SanitizedFile:0x00007fd4c9b2ebe8        
 @content=nil,
 @content_type=nil,
 @file=
  #<Rack::Test::UploadedFile:0x00007fd4c8edf428
   @content_type="image/jpeg",
   @original_filename="image1.jpeg",
   @tempfile=#<File:/tmp/image120210512-17-v1jenc.jpeg>>,
 @original_filename=nil>
new_file.empty?
=> false
new_file.size
=> 14946

new_file.is_path?
=> false
# ensure there's no override (see comment)
uploader.move_to_cache             
=> false

new_file
```

I think the place this _might_ be going wrong is in copy\_to:

```text
          else
            @file = new_file.copy_to(File.expand_path(workfile_path, root), permissions, directory_permissions)
```

Since I have this 

```ruby
uploader.send(:workfile_path)                                        
=> "/opt/apps/forem/tmp/1620764971-196491613461237-0002-9396/image1.jpeg"
uploader.file
=> #<CarrierWave::SanitizedFile:0x00007fd4c9809698
 @content=nil,
 @content_type="image/jpeg",
 @file="/opt/apps/forem/tmp/1620764971-196491613461237-0002-9396/image1.jpeg",
 @original_filename=nil>
 
File.expand_path(uploader.send(:workfile_path), uploader.send(:root))
=> "/opt/apps/forem/tmp/1620764971-196491613461237-0002-9396/image1.jpeg"                              
uploader.send(:permissions)
=> 420 # 0644 octal                                                        
uploader.send(:directory_permissions)                                
=> 493 # 0755 octal

# indeed - this is where it goes wrong
[41] pry(#<CarrierWave::Mounter>)> copied=  new_file.copy_to("/opt/apps/forem/tmp/image2.jpeg", 420, 493)                                                                                                      
=> #<CarrierWave::SanitizedFile:0x00007fd4ca04cda0
 @content=nil,
 @content_type="image/jpeg",
 @file="/opt/apps/forem/tmp/image2.jpeg",
 @original_filename=nil>
[42] pry(#<CarrierWave::Mounter>)> copied.size
=> 0                                          
[43] pry(#<CarrierWave::Mounter>)> copied.read
=> ""                                         
[44] pry(#<CarrierWave::Mounter>)> new_file.read.size
=> 14946                                      
```





So this is aggravating - FileUtils.cp is getting a valid source, a valid destination, it's creating a new entry \(file\) but nothing is getting put into it

```ruby
FileUtils.cp(path, "/tmp/whynot.jpg", verbose: true)
cp /tmp/image120210512-17-v1jenc.jpeg /tmp/whynot.jpg                                           
# it's a lie, verbose just says what it "intends" to do - not what actually happens
puts `ls -l /tmp/whynot.jpg`
-rw------- 1 forem forem 0 May 12 08:30 /tmp/whynot.jpg                 

puts `ls -l #{path}`
-rw------- 1 forem forem 14946 May 12 07:08 /tmp/image120210512-17-v1jenc.jpeg

# shelling out to system cp of course works correctly
`cp /tmp/image120210512-17-v1jenc.jpeg /tmp/whatever.jpg`

puts `ls -l /tmp -h`
total 52K                                                       
-rw-r--r-- 1 root  root   543 May 11 22:57 core-js-banners
-rw-r--r-- 1 forem forem    0 May 12 08:28 file.file.jpg
-rw------- 1 forem forem  15K May 12 07:08 image120210512-17-v1jenc.jpeg
-rwx------ 1 root  root   757 Oct 27  2020 ks-script-en4f4be7
drwxr-xr-x 3 root  root  4.0K May 11 22:56 v8-compile-cache-0
-rw------- 1 forem forem  15K May 12 08:32 whatever.jpg
-rw------- 1 forem forem    0 May 12 08:30 whynot.jpg
drwxr-xr-x 2 root  root  4.0K May 11 22:57 yarn--1620734242533-0.1696038809539928
drwxr-xr-x 2 root  root  4.0K May 11 22:58 yarn--1620734303883-0.5899124078896565
 

# I'm not supposed to end up in the stdlib looking at file utilities to call cp:

From: /usr/share/ruby/fileutils.rb:416:
Owner: #<Class:FileUtils>
Visibility: public
Signature: cp(src, dest, preserve:?, noop:?, verbose:?)
Number of lines: 7

def cp(src, dest, preserve: nil, noop: nil, verbose: nil)
  fu_output_message "cp#{preserve ? ' -p' : ''} #{[src,dest].flatten.join ' '}" if verbose
  return if noop
  fu_each_src_dest(src, dest) do |s, d|
    copy_file s, d, preserve
  end
end

                                                                         
From: /usr/share/ruby/fileutils.rb:506:
Owner: #<Class:FileUtils>
Visibility: public
Signature: copy_file(src, dest, preserve=?, dereference=?)
Number of lines: 5

def copy_file(src, dest, preserve = false, dereference = true)
  ent = Entry_.new(src, nil, dereference)
  ent.copy_file dest
  ent.copy_metadata dest if preserve
end

```

So why is the stdlib's `FileUtils.cp` creating an empty file successfully but not copying the data? I do see `Entry_` there and it has a `copy_file` method \(which I assume does something like moving the file content?\). 



I think my next step is to put a breakpoint in FileUtils.copy\_file \(I'll probably open the class and redefine the method to add the breakpoint as part of a unit test setup\). That's for tomorrow.



### tomorrow

Is it tomorrow already? Well time to get to work.

```ruby
# added to the start of spec/models/user_spec.rb

module FileUtils
  def self.copy_file(src, dest, preserve = false, dereference=true)
    ent = Entry_.new(src, nil, dereference)
    binding.pry
    ent.copy_file dest
    ent.copy_metadata dest if preserve
  end
end
```

We'll hit this twice, once for the downloaded file being copied to /tmp and a second time for the /tmp file being copied to /opt/apps/forem/tmp/ the first pass is normal

```ruby
[8] pry(FileUtils)> ent.copy_file dest
=> 14946                              

[9] pry(FileUtils)> puts `ls -l #{ent.path}`
-rw-r--r-- 1 forem forem 14946 Mar 23 17:01 /opt/apps/forem/spec/support/fixtures/images/image1.jpeg
=> nil
[10] pry(FileUtils)> puts `ls -lh #{dest}`-rw------- 1 forem forem 15K May 13 16:54 /tmp/image120210513-17-1sgrdmh.jpeg
=> nil
```

We resume and wait for the second pass \(it's basically immediate\)

```ruby
From: /opt/apps/forem/spec/models/user_spec.rb:7 FileUtils.copy_file:

    4: def self.copy_file(src, dest, preserve = false, dereference=true)
    5:   ent = Entry_.new(src, nil, dereference)
    6:   binding.pry
 => 7:   ent.copy_file dest
    8:   ent.copy_metadata dest if preserve
    9: end

[1] pry(FileUtils)> ent
=> #<FileUtils::Entry_ /tmp/image120210513-17-1sgrdmh.jpeg>
[2] pry(FileUtils)> puts `ls -lh #{dest}`
ls: cannot access '/opt/apps/forem/tmp/1620917799-429616350791124-0001-1815/image1.jpeg': No such file or directory
=> nil
[3] pry(FileUtils)> puts `ls -lh #{src}`  
-rw------- 1 forem forem 15K May 13 16:56 /tmp/image120210513-17-1sgrdmh.jpeg
=> nil
[4] pry(FileUtils)> ent.copy_file dest
=> 0
[5] pry(FileUtils)> ent
=> #<FileUtils::Entry_ /tmp/image120210513-17-1sgrdmh.jpeg>
[6] pry(FileUtils)> puts `ls -lh #{dest}`                                                               
-rw------- 1 forem forem 0 May 13 16:57 /opt/apps/forem/tmp/1620917799-429616350791124-0001-1815/image1.jpeg
=> nil
[7] pry(FileUtils)> ent.copy_file dest
=> 0                                  
[8] pry(FileUtils)> ent.copy_file dest
=> 0                
```

This is where things go wrong. We have what looks like the same setup - a src file, a dest path, which we open for writing, and a call to IO.copy\_stream

```ruby
show-method ent.copy_file
                                              
From: /usr/share/ruby/fileutils.rb:1413:
Owner: FileUtils::Entry_
Visibility: public
Signature: copy_file(dest)
Number of lines: 7

def copy_file(dest)
  File.open(path()) do |s|
    File.open(dest, 'wb', s.stat.mode) do |f|
      IO.copy_stream(s, f)
    end
  end
end
```

so `Entry_` objects expose `stat` which basically reflects the stat from the unix filesystem

```ruby
[14] pry(FileUtils)> f = File.open(ent.path)
=> #<File:/tmp/image120210513-17-1sgrdmh.jpeg>

[15] pry(FileUtils)> f.stat.mode
=> 33152

[16] pry(FileUtils)> fd = File.open(dest, 'wb', f.stat.mode)
=> #<File:/opt/apps/forem/tmp/1620917799-429616350791124-0001-1815/image1.jpeg>

[17] pry(FileUtils)> IO.copy_stream(f, fd)
=> 0

 f.stat
=> #<File::Stat
 dev=0x68,
 ino=17969135,
 mode=0100600 (file rw-------),
 nlink=1,
 uid=1000 (forem),
 gid=1000 (forem),
 rdev=0x0 (0, 0),
 size=14946,
 blksize=4096,
 blocks=32,
 atime=2021-05-13 16:56:37.056857169 +0200 (1620917797),
 mtime=2021-05-13 16:56:37.056857169 +0200 (1620917797),
 ctime=2021-05-13 16:56:37.056857169 +0200 (1620917797)>
 
 fd.stat                       
=> #<File::Stat                                    
 dev=0xfe00,
 ino=4468933,
 mode=0100600 (file rw-------),
 nlink=1,
 uid=1000 (forem),
 gid=1000 (forem),
 rdev=0x0 (0, 0),
 size=0,
 blksize=4096,
 blocks=0,
 atime=2021-05-13 16:57:07.369100161 +0200 (1620917827),
 mtime=2021-05-13 17:00:36.430670776 +0200 (1620918036),
 ctime=2021-05-13 17:00:36.430670776 +0200 (1620918036)>
```

Both entries are owned by forem \(current user\) and both are 0x600 \(read-write user only\).

`stat.mode` appears to be the input to `creat` - see `man creat(2)` or `man open(2)`

The primary distinction between this call and the preceding one is that /opt/apps/forem/tmp is on the mounted volume, while /tmp is inside the container. The dest file is _created_ by File.open but copy\_stream does not move any bytes to the target.

Let's open a new file in /tmp/whatever.jpg and observe if the issue is cross-filesystem cross-device copying \(there was a note about that in [https://bugs.ruby-lang.org/issues/13867](https://bugs.ruby-lang.org/issues/13867) that might come into play in a minute\).

```ruby
[26] pry(FileUtils)> ftmp = File.open("/tmp/whatever.jpg", "wb", f.stat.mode)
=> #<File:/tmp/whatever.jpg>                                                 
[27] pry(FileUtils)> IO.copy_stream(f, ftmp)
=> 0
```

"Huh." it looks like maybe the mode is the issue? In any case, it doesn't look like the issue is crossing into a mounted volume if writing to tmp also fails the same way \(weird\).

[https://ruby-doc.org/core-2.7.2/IO.html\#method-c-copy\_stream](https://ruby-doc.org/core-2.7.2/IO.html#method-c-copy_stream) is where we are - it's a c function - good luck in there!

```ruby
[75] pry(FileUtils)> orig="/opt/apps/forem/spec/support/fixtures/images/image1.jpeg"                   
=> "/opt/apps/forem/spec/support/fixtures/images/image1.jpeg"                       
[76] pry(FileUtils)> forig = File.open(orig)                                                            
=> #<File:/opt/apps/forem/spec/support/fixtures/images/image1.jpeg>

forig.stat
=> #<File::Stat                
 dev=0xfe00,
 ino=4064314,
 mode=0100644 (file rw-r--r--),
 nlink=1,
 uid=1000 (forem),
 gid=1000 (forem),
 rdev=0x0 (0, 0),
 size=14946,
 blksize=4096,
 blocks=32,
 atime=2021-05-13 16:54:17.383673726 +0200 (1620917657),
 mtime=2021-03-23 17:01:14.933220555 +0100 (1616515274),
 ctime=2021-04-12 21:52:50.837852041 +0200 (1618257170)>
[79] pry(FileUtils)> ftarget = "/tmp/whynot.jpeg"                                                       
=> "/tmp/whynot.jpeg"     

# this is a mistake - 81 passes a path and not a file to the copy_stream call:                       
[80] pry(FileUtils)> File.open(ftarget, 'wb', forig.stat.mode)
=> #<File:/tmp/whynot.jpeg>                                   
[81] pry(FileUtils)> IO.copy_stream(forig, ftarget)
=> 14946

# so we open a file like we should have
[84] pry(FileUtils)> ftargetf = File.open(ftarget, 'wb', forig.stat.mode)
=> #<File:/tmp/whynot.jpeg>            
# and copy stream                                  
[85] pry(FileUtils)> IO.copy_stream(forig, ftargetf)
=> 0                                                

# give up and reopen
[88] pry(FileUtils)> forig.close       
=> nil                
[91] pry(FileUtils)> forig = File.open(orig)                                            
=> #<File:/opt/apps/forem/spec/support/fixtures/images/image1.jpeg>
[92] pry(FileUtils)> IO.copy_stream(forig, ftargetf)
=> 14946                                            

# we have seek'd to the end:
[93] pry(FileUtils)> forig.pos
=> 14946                      
# read does nothing at EOS so no bytes copied
[94] pry(FileUtils)> IO.copy_stream(forig, ftargetf)
=> 0                       
# lets set pos back to the beginning        
[95] pry(FileUtils)> forig.rewind
=> 0                             
[96] pry(FileUtils)> forig.pos
=> 0                          
[97] pry(FileUtils)> IO.copy_stream(forig, ftargetf)
=> 14946                        


```

I still don't understand how we get into this situation - we have an entry and a src, are we passing an open \(and read\) file during the call, but only when we copy into the mounted volume? that seems like it's a different way of getting 0 bytes copied

```ruby
[106] pry(FileUtils)> fd = File.open(fd, 'wb', f.stat.mode)                            
=> #<File:/opt/apps/forem/tmp/1620917799-429616350791124-0001-1815/image1.jpeg>        
[107] pry(FileUtils)> fd.stat
=> #<File::Stat
 dev=0xfe00,
 ino=4468933,
 mode=0100600 (file rw-------),
 nlink=1,
 uid=1000 (forem),
 gid=1000 (forem),
 rdev=0x0 (0, 0),
 size=0,
 blksize=4096,
 blocks=0,
 atime=2021-05-13 16:57:07.369100161 +0200 (1620917827),
 mtime=2021-05-13 18:15:48.3869598 +0200 (1620922548),
 ctime=2021-05-13 18:15:48.3869598 +0200 (1620922548)>
[108] pry(FileUtils)> fd.size                             
=> 0                                                      
[109] pry(FileUtils)> f.size
=> 14946                    
[110] pry(FileUtils)> f.pos
=> 0                       
[111] pry(FileUtils)> IO.copy_stream(f, fd)
=> 0                                       
[112] pry(FileUtils)> f.pos
=> 0                       
[113] pry(FileUtils)> f.read ; nil
=> nil                            
[114] pry(FileUtils)> f.pos
=> 14946                   
[115] pry(FileUtils)> f.rewind
=> 0                          
[116] pry(FileUtils)> IO.copy_stream(f, fd)
=> 0                                       
```

rewinding and retrying does _not_ fix it here \(the way we actually failed during execution\).

Adding insult to injury - if I open the original file and copy from it \(rather than the temp file\) it just works

```ruby
[121] pry(FileUtils)> IO.copy_stream(f, fd.path)  
=> 0                                              
[122] pry(FileUtils)> forig
=> #<File:/opt/apps/forem/spec/support/fixtures/images/image1.jpeg>
[123] pry(FileUtils)> forig.pos                           
=> 14946                                                  
[124] pry(FileUtils)> forig.rewind
=> 0                              
[125] pry(FileUtils)> IO.copy_stream(forig, fd)
=> 14946               
```

### Summary of where I think I am

* this worked fine in a debian container \(where the source was copied rather than mounted\)
* copying file to tmp works
* copy file from tmp fails
* position does not seek \(we basically didn't read at all from the open file\)
* copying file to ultimate target works.
* copying to another file in tmp works, but copying from _that_ file also fails to copy into the dest

Only "visible" difference is the original file was 0100644 and the tmp file is 0100600 - but we're the correct user and should be able to read \(in fact, we can read, just not copy stream\).



## fix?

I ended up changing the docker-compose file I'm working with to mount /tmp as tmpfs rather than using the /tmp from the overlay filesystem. This appears to have fixed the issues for me \(I now see all but the system tests requiring chrome building with no failures\).

So that's good enough for a 0 level approach - next is to figure out selenium + chromedriver + webdrivers gem + rspec with docker \(molly had a branch with this early on that I might revisit - I remember a lot of quirks around allowed sites and webmock/vcr interacting differently at the end so sessions weren't getting removed\). I suspect we can run selenium/system specs in a separate container \(using `--tag=js` to filter them\).

Second thing I am not doing is using knapsack \(but the whole spec suite takes 12 minutes in docker locally so not sure how much that matters\).

