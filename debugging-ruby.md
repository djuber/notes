---
description: >-
  hints about how to use gdb on a running ruby process, and other tools you
  might prefer
---

# Debugging Ruby

Read the .gdbinit file shipped in the ruby sources. I made the mistake of searching for guides on getting ruby internals from gdb in blog posts \(which might have led to some english language bubble filtering out the obvious canonical source for this - the ruby development team is targetting the gnu toolchain and uses gdb for debugging, their tools are a good first step\).

One of these definitions "rb\_p" \(was rp in very early versions of ruby\) is [defined](https://github.com/ruby/ruby/blob/master/.gdbinit#L858-L860) a little quirky

```text
define rb_p
  call rb_p($arg0)
end
```

This just associates the `rb_p` macro with the same named c function \(in[ io.c](https://github.com/ruby/ruby/blob/master/io.c#L8050-L8071)\).



