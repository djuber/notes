---
description: Debugging an issue with the ruby 3 branch
---

# Rails runaway process in initialize!

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



