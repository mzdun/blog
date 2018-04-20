---
layout: post
title: Inheritance in C++
category: C++ Fundamentals
tags: ["c++", "fundamentals", "black magic"]
---

This article explains the issues of object layout in C++ and consequence of casting between types.<!-- more --> I wanted to write text explaining thunks since late naughts, maybe 2010, but only after I was severely bitten by the `reinterpret_cast` in a large project I was part of in 2013, I knew, what this text will be about: `reinterpret_cast` and how it gets more dangerous with every layer of inheritance in C++. This is version from April 2016, which I wrote for some of my then-colleagues, after the _"Object lifetime and virtual calls"_.

## No inheritance

The layout tells the compiler, where things are in respect of the beginning of an object. The order of members is the same as the order of their declaration, but the offsets might (will) be adjusted for the alignments required by the targeted architecture. This comes from C and applies to any class simple enough to resemble C structures, no matter if the class started with `class` or `struct` keyword.

Then comes in inheritance: when the members of descendant are laid out, the layout of that ancestor must be taken into account.

## Single inheritance

How objects with inheritance are laid out? This is one of the questions that C++ refuses to answer. The compiler is absolutely free to come up with it's own scheme and simply stick to it. However only very exotic ones go beyond simplest solution &ndash; start the layout of the descendant with the layout of ancestor:

```
                    +- A ---------+
+- B -----+         | +- B -----+ |
| member1 |         | | member1 | |
| member2 |   ==>   | | member2 | |
| member3 |         | | member3 | |
+---------+         | +---------+ |
                    | memberA     |
                    | memberB     |
                    | memberC     |
                    +-------------+
```

This means converting between pointer to `A` and pointer to `B` is a NOOP from runtime point of view &ndash; a pointer to `A` is always a pointer to `B`. Both`static_cast` and `reinterpret_cast` vanish at the compilation.

## Multiple inheritance

Things get slightly more complicated with multiple inheritance. Let's say a class `A` this time inherits not only from `B`, but also from `C`. This might result in layout, where members of `C` go right after members of `B`:

```
                    +- A ---------+
+- B -----+         | +- B -----+ |
| member1 |         | | member1 | |
| member2 |   ==>   | | member2 | |
| member3 |         | | member3 | |
+---------+         | +---------+ |
+- C -----+         | +- C -----+ |
| memberX |         | | memberX | |
| memberY |   ==>   | | memberY | |
| memberZ |         | | memberZ | |
+---------+         | +---------+ |
                    | memberA     |
                    | memberB     |
                    | memberC     |
                    +-------------+
```

This has an impact on generated code. First &ndash; it makes `static_cast` to behave differently: a cast from pointer to `A` to pointer to `C` must add size of `B` to the value; the reverse operation requires subtracting this size from the pointer to `C`.

Second &ndash; `reinterpret_cast` starts being simply dangerous. First member of both `A` and `B` is `member1`. If you have a pointer to `A` and want to access`member1`, compiler just dereferences `this`. At the same time, first member of `C` is `memberX`. If you use `reinterpret_cast<A*>(pointer_to_c)`, there is no adjustment done. Suddenly, what used to be `memberX` will be known as `member1` and as a result you'll start getting errors from acting on object whose layout is insane &ndash; errors, which will probably occur miles from the casting itself.

> If you want to convert between to types with inheritance (classes, pointer to classes or references to classes), always use `static_cast` and never`reinterpret_cast`.
> 
> If one of the types is a simple type (e.g. `void*`), you can use `reinterpret_cast`, but be careful with casting it back.

## Virtual inheritance

Since each base class layout becomes part of it's descendant's layout, if you have multiple inheritance you'll sometimes get multiple copies of the same memory layout, where each copy has it's state maintained separately. While sometimes this is needed, it may have unwanted side effect of adding memory.

For that C++ has a mechanism of virtual inheritance. Instead of placing entire virtual ancestor, the layout of the descendant contains only a pointer to another virtual table: virtual type table, `vttbl` (notice additional `t` there). This table consists of offsets to the beginning of each virtual ancestor. Of course the compiler is free to come up with the way it implements `vttbl`. For instance, MSVC reuses `vtbl` for that, making a strange amalgam of virtual offsets and method pointers.

```
                                    +- A ---------+
+- D -----+     +- B -----+         | +- B -----+ |
| memberI |     | vttblB  |         | | vttblB  | |
| memberJ |     |---------|         | |---------| |
| memberK |     | member1 |         | | member1 | |
+---------+     | member2 |   ==>   | | member2 | |
                | member3 |         | | member3 | |
                +---------+         | +---------+ |
                                    | +- C -----+ |
                                    | | vttblC  | |
                +- C -----+         | |---------| |
                | vttblC  |         | | memberX | |
                |---------|   ==>   | | memberY | |
                | memberX |         | | memberZ | |
                | memberY |         | +---------+ |
                | memberZ |         | memberA     |
                +---------+         | memberB     |
                                    | memberC     |
                                    +-------------+
```

