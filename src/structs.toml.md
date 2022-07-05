title = 'How the struct gets made'
subtitle = 'In which peek behind the curtain to see how compilers represent our data types.'
author = 'Tom Panton'
tags = []
published = '2022-07-05T20:16:17.285223829Z'
---
A while back I came across a question online asking why Rust uses a different layout for structs
than C. "Layout" here refers to the way a struct gets represented as a sequence of bytes in memory.
I think it's an excellent question, and it gives us an excuse to mess around with a debugger to see
what's going on in memory, so let's have a go at answering it!

To know _why_ the two languages lay out structs differently, we first need to know _how_ they lay
them out. Let's define a little test struct in C for us to poke and prod at:

```C
#include <stdint.h>

struct TestStruct {
    uint32_t x; // 32 bits = 4 bytes
    uint64_t y; // 64 bits = 8 bytes
    uint32_t z; // 32 bits = 4 bytes
};
```

At a glance, representing `TestStruct` in memory doesn't seem like a particularly difficult thing
to do. My first guess is that it can be 16 contiguous bytes where the first 4 bytes represent `x`,
the next 8 represent `y` and the last 4 represent `z`. Something like this:

![A row of squares representing TestStruct, with each square representing one byte. The first four squares are labelled x, the next eight are labelled y and the last four are labelled z.](/article_media/struct_diagram_1.png)

Let's do a quick experiment to see if I'm right! We'll use C's `sizeof` operator find the size of
`TestStruct` in bytes. If my prediction is correct, it should be 16 bytes.

```C
@@struct_size.c@@
#include <stdint.h>
#include <stdio.h>

struct TestStruct {
    uint32_t x;
    uint64_t y;
    uint32_t z;
};

int main() {
    printf("%zu bytes\n", sizeof (struct TestStruct));
    return 0;
}
```

Let's run it and see what we get:

```
$ clang struct_size.c
$ ./a.out
24 bytes
```

24 bytes?! I was wrong! So what the heck is C doing here?

Well, let's take a look using a debugger! We'll create a `TestStruct` variable and fill its fields
with some values that will be easy to spot later:

```C
@@struct_layout.c@@
#include <stdint.h>

struct TestStruct {
    uint32_t x;
    uint64_t y;
    uint32_t z;
};

int main() {
    struct TestStruct test;
    test.x = 0xcafebabe;
    test.y = 0x0123456789abcdef;
    test.z = 0xfeedface;
    return 0;
}
```

Now let's compile it and load it into LLDB:

```
$ clang -g struct_layout.c
$ lldb a.out
Current executable set to '/home/tom/structs/a.out' (x86_64).
```

Let's put a breakpoint to pause the program right before `main` returns on line 14, and then we can
read the 24 bytes of memory representing our `test` variable. To make it a little easier to read,
we'll ask LLDB to organise the bytes into groups of four.

```
(lldb) breakpoint set --file struct_layout.c --line 14
(lldb) run
(lldb) memory read --format x --size 4 --count `24 / 4` `&test`
0x7fffffffea10: 0xcafebabe 0x00000000 0x89abcdef 0x01234567
0x7fffffffea20: 0xfeedface 0x00000000
```

