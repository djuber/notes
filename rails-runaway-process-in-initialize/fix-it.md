---
description: Maybe this works
---

# fix it?

I started to see the setup work when I ran it in the testcase directory using what looked like the same gemset - the only difference appears to be missing the nokogiri versions for mac \(since I started with no lockfile this makes sense\):

```text
djuber@forem:~/src/testcase38666$ diff -u ../forem/Gemfile.lock Gemfile.lock
--- ../forem/Gemfile.lock       2021-04-08 11:31:56.620181461 -0500
+++ Gemfile.lock        2021-04-08 11:33:08.199653760 -0500
@@ -459,10 +459,6 @@
       http-2 (~> 0.11)
     netrc (0.11.0)
     nio4r (2.5.7)
-    nokogiri (1.11.3-arm64-darwin)
-      racc (~> 1.4)
-    nokogiri (1.11.3-x86_64-darwin)
-      racc (~> 1.4)
     nokogiri (1.11.3-x86_64-linux)
       racc (~> 1.4)
     notiffany (0.1.3)
```

```text
bundle update chartkick ahoy_email marcel oauth omniauth parser sidekiq 
```



This gets the forem gemfile.lock looking very like the one  in my environment - however I'm still getting the same issue \(well, _almost_ the same - it's stilli happening when trying to get the class information for Hash, but the location moved, this could be completely related to the initializers and their ordering\):



```text
require [c function] - (unknown):0
<main> - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/rspec-core-3.10.1/lib/rspec/core/metadata.rb:1
<module:RSpec> - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/rspec-core-3.10.1/lib/rspec/core/metadata.rb:498
<module:Core> - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/rspec-core-3.10.1/lib/rspec/core/metadata.rb:497
<module:HashImitatable> - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/rspec-core-3.10.1/lib/rspec/core/metadata.rb:445
block (2 levels) in <class:Class> - /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/amazing_print-1.3.0/lib/amazing_print/core_ext/class.rb:23
call [c function] - (unknown):0
public_instance_methods [c function] - (unknown):0
```



### When all else fails - debug it!

```c

static inline rb_method_entry_t*
search_method(VALUE klass, ID id, VALUE *defined_class_ptr)
{
    rb_method_entry_t *me = NULL;

    RB_DEBUG_COUNTER_INC(mc_search);

    for (; klass; klass = RCLASS_SUPER(klass)) {
	RB_DEBUG_COUNTER_INC(mc_search_super);
        if ((me = lookup_method_table(klass, id)) != 0) {
            break;
        }
    }

    if (defined_class_ptr) *defined_class_ptr = klass;

    if (me == NULL) RB_DEBUG_COUNTER_INC(mc_search_notfound);

    VM_ASSERT(me == NULL || !METHOD_ENTRY_INVALIDATED(me));
    return me;
}
```

The critical path that's causing the issue is this for loop in search\_method 

