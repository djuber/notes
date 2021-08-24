---
description: >-
  In which I try to clone forem's repo, run docker compose, and use it for
  development for a while
---

# Docker development revisited

I started here in March, and it came up in an internal meeting as a pain point yesterday. 

Today I'm going to try docker \(docker-compose\) based development and get a feeling for the existing pain points. Forem in production \(outside of a few sites on heroku\) is entirely driven by containers, so running in docker shouldn't be wildly different from production, but users consistently have a hard time.



### Checkout the code

```bash
git clone https://github.com/forem/forem
```

This part at least went well.

```bash
cd forem
docker-compose up

Creating network "forem_default" with the default driver
Creating volume "forem_db_data" with default driver
Pulling bundle (quay.io/forem/forem:development)...
development: Pulling from forem/forem
Creating forem_yarn       ... done
Creating forem_postgresql ... done
Creating forem_bundle     ... done
Creating forem_redis      ... done
Creating forem_rails      ... done
Creating forem_webpacker  ... done
Creating forem_sidekiq    ... done
Creating forem_seed       ... done
Attaching to forem_postgresql, forem_yarn, forem_redis, forem_bundle, forem_rails, forem_webpacker, forem_sidekiq, forem_seed

```

This worked fine \(or looked like it did\) - first bundle is run \(in the forem\_bundle container, as scripts/bundle.sh\), periodically containers will notify that they're waiting for other processes to complete \(rails is waiting for redis, postgres, and the sentinel files from the other processes, many other services are waiting on rails\), yarn can run while bundle is running \(these are interleaved in the log output\).

I ran into an issue with the rails container running migrations the first time - I'm leaving the output here messy to illustrate the interleaving - only the `rails_1` and `forem_postgresql` lines are relevant - when running the primary key migration on the notifications table there's a postgresql timeout \(and consequently and AR error raised\). 

```text
rails_1      | == 20200806193438 LargeTablePrimaryKeysToBigints: migrating ===================
rails_1      | migrating ahoy_message PKs to bigints
sidekiq_1    | 2021/08/24 15:36:00 Problem with dial: dial tcp 172.20.0.6:3000: connect: connection refused. Sleeping 20s
seed_1       | 2021/08/24 15:36:01 Problem with dial: dial tcp 172.20.0.6:3000: connect: connection refused. Sleeping 20s
rails_1      | migrating article PKs to bigints
rails_1      | migrating notification PKs to bigints
sidekiq_1    | 2021/08/24 15:36:20 Problem with dial: dial tcp 172.20.0.6:3000: connect: connection refused. Sleeping 20s
seed_1       | 2021/08/24 15:36:21 Problem with dial: dial tcp 172.20.0.6:3000: connect: connection refused. Sleeping 20s
forem_postgresql | 2021-08-24 15:36:23.628 UTC [62] ERROR:  canceling statement due to statement timeout
forem_postgresql | 2021-08-24 15:36:23.628 UTC [62] STATEMENT:          ALTER TABLE notifications
forem_postgresql | 	          ALTER COLUMN id TYPE bigint,
forem_postgresql | 	          ALTER COLUMN notifiable_id TYPE bigint,
forem_postgresql | 	          ALTER COLUMN user_id TYPE bigint
forem_postgresql | 	
rails_1      | rails aborted!
rails_1      | StandardError: An error has occurred, this and all later migrations canceled:
rails_1      | 
rails_1      | PG::QueryCanceled: ERROR:  canceling statement due to statement timeout

```