We can see `0xcafebabe` which is the value we stored in `test.x`, `0x0123456789abcdef` which we
stored in `test.y` (the two groups of four bytes are displayed in reverse because my machine is
[little-endian](https://en.wikipedia.org/wiki/Endianness)) and `0xfeedface` which we stored in
`test.z`. However, there are also some bytes that we didn't tell C to store: there's four bytes
of zeroes sitting between `test.x` and `test.y`, and another four bytes of zeroes after `test.z`!
Although we don't know what these extra 8 bytes are doing there yet, at least we now have a more
accurate idea of how `TestStruct` looks in memory:

![An updated version of the previous diagram. Four additional squares labelled with question marks have been added between x and y, and another four squares labelled by question marks have been added to the end.](/article_media/struct_diagram_2.png)

To understand what those extra bytes are there for, we first need to know about _alignment_.

## So, what's this "alignment" stuff?

Every type has both a size and an alignment. Whereas the size determines how much memory is
required to represent the type, the alignment determines where that memory is allowed to be.
The rule for alignment is simple:

> A value should be stored at a memory address that is a multiple of its alignment.

Most modern CPUs expect this rule to be followed; if it's broken, a variety of platform-dependent
Bad Things can happen such as performance penalties, [crashes](https://www.oracle.com/technetwork/server-storage/sun-sparc-enterprise/documentation/140521-ua2011-d096-p-ext-2306580.pdf#page=93), and [changes to the atomicity guarantees of instructions](https://www.amd.com/system/files/TechDocs/24593.pdf#page=252).

Let's look at some examples. Similar to the `sizeof` operator, we can use the `alignof` operator
to find the alignment of a particular type:

```C
@@alignment.c@@
#include <stdalign.h>
#include <stdint.h>
#include <stdio.h>

int main() {
    printf("uint8_t:  %zu\n", alignof (uint8_t));
    printf("uint32_t: %zu\n", alignof (uint32_t));
    printf("uint64_t: %zu\n", alignof (uint64_t));
    return 0;
}
```

```
$ clang alignment.c
$ ./a.out
uint8_t:  1
uint32_t: 4
uint64_t: 8
```

C tells us that `uint8_t` has an alignment of 1. Every memory address is a multiple of 1, so that
means it's ok for a `uint8_t` to live at any memory address. `uint32_t`, on the other hand, has
an alignment of 4, so it can only live at memory addresses 0, 4, 8, 12, 16 and so on.

Now we can explain what the mysterious extra bytes in `TestStruct` are there for! The 4 bytes
between `x` and `y` are _padding_ to ensure that `y` follows the rule of alignment. `y` is a
`uint64_t` which has an alignment of 8 (on 64-bit platforms); without the padding it would be at
offset 4, which is not a multiple of 8, but when we add the 4 bytes of padding it ends up at offset
8 instead, which is of course a multiple of 8.

![A diagram showing the struct with and without the padding bytes added between x and y. Without the padding, x is immediately followed by y; since x has size 4, y is at offset 4 which is not a multiple of 8. With the 4 bytes of padding between x and y, y is at offset 8 which is a multiple of 8.](/article_media/struct_diagram_3.png)

The 4 bytes of padding after `z` are there to make sure the rule of alignment is followed when we
have an _array_ of `TestStruct`. Suppose we have an array `struct TestStruct a[2]`; arrays are
represented by just storing the elements contiguously in memory, so without the
padding after `z`, `a[1].y` would be at offset 20 + 8 = 28 from the start of the array, which is
not a multiple of 8 so it would break the rule of alignment. With the padding after `z` included,
`a[1].y` is at offset 24 + 8 = 32 from the start of the array, which is a multiple of 8.


![A diagram showing an array of two TestStructs with and without the padding after z. Both arrays include the padding between x and y, however. Without the 4 bytes of padding, the y field of the second element ends up at offset 28, which is not a multiple of 8. With the padding, it ends up at offset 32, which is a multiple of 8.](/article_media/struct_diagram_4.png)

Ok, so now we have a sense of how C lays out structs; the fields are put in memory in the same
order as we wrote them in the struct definition, and extra padding is inserted after some of the
fields when it is needed to follow the rule of alignment.

## Turning our attention to Rust

Time to find out what Rust does differently to C! Let's start off by defining a Rust equivalent
of `TestStruct` and checking its size:

```Rust
@@struct_size.rs@@
#![allow(dead_code)]

use std::mem::size_of;

struct TestStruct {
    x: u32,
    y: u64,
    z: u32,
}

fn main() {
    println!("{} bytes", size_of::<TestStruct>());
}
```

```
$ rustc struct_size.rs
$ ./struct_size
16 bytes
```

16 bytes is smaller than the 24 bytes used by C, so Rust can't be laying out `TestStruct` the same
way. To find out what it's doing, let's use the same trick from before of filling the fields of
the struct with some dummy values then reading the memory using LLDB:

```Rust
@@struct_layout.rs@@
#![feature(bench_black_box)]
#![allow(dead_code)]

use std::hint::black_box;

struct TestStruct {
    x: u32,
    y: u64,
    z: u32,
}

fn main() {
    let test = TestStruct {
        x: 0xcafebabe,
        y: 0x0123456789abcdef,
        z: 0xfeedface,
    };

    // Our test value is not actually used for anything in the program, so the
    // Rust compiler wants to optimise it out. We encourage it not to do this
    // by using the black box function.
    black_box(test);
}
```

```
$ rustc -g struct_layout.rs
$ lldb struct_layout
(lldb) breakpoint set --file struct_layout.rs --line 22
(lldb) run
(lldb) memory read --format x --size 4 --count `16 / 4` `&test`
0x7fffffffe3d8: 0x89abcdef 0x01234567 0xcafebabe 0xfeedface
```

Two things jump out: there's no padding bytes, and the value we stored in `y` appears before the
value we stored in `x`. It looks like Rust has **changed the order of the fields**! This is a cool
little optimisation; by switching the order of `x` and `y` in memory, all of the fields obey the 
rule of alignment without the need for any padding. `y` is now at offset 0 which is a multiple of
8, `x` is at offset 8 which is a multiple of 4, and `z` is at offset 12 which is a multiple of 4.

![A diagram comparing the C-style struct layout to the Rust-style one. In the C-style layout, the fields x, y and z appear in order, with 4 bytes of padding between x and y and another 4 bytes of padding after z. In the Rust-style layout, y appears first, then x and lastly z, with no padding anywhere in the struct. The Rust-style is two-thirds the size of the C-style layout.](/article_media/struct_diagram_5.png)

Getting rid of the padding can have some practical performance benefits; since the overall size
of the struct is smaller, we can fit more in the CPU's limited cache memory, which is _much_
faster to access than RAM.

## "Let me choose the order, dammit!"

Rust's way of doing things might improve performance, but, (angrily shaking fist), what right does
the compiler have to mess with the order of our fields without our permission?! We specifically
said that `x` comes before `y` when we defined `TestStruct`; wouldn't it be better for Rust to
just tell us that it's a suboptimal ordering rather than silently moving the fields around? Then,
we could decide whether or not we want to listen to the compiler's recommendation and manually
change the order of the fields, which would give us more control.

Unfortunately, this manual approach has problems; in particular, it doesn't play nice with
generics. Suppose we have a generic struct like this:

```Rust
struct GenericStruct<T, U> {
    x: T,
    y: U,
    z: u32,
}
```

There's no single ordering of the struct's fields that's optimal (in terms of the amount of
padding required) for all possible choices of `T` and `U`. For example, the only two orderings
that are optimal for both `GenericStruct<u32, u64>` and `GenericStruct<u64, u32>` are `x, z, y`
and `y, z, x`, but neither of these two orderings are optimal for `GenericStruct<u16, u16>`.
Whatever ordering we pick, there's going to be some choice of `T` and `U` that uses more padding
than the minimum possible amount.

That's why it's useful for Rust to pick the order of the fields for us; Rust can use different
orderings depending on the values of the generic parameters so that padding is always minimised.
For `GenericStruct<u32, u64>` it can use the ordering `x, z, y`, and for
`GenericStruct<u16, u16>` it can use a different ordering `x, y, z`.

Despite this, there's still going to be situations where we _need_ to manually specify the order
of the fields in memory, so Rust provides us with the `#[repr(C)]` attribute which lets us use
C's memory layout for a particular struct.

## Back to the original question

Time to answer the question we started with: _why_ do the two languages use different layouts?
Since C is often used for very low-level tasks like FFI and interfacing with hardware, it's
important that data has a consistent and predictable layout in memory; therefore the programmer
is given complete control over the ordering of fields. If you were writing an IP implementation
in C by
[casting the received bytes](https://github.com/torvalds/linux/blob/c1084b6c5620a743f86947caca66d90f24060f56/include/linux/ip.h#L21)
to a
[struct representing the header format](https://github.com/torvalds/linux/blob/c1084b6c5620a743f86947caca66d90f24060f56/include/uapi/linux/ip.h#L86),
and the compiler decided to rearrange the order of that struct's fields, then your program would
misinterpret the IP headers!

Since casting between bytes and structs
[can't be done in safe Rust](https://doc.rust-lang.org/nomicon/transmutes.html), it's more
acceptable for Rust to take a bit of control away from the programmer and reorder the fields.
This means that structs will always have the optimal size without the need for the programmer to
think about alignment, even for generic structs that are impossible to optimise by hand.