The debug counters are in debug\_counter.h \(and are guarded by a compile time define so might just be a cast to void\(0\); 



### Meanwhile in ruby

I added a pry breakpoint to the define\_method for AmazingPrint::Class core extension

```text
[1] pry(Hash)> backtrace
--> #0  block (2 levels) in Class.block (2 levels) in <class:Class>(*args#Array) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/amazing_print-1.3.0/lib/amazing_print/core_ext/class.rb:20
    #1  Object.DelegateClass(superclass#Class, &block#NilClass) at /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/delegate.rb:397
    #2  <class:Response> at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/actionpack-6.1.3.1/lib/action_dispatch/http/response.rb:38
    #3  <module:ActionDispatch> at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/actionpack-6.1.3.1/lib/action_dispatch/http/response.rb:37
    #4  <main> at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/actionpack-6.1.3.1/lib/action_dispatch/http/response.rb:8
    ͱ-- #5  Kernel.require at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:23
    #6  block in Kernel.block in require_with_bootsnap_lfi(path#String, resolved#String) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:23
    #7  Bootsnap::LoadPathCache::LoadedFeaturesIndex.register(short#String, long#String) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/loaded_features_index.rb:92
    #8  Kernel.require_with_bootsnap_lfi(path#String, resolved#String) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:22
    #9  Kernel.require(path#String) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:31
    #10 Kernel.require(path#String) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/zeitwerk-2.4.2/lib/zeitwerk/kernel.rb:34
    #11 block in ActiveSupport::Dependencies::Loadable.block in require(file#String) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:332
    #12 ActiveSupport::Dependencies::Loadable.load_dependency(file#String) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:299
    #13 ActiveSupport::Dependencies::Loadable.require(file#String) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:332
    #14 <main> at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/actionpack-6.1.3.1/lib/action_controller/metal.rb:6
    ͱ-- #15 Kernel.require at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:23
    #16 block in Kernel.block in require_with_bootsnap_lfi(path#String, resolved#String) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:23
    #17 Bootsnap::LoadPathCache::LoadedFeaturesIndex.register(short#String, long#String) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/loaded_features_index.rb:92
    #18 Kernel.require_with_bootsnap_lfi(path#String, resolved#String) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:22
    #19 Kernel.require(path#String) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/bootsnap-1.7.3/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb:31
    #20 Kernel.require(path#String) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/zeitwerk-2.4.2/lib/zeitwerk/kernel.rb:34
    #21 block in ActiveSupport::Dependencies::Loadable.block in require(file#String) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:332
    #22 ActiveSupport::Dependencies::Loadable.load_dependency(file#String) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:299
    #23 ActiveSupport::Dependencies::Loadable.require(file#String) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/dependencies.rb:332
    #24 #<Class:Datadog::Contrib::ActionPack::ActionController::Patcher>.patch at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/action_pack/action_controller/patcher.rb:19
    #25 block in Datadog::Contrib::Patcher::CommonMethods.block in patch at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/patcher.rb:27
    #26 block in Datadog::Utils::OnlyOnce.block in run at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/utils/only_once.rb:25
    ͱ-- #27 Thread::Mutex.synchronize at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/utils/only_once.rb:20
    #28 Datadog::Utils::OnlyOnce.run at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/utils/only_once.rb:20
    #29 Datadog::Contrib::Patcher::CommonMethods.patch at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/patcher.rb:25
    #30 #<Class:Datadog::Contrib::ActionPack::Patcher>.patch at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/action_pack/patcher.rb:18
    #31 block in Datadog::Contrib::Patcher::CommonMethods.block in patch at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/patcher.rb:27
    #32 block in Datadog::Utils::OnlyOnce.block in run at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/utils/only_once.rb:25
    ͱ-- #33 Thread::Mutex.synchronize at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/utils/only_once.rb:20
    #34 Datadog::Utils::OnlyOnce.run at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/utils/only_once.rb:20
    #35 Datadog::Contrib::Patcher::CommonMethods.patch at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/patcher.rb:25
    #36 Datadog::Contrib::Patchable::InstanceMethods.patch at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/patchable.rb:54
    #37 block in Datadog::Contrib::Extensions::Configuration.block in configure(target#Datadog::Configuration::Settings, opts#Hash) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/extensions.rb:35
    ͱ-- #38 Hash.each_key at /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/set.rb:344
    #39 Set.each(&block#Proc) at /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/set.rb:344
    #40 Datadog::Contrib::Extensions::Configuration.configure(target#Datadog::Configuration::Settings, opts#Hash) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/extensions.rb:31
    #41 #<Class:Datadog::Contrib::Rails::Framework>.setup at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/rails/framework.rb:35
    #42 #<Class:Datadog::Contrib::Rails::Patcher>.setup_tracer at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/rails/patcher.rb:102
    #43 block in #<Class:Datadog::Contrib::Rails::Patcher>.block in after_intialize(app#PracticalDeveloper::Application) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/rails/patcher.rb:96
    #44 block in Datadog::Utils::OnlyOnce.block in run at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/utils/only_once.rb:25
    ͱ-- #45 Thread::Mutex.synchronize at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/utils/only_once.rb:20
    #46 Datadog::Utils::OnlyOnce.run at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/utils/only_once.rb:20
    #47 #<Class:Datadog::Contrib::Rails::Patcher>.after_intialize(app#PracticalDeveloper::Application) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/rails/patcher.rb:93                                                         
    #48 block in #<Class:Datadog::Contrib::Rails::Patcher>.block in patch_after_intialize at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/ddtrace-0.47.0/lib/ddtrace/contrib/rails/patcher.rb:88
    ͱ-- #49 BasicObject.instance_eval(*args) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/lazy_load_hooks.rb:73
    #50 block in ActiveSupport::LazyLoadHooks.block in execute_hook(name#Symbol, base#PracticalDeveloper::Application, options#Hash, block#Proc) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/lazy_load_hooks.rb:73
    #51 ActiveSupport::LazyLoadHooks.with_execution_control(name#Symbol, block#Proc, once#NilClass) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/lazy_load_hooks.rb:61
    #52 ActiveSupport::LazyLoadHooks.execute_hook(name#Symbol, base#PracticalDeveloper::Application, options#Hash, block#Proc) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/lazy_load_hooks.rb:66
    #53 block in ActiveSupport::LazyLoadHooks.block in run_load_hooks(name#Symbol, base#PracticalDeveloper::Application) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/lazy_load_hooks.rb:52
    ͱ-- #54 Array.each at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/lazy_load_hooks.rb:51
    #55 ActiveSupport::LazyLoadHooks.run_load_hooks(name#Symbol, base#PracticalDeveloper::Application) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/activesupport-6.1.3.1/lib/active_support/lazy_load_hooks.rb:51
    #56 block in block in <module:Finisher> at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/application/finisher.rb:140
    ͱ-- #57 BasicObject.instance_exec(*args) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/initializable.rb:32
    #58 Rails::Initializable::Initializer.run(*args#Array) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/initializable.rb:32
    #59 block in Rails::Initializable.block in run_initializers(group#Symbol, *args#Array) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/initializable.rb:61
    #60 block in #<Class:TSort>.block in tsort_each(each_node#Method, each_child#Method) at /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/tsort.rb:228
    #61 block (2 levels) in #<Class:TSort>.block (2 levels) in each_strongly_connected_component(each_node#Method, each_child#Method) at /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/tsort.rb:350
    #62 #<Class:TSort>.each_strongly_connected_component_from(node#Rails::Initializable::Initializer, each_child#Method, id_map#Hash, stack#Array) at /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/tsort.rb:431
    #63 block in #<Class:TSort>.block in each_strongly_connected_component(each_node#Method, each_child#Method) at /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/tsort.rb:349
    ͱ-- #64 Rails::Initializable::Collection.each at /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/tsort.rb:347
    ͱ-- #65 Method.call(*args) at /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/tsort.rb:347
    #66 #<Class:TSort>.each_strongly_connected_component(each_node#Method, each_child#Method) at /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/tsort.rb:347
    #67 #<Class:TSort>.tsort_each(each_node#Method, each_child#Method) at /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/tsort.rb:226
    #68 TSort.tsort_each(&block#Proc) at /home/djuber/.rbenv/versions/3.0.0/lib/ruby/3.0.0/tsort.rb:205
    #69 Rails::Initializable.run_initializers(group#Symbol, *args#Array) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/initializable.rb:60
    #70 Rails::Application.initialize!(group#Symbol) at /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/railties-6.1.3.1/lib/rails/application.rb:384
    #71 <main> at /home/djuber/src/forem/testcase.rb:10

From: /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/amazing_print-1.3.0/lib/amazing_print/core_ext/class.rb:20 Class#protected_instance_methods:

    18: define_method name do |*args|
    19:   binding.pry
 => 20:   methods = original_method.bind(self).call(*args)
    21:   methods.instance_variable_set(:@__awesome_methods__, self)
    22:   methods.extend(AwesomeMethodArray)
    23:   methods
    24: end

```

Inside `protected_instance_methods`here - and even show-method Hash was sufficient to get the system locked in a loop \(the backtrace points to `public_instance_methods`   but that's not any different.\) Will kill and get back here in a second and be _very_ gentle.

So much for gentle - cd Hash worked - but pry's `ls` command calls `public_instance_methods`   as part of the introspection. Re-attach and find the problem I guess - time to learn  about inspecting stack frames.

```text
    (gdb) backtrace
#0  resolve_refined_method (refinements=8, me=0x55b6264afec0, defined_class_ptr=0x0) at vm_method.c:1231
#1  0x00007f11d355bc6e in method_entry_i (key=140079, value=94240807412640, data=0x7ffc8f658560) at class.c:1365
#2  0x00007f11d3751630 in rb_id_table_foreach (tbl=0x55b622e87cb0, func=func@entry=0x7f11d355bc20 <method_entry_i>, data=data@entry=0x7ffc8f658560) at id_table.c:299
#3  0x00007f11d355c677 in add_instance_method_list (Reading in symbols for proc.c...
me_arg=0x7ffc8f658560, mod=94240738526240) at class.c:1386
#4  class_instance_method_list (argc=<optimized out>, argv=<optimized out>, mod=94240738526240, obj=<optimized out>, func=0x7f11d355bd80 <ins_methods_pub_i>) at class.c:1422
#5  0x00007f11d37a66c6 in vm_call0_cfunc_with_frame (argv=0x7f11d2cd8328, calling=0x7ffc8f6585d0, ec=0x55b621b48810) at vm_eval.c:95
#6  vm_call0_cfunc (argv=0x7f11d2cd8328, calling=0x7ffc8f6585d0, ec=0x55b621b48810) at vm_eval.c:109


(gdb) print me->def->type
$1 = VM_METHOD_TYPE_REFINED
(gdb) print me->def
$2 = (struct rb_method_definition_struct * const) 0x55b625207470
(gdb) print *me->def
$3 = {type = VM_METHOD_TYPE_REFINED, 
      alias_count = 1, 
      complemented_count = 0, 
      body = {
        iseq = {
          iseqptr = 0x0, 
          cref = 0x0}, 
        cfunc = {
          func = 0x0, 
          invoker = 0x0, 
          argc = 0}, 
        attr = {id = 0, location = 0}, 
        alias = {original_me = 0x0}, 
        refined = {orig_me = 0x0, owner = 0}, 
        bmethod = {proc = 0, hooks = 0x0, defined_ractor = 0}, 
        optimize_type = OPTIMIZED_METHOD_TYPE_SEND}, 
        original_id = 140079, 
        method_serial = 34747
      }
(gdb) print me
$4 = (const rb_method_entry_t *) 0x55b6264afec0     

(gdb) print *me
$5 = {
      flags = 90234, 
      defined_class = 94240738526240, 
      def = 0x55b625207470, 
      called_id = 140079, 
      owner = 94240738526240
      }
```

Here `me` is a method entry or `rb_method_entry_t`

{% code title="include/ruby/method.h" %}
```c

/* method data type */

typedef struct rb_method_entry_struct {
    VALUE flags;
    VALUE defined_class;
    struct rb_method_definition_struct * const def;
    ID called_id;
    VALUE owner;
} rb_method_entry_t;
```
{% endcode %}

One hint if you're trying this again - pointers are hex values and VALUE's \(which I assume are lookup table keys?\) are integer types.

{% code title="include/ruby/internal/value.h" %}
```c


#if defined HAVE_UINTPTR_T && 0
typedef uintptr_t VALUE;
typedef uintptr_t ID;
# define SIGNED_VALUE intptr_t
# define SIZEOF_VALUE SIZEOF_UINTPTR_T
# undef PRI_VALUE_PREFIX
# define RBIMPL_VALUE_NULL UINTPTR_C(0)
# define RBIMPL_VALUE_ONE  UINTPTR_C(1)
# define RBIMPL_VALUE_FULL UINTPTR_MAX

```
{% endcode %}

```text
(gdb) p me->def->body
$17 = {iseq = {iseqptr = 0x0, cref = 0x0}, cfunc = {func = 0x0, invoker = 0x0, argc = 0}, attr = {id = 0, location = 0}, alias = {original_me = 0x0}, refined = {orig_me = 0x0, owner = 0}, bmethod = {proc = 0, hooks = 0x0, defined_ractor = 0}, optimize_type = OPTIMIZED_METHOD_TYPE_SEND}
(gdb) p me->def->body->refined
$18 = {orig_me = 0x0, owner = 0}

(gdb) print rb_class_superclass(me->owner)
$19 = 94240738666680
(gdb) print me
$20 = (const rb_method_entry_t *) 0x55b6264afec0
(gdb) print *me
$21 = {flags = 90234, defined_class = 94240738526240, def = 0x55b625207470, called_id = 140079, owner = 94240738526240}
```

This is looking at the class's methods \(we were passed in a method entry for a defined class value pointer\) super class, if there is no super we return 0 - if there is we call `search_method_protect`

