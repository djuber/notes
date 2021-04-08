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
djuber@forem:~/src/testcase38666$ bin/setup
# again all is well

```

The existence of the test elasticsearch doesn't seem related - it's definitely hanging in initialize! and the fact that some particular rails command is getting called is not the issue. Booting the app is the problem here, and a loop or infinite loop is happening within.

Will attempt to close in on the problem using pry instead.