Notice none of the classes (apart from `D`) has layout for class `D`. For that to happen, you need to try and create the objects themselves (offsets assume each member takes 4 bytes):

```
                          A a: +---------+ classes A and B
D d: +---------+               | vttbl_1 | [D: 44]---+
     | memberI | class D       | member1 |           |
     | memberJ |               | member2 |           |
     | memberK |               | member3 |           |
     +---------+               | vttbl_2 | [D: 28]   | class C
                               | memberX |     |     |
B b: +-------- + class B       | memberY |     |     |
     | vttbl   | [D: 16]       | memberZ |     |     |
     | member1 |     |         | memberA |     |     |
     | member2 |     |         | memberB |     |     |
     | member3 |     |         | memberC |     |     |
     | memberI | <---+         | memberI | <---+-----+ class D
     | memberJ |               | memberJ | 
     | memberK |               | memberK | 
     +---------+               +---------+
```

While `reinterpret_cast` is as unusable in this context as it is with "normal" multiple inheritance, `static_cast` starts to be even stranger than before. Notice, how `vttbl` for class `B` in object `b` has a different offset to `D` than in object `a`?

## Casting from descendant to ancestor

So, `static_cast` between `A` and `B` is still a NOOP, between `A` and `C` still consists of adding or subtracting `sizeof(B)` to or from input pointer, but how do you convert between any of `A`, `B` and `C` &ndash; and `D`? How do you make the reverse cast?

Casting from pointer to `B` to pointer to `D`, in it's essence, is still a pointer adjustment. Only the amount is not a compile-time constant, but run-time query: the generated code must consult the `vttbl` attached to the object for the position of `D`, which requires a dereference from `this` to `vttbl` and getting n_th_entry, just like with locating the virtual method. After that we have an offset to add to the pointer to cast it to `D`.

## Casting from ancestor to descendant

But how do you do `static_cast<A*>(pointer_to_D)`? Do you subtract 44 or 16? The answer is &ndash; you can't know, there is not enough information given, so both `static_cast` and C-style cast from virtual ancestor to descendant is not int the language.

One way to do that is RTTI. Run-time Type Info, when turned on, attaches a small chunk of data to every object created in C++ program. It contains information on what exact type this object is and how it interacts with other types. This is used by `dynamic_cast`, a run time casting operator, to properly locate reverse offsets. Those chunk of data, however, put a tax on the program most programmers and product managers are unwilling to pay.

What are other options? Well, virtual methods. If you have a virtual method in `D`, then it's code doesn't care for `A`s, `B`s or `C`s of it's world, so no issue there. If you write your version for any of these descending classes, you write this as if `this` was of type `A*` or `B*`, just like with any other overriding code.

## VTBL thunks

But the `vtbl` is not produced by the compiler until it sees any object created. At this point `vttbl` is created and all needed offsets are in place. Such `vtbl`contains pointers to all the overridden methods, but not direct ones.

Since the code for the method is already created and that code make some assumptions about `this` pointer, the offset value between this descendant and "movable" class is taken by the compiler from newly created `vttbl` and a thunk is created. This thunk's responsibility is to adjust pointers between virtual ancestor and correct descendant.

For instance, MSVC (which uses `ECX` to move `this`), when resolving a virtual method for `D::g`, as overridden by `B::g`, placed address 0x012D9D7C in one of the `vtbl`s, while true method lived under 0x012D2807. The latter method was, however, prepared with assumption where `this` should point to. It happens, that for this particular pair of types, the place `this` should point to is 28 (or 0x1C) bytes before beginning of virtual ancestor. So the address 0x012D9D7C became a house for this jump thunk:

```
B::g:
012D9D7C  sub         ecx,1Ch
012D9D7F  jmp         B::g (012D2807h)
```

Those thunks are so common, the Microsoft's debugger has a support to tell you the type (or: purpose) of such methods. For instance, the debugger calls method under  0x012D2807 ``[thunk]:B::g`adjustor{28}'``, as in "`B::g`, which is really an _adjustor thunk_ subtracting (or adding) 28 bytes". Why are they common, if the virtual inheritance is not?

The reason is, you need the thunk not only in virtual inheritance, you'll need it in any multiple inheritance. Each time a method is overridden for a class, whose sub-layout does not start at the beginning of current layout, an adjustor thunk _will_ be created to smooth things over, to help you not think too much how crazy multiple inheritance really is.

So, if you would want to take one thing from all of this, I'd suggest this:

> Never, ever use `reinterpret_cast`, unless absolutely necessary. Because when you do, well, [nasal demons](http://www.catb.org/jargon/html/N/nasal-demons.html).
