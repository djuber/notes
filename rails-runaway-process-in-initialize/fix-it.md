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

