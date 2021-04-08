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
<main> - /home/djuber/.rbenv/versions/3.0.0/bin/bundle:26
load [c function] - (unknown):0
<top (required)> - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/gems/3.0.0/gems/bundler-2.2.15/exe/bundle:37
with_friendly_errors - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/gems/3.0.0/gems/bundler-2.2.15/lib/bundler/friendly_errors.rb:138
block in <top (required)> - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/gems/3.0.0/gems/bundler-2.2.15/exe/bundle:50
start - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/gems/3.0.0/gems/bundler-2.2.15/lib/bundler/cli.rb:27
start - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/gems/3.0.0/gems/bundler-2.2.15/lib/bundler/vendor/thor/lib/thor/base.rb:495
dispatch - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/gems/3.0.0/gems/bundler-2.2.15/lib/bundler/cli.rb:34
dispatch - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/gems/3.0.0/gems/bundler-2.2.15/lib/bundler/vendor/thor/lib/thor.rb:393
invoke_command - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/gems/3.0.0/gems/bundler-2.2.15/lib/bundler/vendor/thor/lib/thor/invocation.rb:129
run - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/gems/3.0.0/gems/bundler-2.2.15/lib/bundler/vendor/thor/lib/thor/command.rb:37
exec - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/gems/3.0.0/gems/bundler-2.2.15/lib/bundler/cli.rb:495
run - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/gems/3.0.0/gems/bundler-2.2.15/lib/bundler/cli/exec.rb:35
kernel_load - /home/djuber/.rbenv/versions/3.0.0/lib/ruby/gems/3.0.0/gems/bundler-2.2.15/lib/bundler/cli/exec.rb:70
load [c function] - (unknown):0
<top (required)> - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/bin/pry:26
load [c function] - (unknown):0
<top (required)> - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/pry-0.13.1/bin/pry:13
start - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/pry-0.13.1/lib/pry/cli.rb:120
start_with_pry_byebug - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/pry-byebug-3.9.0/lib/pry-byebug/pry_ext.rb:15
start - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/pry-0.13.1/lib/pry/pry_class.rb:195
start - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/pry-0.13.1/lib/pry/repl.rb:16
start - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/pry-0.13.1/lib/pry/repl.rb:41
with_ownership - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/pry-0.13.1/lib/pry/input_lock.rb:79
__with_ownership - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/pry-0.13.1/lib/pry/input_lock.rb:73
block in start - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/pry-0.13.1/lib/pry/repl.rb:38
repl - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/pry-0.13.1/lib/pry/repl.rb:80
loop [c function] - (unknown):0
block in repl - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/pry-0.13.1/lib/pry/repl.rb:79
eval - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/pry-0.13.1/lib/pry/pry_instance.rb:277
catch [c function] - (unknown):0
block in eval - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/pry-0.13.1/lib/pry/pry_instance.rb:268
catch [c function] - (unknown):0
block (2 levels) in eval - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/pry-0.13.1/lib/pry/pry_instance.rb:266
handle_line - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/pry-0.13.1/lib/pry/pry_instance.rb:677
evaluate_ruby - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/pry-0.13.1/lib/pry/pry_instance.rb:295
eval [c function] - (unknown):0
__pry__ - (pry):3
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