This is fixable \(because I know how the rails migrations work this is just a case of stopping the docker processes, and restarting again - the db is persisted to storage so the schema migrations which succeeded will be skipped - and the bundle/yarn/webpack steps should be a little faster since everything is in place already. This would be a seriouly aggravating case for someone else \(the only logged events after the rails container dies are two other containers seed and sidekiq saying they cannot connect to port 3000, and they sleep waiting longer\).



```text
^CGracefully stopping... (press Ctrl+C again to force)
Stopping forem_seed       ... done
Stopping forem_sidekiq    ... done
Stopping forem_webpacker  ... done
Stopping forem_redis      ... done
Stopping forem_postgresql ... done
djuber@busybee:~/src/forem$ docker-compose down
Removing forem_seed       ... done
Removing forem_sidekiq    ... done
Removing forem_webpacker  ... done
Removing forem_rails      ... done
Removing forem_redis      ... done
Removing forem_postgresql ... done
Removing forem_yarn       ... done
Removing forem_bundle     ... done
Removing network forem_default
djuber@busybee:~/src/forem$ docker-compose up
Creating network "forem_default" with the default driver
Creating forem_postgresql ... done
Creating forem_yarn       ... done
Creating forem_bundle     ... done
Creating forem_redis      ... done

```

On the second pass the migrations completed - rails listener started successfully - and sidekiq started up - the seed step however raised an error related to an unset APP\_PROTOCOL in the environment

```text
seed_1       | Seeding with multiplication factor: 1
seed_1       | 
seed_1       |   1. Creating Organizations.
seed_1       |   2. Creating DEV profile fields.
seed_1       |   3. Creating 10 Users.
seed_1       |   User with email = admin@forem.local not found, proceeding...
seed_1       | rake aborted!
seed_1       | NoMethodError: undefined method `protocol' for nil:NilClass
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/actionview-6.1.4.1/lib/action_view/helpers/asset_url_helper.rb:303:in `compute_asset_host'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/actionview-6.1.4.1/lib/action_view/helpers/asset_url_helper.rb:212:in `asset_path'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/cloudinary-1.21.0/lib/cloudinary/helper.rb:378:in `path_to_asset'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/actionview-6.1.4.1/lib/action_view/helpers/asset_url_helper.rb:231:in `asset_url'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/actionview-6.1.4.1/lib/action_view/helpers/asset_url_helper.rb:390:in `image_url'
seed_1       | /opt/apps/forem/app/lib/url.rb:67:in `local_image'
seed_1       | /opt/apps/forem/app/models/settings/general.rb:56:in `block in <class:General>'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/rails-settings-cached-2.6.0/lib/rails-settings/base.rb:95:in `block in _define_field'
seed_1       | /opt/apps/forem/app/services/users/create_mascot_account.rb:7:in `<class:CreateMascotAccount>'
seed_1       | /opt/apps/forem/app/services/users/create_mascot_account.rb:2:in `<module:Users>'
seed_1       | /opt/apps/forem/app/services/users/create_mascot_account.rb:1:in `<main>'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/bootsnap-1.7.7/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:23:in `require'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/bootsnap-1.7.7/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:23:in `block in require_with_bootsnap_lfi'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/bootsnap-1.7.7/lib/bootsnap/load_path_cache/loaded_features_index.rb:92:in `register'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/bootsnap-1.7.7/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:22:in `require_with_bootsnap_lfi'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/bootsnap-1.7.7/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:31:in `require'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/zeitwerk-2.4.2/lib/zeitwerk/kernel.rb:26:in `require'
seed_1       | /opt/apps/forem/db/seeds.rb:158:in `<main>'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/bootsnap-1.7.7/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:60:in `load'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/bootsnap-1.7.7/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:60:in `load'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/railties-6.1.4.1/lib/rails/engine.rb:566:in `block in load_seed'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.1.4.1/lib/active_support/callbacks.rb:117:in `block in run_callbacks'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.1.4.1/lib/active_support/execution_wrapper.rb:88:in `wrap'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/railties-6.1.4.1/lib/rails/engine.rb:640:in `block (2 levels) in <class:Engine>'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.1.4.1/lib/active_support/callbacks.rb:126:in `instance_exec'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.1.4.1/lib/active_support/callbacks.rb:126:in `block in run_callbacks'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.1.4.1/lib/active_support/callbacks.rb:137:in `run_callbacks'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/railties-6.1.4.1/lib/rails/engine.rb:566:in `load_seed'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activerecord-6.1.4.1/lib/active_record/tasks/database_tasks.rb:450:in `load_seed'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activerecord-6.1.4.1/lib/active_record/railties/databases.rake:392:in `block (2 levels) in <main>'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/honeycomb-beeline-2.6.0/lib/honeycomb/integrations/rake.rb:21:in `block in execute'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/honeycomb-beeline-2.6.0/lib/honeycomb/client.rb:62:in `start_span'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/honeycomb-beeline-2.6.0/lib/honeycomb/integrations/rake.rb:16:in `execute'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/rake-13.0.6/exe/rake:27:in `<top (required)>'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/cli/exec.rb:63:in `load'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/cli/exec.rb:63:in `kernel_load'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/cli/exec.rb:28:in `run'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/cli.rb:494:in `exec'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/vendor/thor/lib/thor/command.rb:27:in `run'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/vendor/thor/lib/thor/invocation.rb:127:in `invoke_command'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/vendor/thor/lib/thor.rb:392:in `dispatch'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/cli.rb:30:in `dispatch'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/vendor/thor/lib/thor/base.rb:485:in `start'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/cli.rb:24:in `start'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/exe/bundle:49:in `block in <top (required)>'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/friendly_errors.rb:130:in `with_friendly_errors'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/exe/bundle:37:in `<top (required)>'
seed_1       | /usr/local/bin/bundle:23:in `load'
seed_1       | /usr/local/bin/bundle:23:in `<main>'
seed_1       | Tasks: TOP => db:seed
seed_1       | (See full trace by running task with --trace)
seed_1       | 2021/08/24 15:52:15 Command exited with error: exit status 1
forem_seed exited with code 1

```

The code in url.rb where this occurred was the call to `image_url` with image name and host \(should be set to the asset host if present\):



```ruby
  # Creates an Image URL - a shortcut for the .image_url helper
  #
  # @param image_name [String] the image file name
  # @param host [String] (optional) the host for the image URL you'd like to use
  def self.local_image(image_name, host: nil)
    host ||= ActionController::Base.asset_host || url(nil)
    ActionController::Base.helpers.image_url(image_name, host: host)
  end

```

Ah - `asset_host` is in the environment but I \(as I always do\) forget to copy the environment file from the sample. Do that now and restart

```text
djuber@busybee:~/src/forem$ cp .env_sample .env
djuber@busybee:~/src/forem$ docker-compose down
Removing forem_seed       ... done
Removing forem_webpacker  ... done
Removing forem_sidekiq    ... done
Removing forem_rails      ... done
Removing forem_bundle     ... done
Removing forem_yarn       ... done
Removing forem_redis      ... done
Removing forem_postgresql ... done
Removing network forem_default
djuber@busybee:~/src/forem$ docker-compose up

```

Failed again with another problem \(limit was nil\)?

```text
seed_1       |   11. Creating Badges.
seed_1       | rake aborted!
seed_1       | NoMethodError: undefined method `limit' for nil:NilClass
seed_1       | /opt/apps/forem/db/seeds.rb:410:in `block in <main>'
seed_1       | /opt/apps/forem/app/lib/seeder.rb:30:in `create_if_none'
seed_1       | /opt/apps/forem/db/seeds.rb:401:in `<main>'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/bootsnap-1.7.7/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:60:in `load'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/bootsnap-1.7.7/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:60:in `load'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/railties-6.1.4.1/lib/rails/engine.rb:566:in `block in load_seed'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.1.4.1/lib/active_support/callbacks.rb:117:in `block in run_callbacks'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.1.4.1/lib/active_support/execution_wrapper.rb:88:in `wrap'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/railties-6.1.4.1/lib/rails/engine.rb:640:in `block (2 levels) in <class:Engine>'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.1.4.1/lib/active_support/callbacks.rb:126:in `instance_exec'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.1.4.1/lib/active_support/callbacks.rb:126:in `block in run_callbacks'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.1.4.1/lib/active_support/callbacks.rb:137:in `run_callbacks'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/railties-6.1.4.1/lib/rails/engine.rb:566:in `load_seed'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activerecord-6.1.4.1/lib/active_record/tasks/database_tasks.rb:450:in `load_seed'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activerecord-6.1.4.1/lib/active_record/railties/databases.rake:392:in `block (2 levels) in <main>'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/honeycomb-beeline-2.6.0/lib/honeycomb/integrations/rake.rb:21:in `block in execute'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/honeycomb-beeline-2.6.0/lib/honeycomb/client.rb:62:in `start_span'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/honeycomb-beeline-2.6.0/lib/honeycomb/integrations/rake.rb:16:in `execute'
seed_1       | /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/rake-13.0.6/exe/rake:27:in `<top (required)>'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/cli/exec.rb:63:in `load'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/cli/exec.rb:63:in `kernel_load'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/cli/exec.rb:28:in `run'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/cli.rb:494:in `exec'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/vendor/thor/lib/thor/command.rb:27:in `run'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/vendor/thor/lib/thor/invocation.rb:127:in `invoke_command'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/vendor/thor/lib/thor.rb:392:in `dispatch'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/cli.rb:30:in `dispatch'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/vendor/thor/lib/thor/base.rb:485:in `start'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/cli.rb:24:in `start'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/exe/bundle:49:in `block in <top (required)>'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/lib/bundler/friendly_errors.rb:130:in `with_friendly_errors'
seed_1       | /usr/local/share/gems/gems/bundler-2.2.15/exe/bundle:37:in `<top (required)>'
seed_1       | /usr/local/bin/bundle:23:in `load'
seed_1       | /usr/local/bin/bundle:23:in `<main>'
seed_1       | Tasks: TOP => db:seed
seed_1       | (See full trace by running task with --trace)
seed_1       | 2021/08/24 16:03:15 Command exited with error: exit status 1

```

The seeder code running when this happened was here

```ruby
##############################################################################

seeder.create_if_none(Badge) do
  5.times do
    Badge.create!(
      title: "#{Faker::Lorem.word} #{rand(100)}",
      description: Faker::Lorem.sentence,
      badge_image: File.open(Rails.root.join("app/assets/images/#{rand(1..40)}.png")),
    )
  end

  users_in_random_order.limit(10).each do |user|
    user.badge_achievements.create!(
      badge: Badge.order(Arel.sql("RANDOM()")).limit(1).take,
      rewarding_context_message_markdown: Faker::Markdown.random,
    )
  end
end

```

This suggests that `users_in_random_order` is nil? This is because the seed needs to run all at once \(if there are User's already in the system, the create if none causes a skip

```text
seed_1       |   3. Users already exist. Skipping.
seed_1       |   User with email = admin@forem.local found, skipping.

```

There's a second definition of `users_in_random_order` later, but it's inside a block and only the value from the user seed is available later. Seeds are not restartable if they have failed. 

In a command line world, what I might do is just call db:reset or db:schema:load to clear the data - here there are some options \(you could call the rake tasks using `docker run` on the forem\_rails container,  unfortunately just restarting gets the seed to detect badges exist and skipping the block completely

```text
seed_1       |   11. Badges already exist. Skipping.

```

This is probably non-critical - I think the context is that users will not be awarded badges from the seed 

```text
djuber@busybee:~/src/forem$ docker-compose run rails /bin/bash
Starting forem_bundle     ... done
Starting forem_postgresql ... done
Starting forem_yarn       ... done
Starting forem_redis      ... done
2021/08/24 16:15:10 Waiting for: tcp://db:5432
2021/08/24 16:15:10 Waiting for: tcp://redis:6379
2021/08/24 16:15:10 Waiting for: file:///opt/apps/forem/vendor/bundle/.bundle_finished
2021/08/24 16:15:10 Connected to tcp://redis:6379
2021/08/24 16:15:10 Connected to tcp://db:5432
2021/08/24 16:15:20 File file:///opt/apps/forem/vendor/bundle/.bundle_finished had been generated
[root@61c2f400073e forem]# pwd
/opt/apps/forem
[root@61c2f400073e forem]# bundle exec rails db:schema:load
I, [2021-08-24T16:15:35.891457 #34]  INFO -- : Allowing 172.24.0.1 for BetterErrors and Web Console

WARNING: sqlite and mysql triggers may not be shared by multiple actions
[root@61c2f400073e forem]# exit
2021/08/24 16:16:38 Command finished successfully.
djuber@busybee:~/src/forem$ docker-compose down
Stopping forem_redis      ... done
Stopping forem_postgresql ... done
Removing forem_rails_run_9a1910b1cefe ... done
Removing forem_seed                   ... done
Removing forem_webpacker              ... done
Removing forem_sidekiq                ... done
Removing forem_rails                  ... done
Removing forem_redis                  ... done
Removing forem_yarn                   ... done
Removing forem_bundle                 ... done
Removing forem_postgresql             ... done
Removing network forem_default

```

Now since the db:schema:load code ran, the seed should run end to end.



I made one local change to the badges block to ensure the badges are assigned to users in random order rather than failing if the user creation is skipped

```text
$ git diff db/seeds.rb 
diff --git a/db/seeds.rb b/db/seeds.rb
index 53ceec498..9ed182e78 100644
--- a/db/seeds.rb
+++ b/db/seeds.rb
@@ -407,6 +407,9 @@
     )
   end
 
+  # locally set this as we do below in listings in case the User model was skipped
+  users_in_random_order = User.order(Arel.sql("RANDOM()"))
+
   users_in_random_order.limit(10).each do |user|
     user.badge_achievements.create!(
       badge: Badge.order(Arel.sql("RANDOM()")).limit(1).take,
```

This wasn't strictly needed \(resetting the db was sufficient, since if all the seeds  run end to end there's no problem\)

```text
seed_1       |   12. Creating FeedbackMessages.
seed_1       |   13. Creating ListingCategories.
seed_1       |   14. Creating Listings.
seed_1       |   15. Creating Pages.
seed_1       |   16. Creating Sponsorships.
seed_1       |   17. NavigationLinks already exist. Skipping.
seed_1       | 
seed_1       |   ::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
seed_1       |   ::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
seed_1       |   ::::'########::'#######::'########::'########:'##::::'##::::
seed_1       |   :::: ##.....::'##.... ##: ##.... ##: ##.....:: ###::'###::::
seed_1       |   :::: ##::::::: ##:::: ##: ##:::: ##: ##::::::: ####'####::::
seed_1       |   :::: ######::: ##:::: ##: ########:: ######::: ## ### ##::::
seed_1       |   :::: ##...:::: ##:::: ##: ##.. ##::: ##...:::: ##. #: ##::::
seed_1       |   :::: ##::::::: ##:::: ##: ##::. ##:: ##::::::: ##:.:: ##::::
seed_1       |   :::: ##:::::::. #######:: ##:::. ##: ########: ##:::: ##::::
seed_1       |   ::::..:::::::::.......:::..:::::..::........::..:::::..:::::
seed_1       |   ::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
seed_1       | 
seed_1       |   All done!
seed_1       | 2021/08/24 16:19:03 Command finished successfully.

```

There are a lot of callbacks on the model creations that enqueued a pile of sidekiq jobs - but once that queue goes quiet \(podcast episode creation is the main item taking time\) you're just looking at a dockerized rails container.

This is even bound to 0.0.0.0 so if you're running docker on a remote host \(I am, in this case\) you'll still be able to use it - I can view the site at http://192.168.0.12:3000 from my desk \(not that address\). 



