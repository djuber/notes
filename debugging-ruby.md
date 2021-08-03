---
description: >-
  hints about how to use gdb on a running ruby process, and other tools you
  might prefer
---

# Debugging Ruby

Read the .gdbinit file shipped in the ruby sources. I made the mistake of searching for guides on getting ruby internals from gdb in blog posts \(which might have led to some english language bubble filtering out the obvious canonical source for this - the ruby development team is targetting the gnu toolchain and uses gdb for debugging, their tools are a good first step\).

One of these definitions "rb\_p" is [defined](https://github.com/ruby/ruby/blob/master/.gdbinit#L858-L860) a little quirky

```text
define rb_p
  call rb_p($arg0)
end
```

This just associates the `rb_p` macro with the same named c function \(in[ io.c](https://github.com/ruby/ruby/blob/master/io.c#L8050-L8071) - or the ruby 2 pre-ractor [version](https://github.com/ruby/ruby/blob/5445e0435260b449decf2ac16f9d09bae3cafe72/io.c#L7797-L7810)\).

There's a different implementation called "rp" [https://github.com/ruby/ruby/blob/master/.gdbinit\#L25-L311](https://github.com/ruby/ruby/blob/master/.gdbinit#L25-L311) that basically is a giant conditional to step through a pointer from each of the internally defined types to get the values out. 

A lot of the ruby id's are C pointers, _but_ they're masked/converted - so some of the functions in the gdbinit work to move long object id numbers to pointers \(since that's what C tools like gdb will need to traverse objects\). The further in to the VM you move the more likely the items passed between functions will be pointers \(real pointers to "real" memory on the machine\) and not object id \(VALUE types\).

I think `rb_obj_id` is the C function to convert these \([https://github.com/ruby/ruby/blob/master/gc.c\#L4493-L4526](https://github.com/ruby/ruby/blob/master/gc.c#L4493-L4526)\)

```c

/*
 *  Document-method: __id__
 *  Document-method: object_id
 *
 *  call-seq:
 *     obj.__id__       -> integer
 *     obj.object_id    -> integer
 *
 *  Returns an integer identifier for +obj+.
 *
 *  The same number will be returned on all calls to +object_id+ for a given
 *  object, and no two active objects will share an id.
 *
 *  Note: that some objects of builtin classes are reused for optimization.
 *  This is the case for immediate values and frozen string literals.
 *
 *  BasicObject implements +__id__+, Kernel implements +object_id+.
 *
 *  Immediate values are not passed by reference but are passed by value:
 *  +nil+, +true+, +false+, Fixnums, Symbols, and some Floats.
 *
 *      Object.new.object_id  == Object.new.object_id  # => false
 *      (21 * 2).object_id    == (21 * 2).object_id    # => true
 *      "hello".object_id     == "hello".object_id     # => false
 *      "hi".freeze.object_id == "hi".freeze.object_id # => true
 */

VALUE
rb_obj_id(VALUE obj)
{
    /*
     *                32-bit VALUE space
     *          MSB ------------------------ LSB
     *  false   00000000000000000000000000000000
     *  true    00000000000000000000000000000010
     *  nil     00000000000000000000000000000100
     *  undef   00000000000000000000000000000110
     *  symbol  ssssssssssssssssssssssss00001110
     *  object  oooooooooooooooooooooooooooooo00        = 0 (mod sizeof(RVALUE))
     *  fixnum  fffffffffffffffffffffffffffffff1
     *
     *                    object_id space
     *                                       LSB
     *  false   00000000000000000000000000000000
     *  true    00000000000000000000000000000010
     *  nil     00000000000000000000000000000100
     *  undef   00000000000000000000000000000110
     *  symbol   000SSSSSSSSSSSSSSSSSSSSSSSSSSS0        S...S % A = 4 (S...S = s...s * A + 4)
     *  object   oooooooooooooooooooooooooooooo0        o...o % A = 0
     *  fixnum  fffffffffffffffffffffffffffffff1        bignum if required
     *
     *  where A = sizeof(RVALUE)/4
     *
     *  sizeof(RVALUE) is
     *  20 if 32-bit, double is 4-byte aligned
     *  24 if 32-bit, double is 8-byte aligned
     *  40 if 64-bit
     */

    return rb_find_object_id(obj, cached_object_id);
}

static VALUE
rb_find_object_id(VALUE obj, VALUE (*get_heap_object_id)(VALUE))
{
    if (STATIC_SYM_P(obj)) {
        return (SYM2ID(obj) * sizeof(RVALUE) + (4 << 2)) | FIXNUM_FLAG;
    }
    else if (FLONUM_P(obj)) {
#if SIZEOF_LONG == SIZEOF_VOIDP
        return LONG2NUM((SIGNED_VALUE)obj);
#else
        return LL2NUM((SIGNED_VALUE)obj);
#endif
    }
    else if (SPECIAL_CONST_P(obj)) {
        return LONG2NUM((SIGNED_VALUE)obj);
    }

    return get_heap_object_id(obj);
}


static VALUE
cached_object_id(VALUE obj)
{
    VALUE id;
    rb_objspace_t *objspace = &rb_objspace;

    if (st_lookup(objspace->obj_to_id_tbl, (st_data_t)obj, &id)) {
        GC_ASSERT(FL_TEST(obj, FL_SEEN_OBJ_ID));
        return id;
    }
    else {
        GC_ASSERT(!FL_TEST(obj, FL_SEEN_OBJ_ID));

        id = objspace->next_object_id;
        objspace->next_object_id = rb_int_plus(id, INT2FIX(OBJ_ID_INCREMENT));

        st_insert(objspace->obj_to_id_tbl, (st_data_t)obj, (st_data_t)id);
        st_insert(objspace->id_to_obj_tbl, (st_data_t)id, (st_data_t)obj);
        FL_SET(obj, FL_SEEN_OBJ_ID);

        return id;
    }
}

```

Most of that is just documenting how object id's are built - there's also a reverse function "id2ref" that takes an object id and responds with a pointer \(from the object table? I'm mildly confused by most of this\).

