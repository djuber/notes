---
description: Debugging an issue with the ruby 3 branch
---

# Rails runaway process in initialize!

This was an in-progress working document while I centered in on the issues with starting rails on the ruby 3 branch. The core issue was [https://bugs.ruby-lang.org/issues/17494\#note-9](https://bugs.ruby-lang.org/issues/17494#note-9) and is still pending \(our move to ruby 3.0 is blocked by this issue with refined method resolution. The patch from Jeremy Evans was effective when tested \(but has not been submitted to ruby-lang for inclusion in 3.0.x yet\).





Checked out the citizen428/ruby3 branch

bundle

run the following \(cribbed from boot or other binstubs to load the app\):

```text
require File.expand_path("config/application", __dir__)
@app = PracticalDeveloper::Application.new
@app.initialize! 
# rails freezes here and cpu goes wild
```

At this point rails freezes up - the cpu goes to 100% - strace records no activity - and a stack trace dump shows this:

```text
djuber@forem:~$ rbspy snapshot -p 3803
# SNIPPED all the code running pry - initialize! is the message I sent last

initialize! - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/application.rb:387
run_initializers - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/initializable.rb:64
tsort_each - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/tsort.rb:206
tsort_each - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/tsort.rb:233
each_strongly_connected_component - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/tsort.rb:355
call [c function] - (unknown):0
each [c function] - (unknown):0
block in each_strongly_connected_component - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/tsort.rb:353
each_strongly_connected_component_from - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/tsort.rb:435
block (2 levels) in each_strongly_connected_component - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/tsort.rb:351
block in tsort_each - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/tsort.rb:232
block in run_initializers - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/initializable.rb:62
run - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/initializable.rb:33
instance_exec [c function] - (unknown):0
block in <module:Finisher> - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/application/finisher.rb:141
run_load_hooks - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/lazy_load_hooks.rb:54
each [c function] - (unknown):0
block in run_load_hooks - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/lazy_load_hooks.rb:53
execute_hook - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/lazy_load_hooks.rb:77
with_execution_control - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/lazy_load_hooks.rb:63
block in execute_hook - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/lazy_load_hooks.rb:76
instance_eval [c function] - (unknown):0
block in patch_after_intialize - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/rails/patcher.rb:89
after_intialize - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/rails/patcher.rb:98
run - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/utils/only_once.rb:27
synchronize [c function] - (unknown):0
block in run - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/utils/only_once.rb:26
block in after_intialize - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/rails/patcher.rb:97
setup_tracer - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/rails/patcher.rb:103
setup - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/rails/framework.rb:50
configure - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/extensions.rb:57
each - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/set.rb:346
each_key [c function] - (unknown):0
block in configure - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/extensions.rb:51
patch - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/patchable.rb:56
patch - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/patcher.rb:35
run - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/utils/only_once.rb:27
synchronize [c function] - (unknown):0
block in run - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/utils/only_once.rb:26
block in patch - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/patcher.rb:34
patch - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/action_pack/patcher.rb:19
patch - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/patcher.rb:35
run - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/utils/only_once.rb:27
synchronize [c function] - (unknown):0
block in run - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/utils/only_once.rb:26
block in patch - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/patcher.rb:34
patch - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/action_pack/action_controller/patcher.rb:20
require - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:334
load_dependency - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:304
block in require - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:332
require - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/zeitwerk-2.4.2/lib/zeitwerk/kernel.rb:43
require - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:46
require_with_bootsnap_lfi - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:25
register - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/loaded_features_index.rb:114
block in require_with_bootsnap_lfi - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:24
require [c function] - (unknown):0
<main> - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/actionpack-6.1.3.1/lib/action_controller/metal.rb:8
require - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:334
load_dependency - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:304
block in require - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:332
require - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/zeitwerk-2.4.2/lib/zeitwerk/kernel.rb:43
require - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:46
require_with_bootsnap_lfi - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:25
register - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/loaded_features_index.rb:114
block in require_with_bootsnap_lfi - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:24
require [c function] - (unknown):0
<main> - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/actionpack-6.1.3.1/lib/action_dispatch/http/response.rb:8
<module:ActionDispatch> - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/actionpack-6.1.3.1/lib/action_dispatch/http/response.rb:540
<class:Response> - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/actionpack-6.1.3.1/lib/action_dispatch/http/response.rb:537
DelegateClass - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/delegate.rb:444
protected_instance_methods [c function] - (unknown):0

```



Record suggests `protected_instance_methods`  is the hotspot \(not sure if something else is happening here\). Initially the call stack included a wrapper for amazing print which I commented out in the gemfile - I'm going to remove bootsnap next and possibly zeitwerk.

If you comment out bootsnap - remove from config/boot.rb as well.





At this point I reverted all of my changes \(disabling gems and configurations\) and am going to do two things

* work inside out to get as close to the issue as possible
* start with a new rails app, using postgres, and not non-rails gems, to see if I can capture the same behavior and feed that back to Eilleen in [https://github.com/rails/rails/issues/38666](https://github.com/rails/rails/issues/38666)
* if I can't replicate this in a dummy app - how does our app differ from the dummy \(initializers, etc\). Feed this back to _our_ issue [https://github.com/forem/forem/pull/12103](https://github.com/forem/forem/pull/12103)

### Considerations

There doesn't seem to be any link between the reported yarn issue and the rails isse

Test case setup - nothing bad happened here:

```text
cd ~/src
rbenv local 3.0.0
gem install rails
# Successfully installed rails-6.1.3.1

# start a new project, using all defaults except psql:
rails new --database=postgresql testcase38666
... bundle bundle yarn yarn ... 
Webpacker successfully installed 🎉 🍰

djuber@forem:~/src$ cd testcase38666/
djuber@forem:~/src/testcase38666$ bin/setup 
== Installing dependencies ==
The Gemfile's dependencies are satisfied
 yarn install v1.22.10
 [1/4] Resolving packages...
 success Already up-to-date.
 Done in 0.36s.

== Preparing database ==
Created database 'testcase38666_development'
Created database 'testcase38666_test'

== Removing old logs and tempfiles ==

== Restarting application server ==
djuber@forem:~/src/testcase38666$ 
```

I noticed there was no bundle config in the test project, so I copied forem's

```text
djuber@forem:~/src/testcase38666$ mkdir -p .bundle; cp -av ../forem/.bundle/config .bundle/
djuber@forem:~/src/testcase38666$ bundle
djuber@forem:~/src/testcase38666$ bin/setup
# again all is well


djuber@forem:~/src/testcase38666$ cp -av ../forem/Gemfile .
djuber@forem:~/src/testcase38666$ bundle
djuber@forem:~/src/testcase38666$ bin/setup
== Installing dependencies ==
The Gemfile's dependencies are satisfied
 yarn install v1.22.10
 [1/4] Resolving packages...
 success Already up-to-date.
 Done in 0.36s.

== Preparing database ==

BINGO!!! This freezes - it's 100% one of our gems.
```

The existence of the test elasticsearch doesn't seem related - it's definitely hanging in initialize! and the fact that some particular rails command is getting called is not the issue. Booting the app is the problem here, and a loop or infinite loop is happening within.

Next refinement - comment out all group :development or group :test blocks - leaving only prod and the core app - rerun bundle - rerun setup. Effectively - everything after line 116 in the Gemfile is removed from bundle.

```text
Fetching gem metadata from https://rubygems.org/.......
Removing outdated .gem files from vendor/cache
  * amazing_print-1.3.0.gem
  * ast-2.4.2.gem
  * benchmark-ips-2.8.4.gem
  * better_errors-2.9.1.gem
  * better_html-1.0.16.gem
  * bindex-0.8.1.gem
  * brakeman-5.0.0.gem
  * bullet-6.1.4.gem
  * bundler-audit-0.8.0.gem
  * byebug-11.1.3.gem
  * capybara-3.35.3.gem
  * childprocess-3.0.0.gem
  * coderay-1.1.3.gem
  * crack-0.4.5.gem
  * cypress-rails-0.5.0.gem
  * dante-0.2.0.gem
  * dead_end-1.1.6.gem
  * derailed_benchmarks-2.0.1.gem
  * diff-lcs-1.4.4.gem
  * docile-1.3.5.gem
  * dotenv-2.7.6.gem
  * dotenv-rails-2.7.6.gem
  * em-websocket-0.5.2.gem
  * erb_lint-0.0.37.gem
  * eventmachine-1.2.7.gem
  * exifr-1.3.9.gem
  * factory_bot-6.1.0.gem
  * factory_bot_rails-6.1.0.gem
  * faker-2.17.0.gem
  * get_process_mem-0.2.7.gem
  * guard-2.16.2.gem
  * guard-compat-1.2.1.gem
  * guard-livereload-2.5.2.gem
  * hashdiff-1.0.1.gem
  * heapy-0.2.0.gem
  * html_tokenizer-0.0.7.gem
  * knapsack_pro-2.11.0.gem
  * launchy-2.5.0.gem
  * listen-3.5.1.gem
  * lumberjack-1.2.8.gem
  * memory_profiler-1.0.0.gem
  * mini_histogram-0.3.1.gem
  * nenv-0.3.0.gem
  * notiffany-0.1.3.gem
  * parser-3.0.1.0.gem
  * pry-0.13.1.gem
  * pry-byebug-3.9.0.gem
  * pry-rails-0.3.9.gem
  * pundit-matchers-1.6.0.gem
  * regexp_parser-2.1.1.gem
  * rspec-core-3.10.1.gem
  * rspec-expectations-3.10.1.gem
  * rspec-mocks-3.10.2.gem
  * rspec-rails-5.0.1.gem
  * rspec-retry-0.6.2.gem
  * rspec-support-3.10.2.gem
  * rubocop-1.12.1.gem
  * rubocop-ast-1.4.1.gem
  * rubocop-performance-1.10.2.gem
  * rubocop-rails-2.9.1.gem
  * rubocop-rspec-2.2.0.gem
  * ruby-prof-1.4.3.gem
  * ruby-progressbar-1.11.0.gem
  * ruby-statistics-2.1.3.gem
  * selenium-webdriver-3.142.7.gem
  * shellany-0.0.1.gem
  * shoulda-matchers-4.5.1.gem
  * simplecov-0.21.2.gem
  * simplecov-html-0.12.3.gem
  * simplecov_json_formatter-0.1.2.gem
  * smart_properties-1.15.0.gem
  * spring-2.1.1.gem
  * spring-commands-rspec-1.0.4.gem
  * stackprof-0.2.16.gem
  * stripe-ruby-mock-3.1.0.rc2.gem
  * test-prof-1.0.2.gem
  * timecop-0.9.4.gem
  * unicode-display_width-2.0.0.gem
  * uniform_notifier-1.14.2.gem
  * vcr-6.0.0.gem
  * web-console-4.1.0.gem
  * webdrivers-4.6.0.gem
  * webmock-3.12.2.gem
  * xpath-3.2.0.gem
  * yard-0.9.26.gem
  * yard-activerecord-0.0.16.gem
  * yard-activesupport-concern-0.0.1.gem
  * zonebie-0.6.1.gem
Bundle complete! 103 Gemfile dependencies, 251 gems now installed.
Bundled gems are installed into `./vendor/cache`
djuber@forem:~/src/testcase38666$ 
```

So this fails because listen is expected in development but not available \(at least it fails\):

```text
djuber@forem:~/src/testcase38666$ bin/setup     
== Installing dependencies ==
The Gemfile's dependencies are satisfied
 yarn install v1.22.10
 [1/4] Resolving packages...
 success Already up-to-date.
 Done in 0.36s.

== Preparing database ==
rails aborted!
LoadError: cannot load such file -- listen
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:23:in `require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:23:in `block in require_with_bootsnap_lfi'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/loaded_features_index.rb:89:in `register'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:22:in `require_with_bootsnap_lfi'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:44:in `require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/zeitwerk-2.4.2/lib/zeitwerk/kernel.rb:34:in `require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:332:in `block in require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:299:in `load_dependency'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:332:in `require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/evented_file_update_checker.rb:6:in `<main>'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:23:in `require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:23:in `block in require_with_bootsnap_lfi'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/loaded_features_index.rb:92:in `register'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:22:in `require_with_bootsnap_lfi'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:31:in `require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/zeitwerk-2.4.2/lib/zeitwerk/kernel.rb:34:in `require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:332:in `block in require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:299:in `load_dependency'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:332:in `require'
/home/djuber/src/testcase38666/config/environments/development.rb:72:in `block in <main>'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/railtie.rb:234:in `instance_eval'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/railtie.rb:234:in `configure'
/home/djuber/src/testcase38666/config/environments/development.rb:3:in `<main>'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:23:in `require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:23:in `block in require_with_bootsnap_lfi'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/loaded_features_index.rb:92:in `register'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:22:in `require_with_bootsnap_lfi'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:31:in `require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/zeitwerk-2.4.2/lib/zeitwerk/kernel.rb:34:in `require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:332:in `block in require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:299:in `load_dependency'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:332:in `require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/engine.rb:571:in `block (2 levels) in <class:Engine>'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/engine.rb:570:in `each'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/engine.rb:570:in `block in <class:Engine>'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/initializable.rb:32:in `instance_exec'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/initializable.rb:32:in `run'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/initializable.rb:61:in `block in run_initializers'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/initializable.rb:50:in `each'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/initializable.rb:50:in `tsort_each_child'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/initializable.rb:60:in `run_initializers'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/application.rb:384:in `initialize!'
/home/djuber/src/testcase38666/config/environment.rb:5:in `<main>'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:23:in `require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:23:in `block in require_with_bootsnap_lfi'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/loaded_features_index.rb:92:in `register'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:22:in `require_with_bootsnap_lfi'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:31:in `require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/zeitwerk-2.4.2/lib/zeitwerk/kernel.rb:34:in `require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:332:in `block in require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:299:in `load_dependency'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:332:in `require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/application.rb:360:in `require_environment!'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/application.rb:526:in `block in run_tasks_blocks'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:281:in `block in execute'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:281:in `each'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:281:in `execute'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/honeycomb-beeline-2.4.0/lib/honeycomb/integrations/rake.rb:14:in `execute'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:219:in `block in invoke_with_call_chain'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:199:in `synchronize'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:199:in `invoke_with_call_chain'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:243:in `block in invoke_prerequisites'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:241:in `each'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:241:in `invoke_prerequisites'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:218:in `block in invoke_with_call_chain'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:199:in `synchronize'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:199:in `invoke_with_call_chain'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:243:in `block in invoke_prerequisites'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:241:in `each'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:241:in `invoke_prerequisites'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:218:in `block in invoke_with_call_chain'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:199:in `synchronize'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:199:in `invoke_with_call_chain'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/task.rb:188:in `invoke'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/application.rb:160:in `invoke_task'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/application.rb:116:in `block (2 levels) in top_level'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/application.rb:116:in `each'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/application.rb:116:in `block in top_level'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/application.rb:125:in `run_with_threads'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/application.rb:110:in `top_level'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/commands/rake/rake_command.rb:24:in `block (2 levels) in perform'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/application.rb:186:in `standard_exception_handling'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/commands/rake/rake_command.rb:24:in `block in perform'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/rake-13.0.3/lib/rake/rake_module.rb:59:in `with_application'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/commands/rake/rake_command.rb:18:in `perform'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/command.rb:52:in `invoke'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/commands.rb:18:in `<main>'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:23:in `require'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:23:in `block in require_with_bootsnap_lfi'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/loaded_features_index.rb:92:in `register'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:22:in `require_with_bootsnap_lfi'
/home/djuber/src/testcase38666/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:31:in `require'
Tasks: TOP => db:prepare => db:load_config => environment
(See full trace by running task with --trace)

== Command ["bin/rails db:prepare"] failed ==
```

Let's _only_ add listen back, with no version restriction:

```ruby
group :development do
  gem "listen"
end
```

And test again

```text
djuber@forem:~/src/testcase38666$ bundle
Updating files in vendor/cache
  * listen-3.5.1.gem
Bundle complete! 104 Gemfile dependencies, 252 gems now installed.
Bundled gems are installed into `./vendor/cache`
djuber@forem:~/src/testcase38666$ bin/setup     
== Installing dependencies ==
The Gemfile's dependencies are satisfied
 yarn install v1.22.10
 [1/4] Resolving packages...
 success Already up-to-date.
 Done in 0.34s.

== Preparing database ==
```

And we're back were we started. I now have a run-away rails process. Seems like `listen` in development plus something in our gemset is problematic.



Can we isolate that

```ruby
# rails new --database=postgresql listener

# Bundle edge Rails instead: gem 'rails', github: 'rails/rails', branch: 'main'
gem 'rails', '~> 6.1.3', '>= 6.1.3.1'
# Use postgresql as the database for Active Record
gem 'pg', '~> 1.1'
# Use Puma as the app server
gem 'puma', '~> 5.0'
# Use SCSS for stylesheets
gem 'sass-rails', '>= 6'
# Transpile app-like JavaScript. Read more: https://github.com/rails/webpacker
gem 'webpacker', '~> 5.0'
# Turbolinks makes navigating your web application faster. Read more: https://github.com/turbolinks/turbolinks
gem 'turbolinks', '~> 5'
# Build JSON APIs with ease. Read more: https://github.com/rails/jbuilder
gem 'jbuilder', '~> 2.7'
# Use Redis adapter to run Action Cable in production
# gem 'redis', '~> 4.0'
# Use Active Model has_secure_password
# gem 'bcrypt', '~> 3.1.7'

# Use Active Storage variant
# gem 'image_processing', '~> 1.2'

# Reduces boot times through caching; required in config/boot.rb
gem 'bootsnap', '>= 1.4.4', require: false

group :development, :test do
  # Call 'byebug' anywhere in the code to stop execution and get a debugger console
  gem 'byebug', platforms: [:mri, :mingw, :x64_mingw]
end

group :development do
  # Access an interactive console on exception pages or by calling 'console' anywhere in the code.
  gem 'web-console', '>= 4.1.0'
  # Display performance information such as SQL time and flame graphs for each request in your browser.
  # Can be configured to work on production as well see: https://github.com/MiniProfiler/rack-mini-profiler/blob/master/README.md
  gem 'rack-mini-profiler', '~> 2.0'
  gem 'listen', '~> 3.3'
  # Spring speeds up development by keeping your application running in the background. Read more: https://github.com/rails/spring
  gem 'spring'
end

```

Listen is there by default in a vanilla rails program \(which  does _not_ exhibit this problem\).

Also, it's important to clarify what we've seen - the app won't boot without listen \(since that's configured in the dev env\) but freezes when listen _is_ put back, rather than pointing to listen being the problem, it only points to listen being required at dev time. I don't think this means what I wanted it to - only that _something_ in the gemset is a problem, and you have to keep listen installed to make that possible.

{% hint style="info" %}
Question at this point: why doesn't this happen on the Forem app \(ruby 2.7.2 and rails 6.1 with more or less the same gems\)? It's _either_ ruby 3 or a gem or both. 
{% endhint %}

What about commenting out all of the gems I can from the dummy app \(since apparently requiring the gems, without adding any initializer code, is sufficient to introduce this issue in a clean project\)? 



Commented out all  core \(non-group specific\) gems except bootsnap, pg, puma, rails, sprockets, webpacker

bundle, bin/setup, here we go!

```text
djuber@forem:~/src/testcase38666$ bundle
Fetching gem metadata from https://rubygems.org/............
Resolving dependencies...
Using rake 13.0.3
Using concurrent-ruby 1.1.8
Using minitest 5.14.4
Using zeitwerk 2.4.2
Using builder 3.2.4
Using erubi 1.10.0
Using racc 1.5.2
Using crass 1.0.6
Using ffi 1.15.0
Using rack 2.2.3
Using method_source 1.0.0
Using nio4r 2.5.7
Using pg 1.2.3
Using mini_mime 1.0.3
Using semantic_range 3.0.0
Using thor 1.1.0
Using tzinfo 2.0.4
Using nokogiri 1.11.3 (x86_64-linux)
Using rb-inotify 0.10.1
Using rack-test 1.1.0
Using rack-host-redirect 1.3.0
Using rack-proxy 0.6.5
Using sprockets 4.0.2
Using puma 5.2.2
Using loofah 2.9.1
Using mail 2.7.1
Using i18n 1.8.10
Using nakayoshi_fork 0.0.4
Using activesupport 6.1.3.1
Using marcel 1.0.1
Using rails-dom-testing 2.0.3
Using rb-fsevent 0.10.4
Using websocket-extensions 0.1.5
Using listen 3.5.1
Using rails-html-sanitizer 1.3.0
Using globalid 0.4.2
Using actionview 6.1.3.1
Using websocket-driver 0.7.3
Using actionpack 6.1.3.1
Using activejob 6.1.3.1
Using bundler 2.2.15
Using msgpack 1.4.2
Using activemodel 6.1.3.1
Using actioncable 6.1.3.1
Using actionmailer 6.1.3.1
Using railties 6.1.3.1
Using sprockets-rails 3.2.2
Using activerecord 6.1.3.1
Using bootsnap 1.7.3
Using webpacker 5.2.1
Using activestorage 6.1.3.1
Using hypershield 0.2.2
Using actionmailbox 6.1.3.1
Using actiontext 6.1.3.1
Using rails 6.1.3.1
Updating files in vendor/cache
Removing outdated .gem files from vendor/cache
  * active_record_union-1.3.0.gem
  * acts-as-taggable-on-7.0.0.gem
  * addressable-2.7.0.gem
  * ahoy_email-2.0.3.gem
  * ahoy_matey-3.2.0.gem
  * ancestry-3.2.1.gem
  * anyway_config-2.1.0.gem
  * aws-eventstream-1.1.1.gem
  * aws-sigv4-1.2.3.gem
  * aws_cf_signer-0.1.3.gem
  * axiom-types-0.1.1.gem
  * bcrypt-3.1.16.gem
  * blazer-2.4.2.gem
  * browser-5.3.1.gem
  * brpoplpush-redis_script-0.1.2.gem
  * buftok-0.2.0.gem
  * carrierwave-2.2.1.gem
  * carrierwave-bombshelter-0.2.2.gem
  * chartkick-4.0.2.gem
  * cloudinary-1.20.0.gem
  * coercible-1.0.0.gem
  * coffee-rails-5.0.0.gem
  * coffee-script-2.4.1.gem
  * coffee-script-source-1.12.2.gem
  * connection_pool-2.2.3.gem
  * counter_culture-2.8.0.gem
  * ddtrace-0.47.0.gem
  * descendants_tracker-0.0.4.gem
  * device_detector-1.0.5.gem
  * devise_invitable-2.0.3.gem
  * distribution-0.8.0.gem
  * dogstatsd-ruby-4.8.3.gem
  * domain_name-0.5.20190701.gem
  * doorkeeper-5.5.1.gem
  * elasticsearch-7.12.0.gem
  * elasticsearch-api-7.12.0.gem
  * elasticsearch-transport-7.12.0.gem
  * email_validator-2.2.3.gem
  * emoji_regex-3.2.2.gem
  * equalizer-0.0.11.gem
  * errbase-0.2.1.gem
  * et-orbi-1.2.4.gem
  * excon-0.79.0.gem
  * execjs-2.7.0.gem
  * faraday-1.3.0.gem
  * faraday-net_http-1.0.1.gem
  * fastimage-2.2.3.gem
  * fastly-3.0.1.gem
  * feedjira-3.1.2.gem
  * ffi-compiler-1.0.1.gem
  * field_test-0.4.1.gem
  * flipper-0.20.4.gem
  * flipper-active_record-0.20.4.gem
  * flipper-ui-0.20.4.gem
  * fog-aws-3.10.0.gem
  * fog-core-2.2.3.gem
  * fog-json-1.2.0.gem
  * fog-xml-0.1.3.gem
  * formatador-0.2.5.gem
  * front_matter_parser-1.0.0.gem
  * fugit-1.4.4.gem
  * gemoji-4.0.0.rc2.gem
  * geocoder-1.6.6.gem
  * gibbon-3.4.0.gem
  * hashie-4.1.0.gem
  * hiredis-0.6.3.gem
  * hkdf-0.3.0.gem
  * honeybadger-4.8.0.gem
  * honeycomb-beeline-2.4.0.gem
  * html_truncator-0.4.2.gem
  * htmlentities-4.3.4.gem
  * http-2-0.11.0.gem
  * http-4.4.1.gem
  * http-accept-1.7.0.gem
  * http-cookie-1.0.3.gem
  * http-form_data-2.3.0.gem
  * http-parser-1.2.3.gem
  * http_parser.rb-0.6.0.gem
  * httparty-0.18.1.gem
  * httpclient-2.8.3.gem
  * ice_nine-0.11.2.gem
  * image_processing-1.12.1.gem
  * imgproxy-2.0.0.gem
  * inline_svg-1.7.2.gem
  * ipaddress-0.8.3.gem
  * jbuilder-2.11.2.gem
  * jquery-fileupload-rails-0.4.7.gem
  * jquery-rails-4.4.0.gem
  * json-2.5.1.gem
  * jsonapi-serializer-2.2.0.gem
  * jwt-2.2.2.gem
  * kaminari-1.2.1.gem
  * kaminari-actionview-1.2.1.gem
  * kaminari-activerecord-1.2.1.gem
  * kaminari-core-1.2.1.gem
  * katex-0.6.1.gem
  * libhoney-1.18.0.gem
  * liquid-5.0.1.gem
  * memoizable-0.4.2.gem
  * mime-types-3.3.1.gem
  * mime-types-data-3.2021.0225.gem
  * mini_magick-4.11.0.gem
  * modis-3.3.0.gem
  * multi_json-1.15.0.gem
  * multi_xml-0.6.0.gem
  * multipart-post-2.1.1.gem
  * naught-1.1.0.gem
  * net-http-persistent-4.0.1.gem
  * net-http2-0.18.4.gem
  * netrc-0.11.0.gem
  * oauth-0.5.6.gem
  * oauth2-1.4.7.gem
  * octokit-4.20.0.gem
  * oj-3.11.3.gem
  * omniauth-2.0.4.gem
  * omniauth-apple-1.0.1.gem
  * omniauth-facebook-8.0.0.gem
  * omniauth-github-2.0.0.gem
  * omniauth-oauth-1.2.0.gem
  * omniauth-oauth2-1.7.1.gem
  * omniauth-rails_csrf_protection-1.0.0.gem
  * omniauth-twitter-1.4.0.gem
  * orm_adapter-0.5.0.gem
  * parallel-1.20.1.gem
  * patron-0.13.3.gem
  * pg_search-2.3.5.gem
  * public_suffix-4.0.6.gem
  * pundit-2.1.0.gem
  * pusher-2.0.1.gem
  * pusher-push-notifications-2.0.1.gem
  * pusher-signature-0.1.8.gem
  * raabro-1.4.0.gem
  * rack-attack-6.5.0.gem
  * rack-cors-1.1.1.gem
  * rack-protection-2.1.0.gem
  * rack-timeout-0.6.0.gem
  * rails-settings-cached-2.5.2.gem
  * rainbow-3.0.0.gem
  * ransack-2.4.2.gem
  * recaptcha-5.7.0.gem
  * redcarpet-3.5.1.gem
  * redis-4.2.5.gem
  * redis-actionpack-5.1.0.gem
  * redis-activesupport-5.2.0.gem
  * redis-rack-2.0.6.gem
  * redis-rails-5.0.2.gem
  * redis-store-1.9.0.gem
  * request_store-1.5.0.gem
  * responders-3.0.1.gem
  * rest-client-2.1.0.gem
  * reverse_markdown-2.0.0.gem
  * rexml-3.2.5.gem
  * rolify-5.3.0.gem
  * rouge-3.26.0.gem
  * rpush-5.4.0.gem
  * rpush-redis-1.1.0.gem
  * rss-0.2.9.gem
  * ruby-next-core-0.12.0.gem
  * ruby-vips-2.1.0.gem
  * ruby2_keywords-0.0.4.gem
  * rubyzip-2.3.0.gem
  * s3_direct_upload-0.1.7.gem
  * safely_block-0.3.0.gem
  * sass-3.7.4.gem
  * sass-listen-4.0.0.gem
  * sass-rails-6.0.0.gem
  * sassc-2.4.0.gem
  * sassc-rails-2.1.2.gem
  * sawyer-0.8.2.gem
  * sax-machine-1.3.2.gem
  * sidekiq-6.2.1.gem
  * sidekiq-cron-1.2.0.gem
  * sidekiq-unique-jobs-7.0.7.gem
  * simple_oauth-0.3.1.gem
  * sitemap_generator-6.1.2.gem
  * slack-notifier-2.3.2.gem
  * ssrf_filter-1.0.7.gem
  * staccato-0.5.3.gem
  * store_attribute-0.8.1.gem
  * storext-3.3.0.gem
  * stripe-5.32.1.gem
  * strong_migrations-0.7.6.gem
  * thread_safe-0.3.6.gem
  * tilt-2.0.10.gem
  * twitter-7.0.0.gem
  * uglifier-4.2.0.gem
  * ulid-1.3.0.gem
  * unf-0.1.4.gem
  * unf_ext-0.0.7.7.gem
  * validate_url-1.0.13.gem
  * vault-0.16.0.gem
  * virtus-1.0.5.gem
  * warden-1.2.9.gem
  * wcag_color_contrast-0.1.0.gem
  * webpush-1.1.0.gem
Removing outdated git and path gems from vendor/cache
  * acts_as_follower-06393d3693a1
  * devise-0cd72a56f984
Bundle complete! 10 Gemfile dependencies, 55 gems now installed.
Bundled gems are installed into `.
```

This still includes the gems labeled production \(including the nakayoshi fork\) and listen, and rails and its dependencies.



```text
djuber@forem:~/src/testcase38666$ bin/setup     
== Installing dependencies ==
The Gemfile's dependencies are satisfied
 yarn install v1.22.10
 [1/4] Resolving packages...
 success Already up-to-date.
 Done in 0.36s.

== Preparing database ==

== Removing old logs and tempfiles ==

== Restarting application server ==
```

Okay, this is helpful - somewhere between 200 gems and 55 gems is our problem.

Bisection search to the rescue! Uncomment down to fastly - 108 gems now installed - all is well - rails db:prepare works fine.

Uncomment down to jsonapi-serializer. bundle. 145 gems now installed. rails db:prepare still works.

Uncomment down to redis-actionpack, bundle, 187 gems now installed - freezes

Between jsonapi-serializer and redis-actionpack is an issue.

This included these \(except pg, puma, and rails, which were not skipped initially\) - one of these, especially if one of them is injecting itself into action dispatch, is a problem.

```ruby
gem "kaminari", "~> 1.2" # A Scope & Engine based, clean, powerful, customizable and sophisticated paginator
gem "katex", "~> 0.6.1" # This rubygem enables you to render TeX math to HTML using KaTeX. It uses ExecJS under the hood
gem "liquid", "~> 5.0" # A secure, non-evaling end user template engine with aesthetic markup
gem "nokogiri", "~> 1.11" # HTML, XML, SAX, and Reader parser
gem "octokit", "~> 4.20" # Simple wrapper for the GitHub API
gem "oj", "~> 3.11" # JSON parser and object serializer
gem "omniauth", "~> 2.0" # A generalized Rack framework for multiple-provider authentication
gem "omniauth-apple", "~> 1.0" # OmniAuth strategy for Sign In with Apple
gem "omniauth-facebook", "~> 8.0" # OmniAuth strategy for Facebook
gem "omniauth-github", "~> 2.0" # OmniAuth strategy for GitHub
gem "omniauth-rails_csrf_protection", "~> 1.0" # Provides CSRF protection on OmniAuth request endpoint on Rails application.
gem "omniauth-twitter", "~> 1.4" # OmniAuth strategy for Twitter
gem "parallel", "~> 1.20" # Run any kind of code in parallel processes
gem "patron", "~> 0.13.3" # HTTP client library based on libcurl, used with Elasticsearch to support http keep-alive connections
gem "pg", "~> 1.2" # Pg is the Ruby interface to the PostgreSQL RDBMS
gem "pg_search", "~> 2.3.5" # PgSearch builds Active Record named scopes that take advantage of PostgreSQL's full text search
gem "puma", "~> 5.2.2" # Puma is a simple, fast, threaded, and highly concurrent HTTP 1.1 server
gem "pundit", "~> 2.1" # Object oriented authorization for Rails applications
gem "pusher", "~> 2.0" # Ruby library for Pusher Channels HTTP API
gem "pusher-push-notifications", "~> 2.0" # Pusher Push Notifications Ruby server SDK
gem "rack-attack", "~> 6.5.0" # Used to throttle requests to prevent brute force attacks
gem "rack-cors", "~> 1.1" # Middleware that will make Rack-based apps CORS compatible
gem "rack-timeout", "~> 0.6" # Rack middleware which aborts requests that have been running for longer than a specified timeout
gem "rails", "~> 6.1" # Ruby on Rails
gem "rails-settings-cached", ">= 2.1.1" # Settings plugin for Rails that makes managing a table of global key, value pairs easy.
gem "ransack", "~> 2.4" # Searching and sorting
gem "recaptcha", "~> 5.7", require: "recaptcha/rails" # Helpers for the reCAPTCHA API
gem "redcarpet", "~> 3.5" # A fast, safe and extensible Markdown to (X)HTML parser
gem "redis", "~> 4.2.5" # Redis ruby client
gem "redis-actionpack", "5.1.0" # Redis session store for ActionPack. Used for storing the Rails session in Redis.
```

Recomment down to pg \(so kaminari through patron\). bundle - 164 gems installed. setup: Success

Uncomment omniauth to patron \(leaving kaminari through oj commented\). bundle. 177 gems now installed. setup hangs

Comment omniauth\* gems \(omniauth through omniauth-twitter, 6 gems total\). 166 gems now installed. Setup succeeds.

Uncomment _only_ omniauth. brings in hashie as a dependency. Works.

Uncomment the four providers \(apple, facebook, github, twitter\) leaving the csrf protection gem commented.

```text
Updating files in vendor/cache
  * omniauth-apple-1.0.1.gem
  * omniauth-facebook-8.0.0.gem
  * omniauth-github-2.0.0.gem
  * omniauth-twitter-1.4.0.gem
  * omniauth-oauth-1.2.0.gem
  * omniauth-oauth2-1.7.1.gem
  * oauth2-1.4.7.gem
  * oauth-0.5.6.gem
Bundle complete! 70 Gemfile dependencies, 176 gems now installed.
Bundled gems are installed into `./vendor/cache`
djuber@forem:~/src/testcase38666$ bin/setup
== Installing dependencies ==
The Gemfile's dependencies are satisfied
 yarn install v1.22.10
 [1/4] Resolving packages...
 success Already up-to-date.
 Done in 0.35s.

== Preparing database ==

```

This froze as well during loading. So it's one of those four I guess. This is really good stuff.

```text
Removing outdated .gem files from vendor/cache
  * oauth-0.5.6.gem
  * oauth2-1.4.7.gem
  * omniauth-apple-1.0.1.gem
  * omniauth-facebook-8.0.0.gem
  * omniauth-github-2.0.0.gem
  * omniauth-oauth-1.2.0.gem
  * omniauth-oauth2-1.7.1.gem
  * omniauth-twitter-1.4.0.gem
Bundle complete! 66 Gemfile dependencies, 168 gems now installed.
Bundled gems are installed into `./vendor/cache`
djuber@forem:~/src/testcase38666$ 
```

Just backing up to the last known safe point to confirm we're okay \(we are\). I'm going to uncomment everything except these four gems before I go any closer \(I want to ensure this isn't one of many problems, that it's the one and only problem, before I go too much farther\). I hit a snag running bundle when I added redis-rack back - it looks like without the explicit versioning it floated up to 2.1.3 - deleting the Gemfile.lock and bundling fresh works.

Restating the situation:

all the test/development convenience gems are commented out - listen is installed in development as a dependency \(installed by vanilla rails\) - all "normal" or top level gems, except the four omniauth providers, are enabled.

```text
Removing outdated .gem files from vendor/cache
  * redis-rack-2.1.3.gem
Bundle complete! 99 Gemfile dependencies, 243 gems now installed.
Bundled gems are installed into `./vendor/cache`
djuber@forem:~/src/testcase38666$ 
```

Setup works - so it's \(definitely?\) one of the gems - or one of it's dependencies.

Let's install the dependencies first \(oauth2 and oauth\) one by one to control for effects, before we enable the  omniauth providers.



```text
gem "omniauth-oauth", "1.2.0"


Updating files in vendor/cache
  * omniauth-oauth-1.2.0.gem
  * oauth-0.5.6.gem
Bundle complete! 100 Gemfile dependencies, 245 gems now installed.
Bundled gems are installed into `./vendor/cache`
```

Freezes. `rbspy snapshot` shows the same location - `protected_instance_methods [c function]`

Switching to my "mostly clean" gemfile in listener and adding that single gem to the default install brings in these gems \(but does not introduce the problem\):

```text
Fetching hashie 4.1.0
Fetching rack-protection 2.1.0
Fetching oauth 0.5.6
Installing rack-protection 2.1.0
Installing oauth 0.5.6
Installing hashie 4.1.0
Fetching omniauth 2.0.4
Installing omniauth 2.0.4
Fetching omniauth-oauth 1.2.0
Installing omniauth-oauth 1.2.0
Bundle complete! 18 Gemfile dependencies, 78 gems now installed.
Use `bundle info [gemname]` to see where a bundled gem is installed.

djuber@forem:~/src/listener$ bin/setup
== Installing dependencies ==
The Gemfile's dependencies are satisfied
yarn install v1.22.10
[1/4] Resolving packages...
success Already up-to-date.
Done in 0.38s.

== Preparing database ==
Created database 'listener_development'
Created database 'listener_test'

== Removing old logs and tempfiles ==

== Restarting application server ==

```

So it looks like it's not _only_ this omniauth-oauth gem but some interaction between something else \(which is frustrating, but fine.\) Before I go any farther, I'm going to relax any version constraints on the omniauth-oauth gem, and call update \(\`bundler attempted to update omniauth-oauth, but its version stayed the same\). Well that's too bad, there's not some silver bullet laying around uninstalled.



Remove omniauth-oauth gem, add omniauth-oauth2, leave oauth installed since that didn't seem to be the problem.

```text

Updating files in vendor/cache
  * omniauth-oauth2-1.7.1.gem
  * oauth2-1.4.7.gem
Removing outdated .gem files from vendor/cache
  * omniauth-oauth-1.2.0.gem
Bundle complete! 101 Gemfile dependencies, 246 gems now installed.
Bundled gems are installed into `./vendor/cache`
djuber@forem:~/src/testcase38666$ 
```

Reading the code that _looks_ like it's related to where things are going awry - we have `class Response` which has a Header which is a `DelegateClass(Hash)`  which appears to be the next thing called

```ruby
def DelegateClass(superclass, &block)
  klass = Class.new(Delegator)
  ignores = [*::Delegator.public_api, :to_s, :inspect, :=~, :!~, :===]
  protected_instance_methods = superclass.protected_instance_methods
  protected_instance_methods -= ignores
  public_instance_methods = superclass.public_instance_methods
  public_instance_methods -= ignores
  klass.module_eval do
```

Fairly sure this call to `protected_instance_methods`  is part of the problem \(or definitely the thing going wrong\) - is there something with a Hash causing issues? Did more than one thing define a DelegateClass in some cyclical way? Why would that break Rails?

{% hint style="info" %}
Hashie defines a Hash class - we want to make sure that's not what Rails is trying to wrap the response headers in. Hashie also doesn't have a test suite against ruby 3 yet? [https://github.com/hashie/hashie/releases/tag/v4.1.0](https://github.com/hashie/hashie/releases/tag/v4.1.0) is what we have in the gemset.
{% endhint %}

Omniauth includes Hashie as a dependency.

Buffer _also_ includes Hashie as a dependency  through faraday\_middleware:

```text

Installing yajl-ruby 1.4.1 with native extensions
Installing faraday_middleware 1.0.0
Fetching buffer 0.1.3
Installing buffer 0.1.3
Updating files in vendor/cache
Fetching buffer 0.1.3
  * buffer-0.1.3.gem
Fetching environs 1.1.0
  * environs-1.1.0.gem
Fetching faraday_middleware 1.0.0
  * faraday_middleware-1.0.0.gem
  * hashie-4.1.0.gem
Fetching yajl-ruby 1.4.1
  * yajl-ruby-1.4.1.gem
Bundle complete! 145 Gemfile dependencies, 333 gems now installed.
Bundled gems are installed into `./vendor/cache`


djuber@forem:~/src/testcase38666$ diff Gemfile ../forem/Gemfile
22d21
< gem "buffer", "~> 0.1"
61,66c60,65
< # gem "omniauth", "~> 2.0" # A generalized Rack framework for multiple-provider authentication
< # gem "omniauth-apple", "~> 1.0" # OmniAuth strategy for Sign In with Apple
< # gem "omniauth-facebook", "~> 8.0" # OmniAuth strategy for Facebook
< # gem "omniauth-github", "~> 2.0" # OmniAuth strategy for GitHub
< # gem "omniauth-rails_csrf_protection", "~> 1.0" # Provides CSRF protection on OmniAuth request endpoint on Rails application.
< # gem "omniauth-twitter", "~> 1.4" # OmniAuth strategy for Twitter
---
> gem "omniauth", "~> 2.0" # A generalized Rack framework for multiple-provider authentication
> gem "omniauth-apple", "~> 1.0" # OmniAuth strategy for Sign In with Apple
> gem "omniauth-facebook", "~> 8.0" # OmniAuth strategy for Facebook
> gem "omniauth-github", "~> 2.0" # OmniAuth strategy for GitHub
> gem "omniauth-rails_csrf_protection", "~> 1.0" # Provides CSRF protection on OmniAuth request endpoint on Rails application.
> gem "omniauth-twitter", "~> 1.4" # OmniAuth strategy for Twitter
```

however - bin/setup worked here \(leaving omniauth commented-  including buffer which was removed in the last commit\).

```text
djuber@forem:~/src/testcase38666$ diff Gemfile.lock ../forem/Gemfile.lock -u
--- Gemfile.lock        2021-04-08 11:01:55.454230874 -0500
+++ ../forem/Gemfile.lock       2021-04-08 08:43:48.731973037 -0500
@@ -86,7 +86,7 @@
       activerecord (>= 5.0, < 6.2)
     addressable (2.7.0)
       public_suffix (>= 2.0.2, < 5.0)
-    ahoy_email (2.0.3)
+    ahoy_email (2.0.2)
       actionmailer (>= 5)
       addressable (>= 2.3.2)
       nokogiri
@@ -137,15 +137,6 @@
     brpoplpush-redis_script (0.1.2)
       concurrent-ruby (~> 1.0, >= 1.0.5)
       redis (>= 1.0, <= 5.0)
-    buffer (0.1.3)
-      addressable
-      environs
-      faraday
-      faraday_middleware
-      hashie
-      multi_json
-      rake
-      yajl-ruby
     buftok (0.2.0)
     builder (3.2.4)
     bullet (6.1.4)
@@ -175,7 +166,7 @@
       activesupport (>= 3.2.0)
       carrierwave
       fastimage
-    chartkick (4.0.2)
+    chartkick (3.4.2)
     childprocess (3.0.0)
     cloudinary (1.20.0)
       aws_cf_signer
@@ -249,7 +240,6 @@
     email_validator (2.2.3)
       activemodel
     emoji_regex (3.2.2)
-    environs (1.1.0)
     equalizer (0.0.11)
     erb_lint (0.0.37)
       activesupport
@@ -279,8 +269,6 @@
       multipart-post (>= 1.2, < 3)
       ruby2_keywords
     faraday-net_http (1.0.1)
-    faraday_middleware (1.0.0)
-      faraday (~> 1.0)
     fastimage (2.2.3)
     fastly (3.0.1)
     feedjira (3.1.2)
@@ -433,13 +421,13 @@
     listen (3.5.1)
       rb-fsevent (~> 0.10, >= 0.10.3)
       rb-inotify (~> 0.9, >= 0.9.10)
-    loofah (2.9.1)
+    loofah (2.9.0)
       crass (~> 1.0.2)
       nokogiri (>= 1.5.9)
     lumberjack (1.2.8)
     mail (2.7.1)
       mini_mime (>= 0.1.1)
-    marcel (1.0.1)
+    marcel (1.0.0)
     memoizable (0.4.2)
       thread_safe (~> 0.3, >= 0.3.1)
     memory_profiler (1.0.0)
@@ -471,18 +459,53 @@
       http-2 (~> 0.11)
     netrc (0.11.0)
     nio4r (2.5.7)
-    nokogiri (1.11.3-x86_64-linux)
+    nokogiri (1.11.2-arm64-darwin)
+      racc (~> 1.4)
+    nokogiri (1.11.2-x86_64-darwin)
+      racc (~> 1.4)
+    nokogiri (1.11.2-x86_64-linux)
       racc (~> 1.4)
     notiffany (0.1.3)
       nenv (~> 0.1)
       shellany (~> 0.0)
+    oauth (0.5.5)
+    oauth2 (1.4.7)
+      faraday (>= 0.8, < 2.0)
+      jwt (>= 1.0, < 3.0)
+      multi_json (~> 1.3)
+      multi_xml (~> 0.5)
+      rack (>= 1.2, < 3)
     octokit (4.20.0)
       faraday (>= 0.9)
       sawyer (~> 0.8.0, >= 0.5.3)
     oj (3.11.3)
+    omniauth (2.0.3)
+      hashie (>= 3.4.6)
+      rack (>= 1.6.2, < 3)
+      rack-protection
+    omniauth-apple (1.0.1)
+      jwt
+      omniauth-oauth2
+    omniauth-facebook (8.0.0)
+      omniauth-oauth2 (~> 1.2)
+    omniauth-github (2.0.0)
+      omniauth (~> 2.0)
+      omniauth-oauth2 (~> 1.7.1)
+    omniauth-oauth (1.2.0)
+      oauth
+      omniauth (>= 1.0, < 3)
+    omniauth-oauth2 (1.7.1)
+      oauth2 (~> 1.4)
+      omniauth (>= 1.9, < 3)
+    omniauth-rails_csrf_protection (1.0.0)
+      actionpack (>= 4.2)
+      omniauth (~> 2.0)
+    omniauth-twitter (1.4.0)
+      omniauth-oauth (~> 1.1)
+      rack
     orm_adapter (0.5.0)
     parallel (1.20.1)
-    parser (3.0.1.0)
+    parser (3.0.0.0)
       ast (~> 2.4.1)
     patron (0.13.3)
     pg (1.2.3)
@@ -696,7 +719,7 @@
     shellany (0.0.1)
     shoulda-matchers (4.5.1)
       activesupport (>= 4.2.0)
-    sidekiq (6.2.1)
+    sidekiq (6.2.0)
       connection_pool (>= 2.2.2)
       rack (~> 2.0)
       redis (>= 4.2.0)
@@ -810,7 +833,6 @@
     websocket-extensions (0.1.5)
     xpath (3.2.0)
       nokogiri (~> 1.8)
-    yajl-ruby (1.4.1)
     yard (0.9.26)
     yard-activerecord (0.0.16)
       yard (>= 0.8.3)
@@ -820,6 +842,10 @@
     zonebie (0.6.1)
 
 PLATFORMS
+  arm64-darwin-20
+  x86_64-darwin
+  x86_64-darwin-19
+  x86_64-darwin-20
   x86_64-linux
 
 DEPENDENCIES
@@ -835,7 +861,6 @@
   blazer (~> 2.4.2)
   bootsnap (>= 1.1.0)
   brakeman (~> 5.0)
-  buffer (~> 0.1)
   bullet (~> 6.1)
   bundler-audit (~> 0.8)
   capybara (~> 3.35.3)
@@ -892,6 +917,12 @@
   nokogiri (~> 1.11)
   octokit (~> 4.20)
   oj (~> 3.11)
+  omniauth (~> 2.0)
+  omniauth-apple (~> 1.0)
+  omniauth-facebook (~> 8.0)
+  omniauth-github (~> 2.0)
+  omniauth-rails_csrf_protection (~> 1.0)
+  omniauth-twitter (~> 1.4)
   parallel (~> 1.20)
   patron (~> 0.13.3)
   pg (~> 1.2)
```

Remove imgproxy, leave buffer, uncomment omniauth.

```text
Updating files in vendor/cache
  * omniauth-2.0.4.gem
  * omniauth-apple-1.0.1.gem
  * omniauth-facebook-8.0.0.gem
  * omniauth-github-2.0.0.gem
  * omniauth-rails_csrf_protection-1.0.0.gem
  * omniauth-twitter-1.4.0.gem
  * omniauth-oauth-1.2.0.gem
  * omniauth-oauth2-1.7.1.gem
  * oauth2-1.4.7.gem
  * oauth-0.5.6.gem
Removing outdated .gem files from vendor/cache
  * anyway_config-2.1.0.gem
  * imgproxy-2.0.0.gem
  * ruby-next-core-0.12.0.gem
Bundle complete! 150 Gemfile dependencies, 340 gems now installed.
Bundled gems are installed into `./vendor/cache`
djuber@forem:~/src/testcase38666$ ruby --version
ruby 3.0.0p0 (2020-12-25 revision 95aff21468) [x86_64-linux]
djuber@forem:~/src/testcase38666$ 
```

Works.

Can I introduce the issue by including imgproxy and omniauth-twitter in listener?



```text

# copied from forem's gemfile:
gem 'buffer', '~> 0.1'
gem 'omniauth'
gem 'omniauth-google'
gem 'omniauth-facebook'
gem 'omniauth-twitter'
gem 'omniauth-oauth', '1.2.0'
gem 'imgproxy', '2.0.0'


djuber@forem:~/src/listener$ bin/setup
== Installing dependencies ==
Bundler can't satisfy your Gemfile's dependencies.
Install missing gems with `bundle install`.
Fetching gem metadata from https://rubygems.org/..........
Resolving dependencies...
Bundler could not find compatible versions for gem "omniauth":
  In snapshot (Gemfile.lock):
    omniauth (= 2.0.4)

  In Gemfile:
    omniauth

    omniauth-google was resolved to 1.0.1, which depends on
      omniauth (~> 1.0.0)

    omniauth-oauth (= 1.2.0) was resolved to 1.2.0, which depends on
      omniauth (>= 1.0, < 3)

Running `bundle update` will rebuild your snapshot from scratch, using only
the gems in your Gemfile, which may resolve the conflict.

== Command ["bundle install"] failed ==
djuber@forem:~/src/listener$ rm Gemfile.lock 
djuber@forem:~/src/listener$ bin/setup== Installing dependencies ==
Bundler can't satisfy your Gemfile's dependencies.
Install missing gems with `bundle install`.

```

Rewinding the state - checking against current master for differences in the Gemfile \(aggravatingly I got this working with a slightly different Gemfile.lock\)



```text
# question my life choices about removing the line numbers 
djuber@forem:~/src/testcase38666$ diff ../testcase38666/Gemfile ../forem-for-docker/Gemfile -u
--- ../testcase38666/Gemfile    2021-04-08 11:15:50.980514232 -0500
+++ ../forem-for-docker/Gemfile 2021-04-08 11:18:07.826752864 -0500

-gem "buffer", "~> 0.1"
-gem "rss", "~> 0.2" # Ruby's offical RSS parser
-gem "sidekiq", "~> 6.2.0" # Sidekiq is used to process background jobs with the help of Redis
+gem "sidekiq", "~> 6.2.1" # Sidekiq is used to process background jobs with the help of Redis
 
+
+  # NOTE: [@rhymes] binding_of_caller 1.0 breaks Docker Compose, see <https://github.com/forem/forem/issues/12068>
+  gem "binding_of_caller", "~> 0.8" # Retrieve the binding of a method's caller

-  gem "pry-byebug", "~> 3.9" # Combine 'pry' with 'byebug'. Adds 'step', 'next', 'finish', 'continue' and 'break' commands to control execution
+  gem "pry-byebug", "~> 3.8" # Combine 'pry' with 'byebug'. Adds 'step', 'next', 'finish', 'continue' and 'break' commands to control execution

```

