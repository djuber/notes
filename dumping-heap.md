---
description: >-
  if you want to get a snapshot of the memory for a ruby process, you could do
  worse
---

# Dumping heap

{% embed url="https://github.com/djudd/reap" %}



 you can connect to the Ruby process with `gdb`, then run:

```text
call rb_eval_string_protect("Thread.new{require 'objspace';f=open('/tmp/heap.json','w');ObjectSpace.dump_all(output: f, full: true);f.close}", 0)
```

So I tried that in sidekiq and it segfaulted... That's too bad.

```text
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
0x00007fdac0c0e8b3 in __GI___select (nfds=14, readfds=0x7fdaaef4ef80, writefds=0x0, 
    exceptfds=0x0, timeout=0x0) at ../sysdeps/unix/sysv/linux/select.c:41
41      ../sysdeps/unix/sysv/linux/select.c: No such file or directory.
(gdb) call rb_eval_string_protect("Thread.new{require 'objspace';f=open('/tmp/heap.json','w');ObjectSpace.dump_all(output: f, full: true);f.close}", 0)

Thread 1 "bundle" received signal SIGSEGV, Segmentation fault.
0x00007fdac0d69a64 in rb_array_const_ptr_transient (a=13926668)
    at ./include/ruby/ruby.h:2185
2185    ./include/ruby/ruby.h: No such file or directory.
The program being debugged was signaled while in a function called from GDB.
GDB remains in the frame where the signal was received.
To change this behavior use "set unwindonsignal on".
Evaluation of the expression containing the function
(rb_eval_string_protect) will be abandoned.
When the function is done executing, GDB will silently stop.
```

This is probably something to do with attaching gdb to a sidekiq parent process, so instead I made a worker to run that code inside a sidekiq thread.

```ruby
require 'objspace'

class HeapDumpWorker
  include Sidekiq::Worker

  sidekiq_options queue: :medium_priority

  def perform(filename)
    File.open(filename, 'w') do |f|
      ObjectSpace.dump_all(output: f, full: true)
    end
    puts "wrote it!"
  end
end

```

In a rails console I can dump the heap to an arbitrary file by passing the path as the sole job arg.



The heap dump format is what looks like json-lines formatted objects \(one object per line\).

There are a few `"type": "ROOT"` top level object arrays \(vm, machine context, end proc, global table, global list, finalizers, ...\) followed by every object \(these objects begin with "address" and not "type"\)



Moved this discussion \(after a successful use of a sidekiq job to dump memory\) to [https://forem.team/danuber/tracking-down-a-memory-hog-23m5](https://forem.team/danuber/tracking-down-a-memory-hog-23m5) - the output from reap was super handy in identifying a misconfiguration.

