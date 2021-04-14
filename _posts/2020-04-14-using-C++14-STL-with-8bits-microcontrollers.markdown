---
layout: post
title:  "Using C++14 STL with 8-bits microcontrollers"
date:   2021-04-14 11:20:47 -0300
categories: gcc avr arduino embedded microcontroller
image: "/assets/img/2021-04-14-Use-temporary-int-objects-to-access-struct-tm-members.png"
share-img: "/assets/img/2021-04-14-Use-temporary-int-objects-to-access-struct-tm-members.png"
author: Felipe Magno de Almeida
---

GCC has support for using its Standard Template Library in AVR 8-bit
microcontrollers since 2016, when we at Expertise Solutions pushed
three commits upstream to GCC project.

# Motivation

You may ask, why use the STL? And if so, why not use another STL?

There are some tutorials and materials on the internet, including a
ArduinoSTL project to implement partial STL support for AVR. However,
they lack anything new from C++14 and up, like lots of type traits and
other meta-programming goodies that allows zero-overhead over very
advanced template-y code.

Other reasons include being able to use dependencies that do require
STL. We were able to use Boost.Spirit library on the AVR for creating
advanced parsers using just a few hundred bytes of footprint size by
using the libstdc++v3.

# Why libstdc++v3 didn't work before?

There were 3 incompatibility problems that caused compilations errors.

## Non-standard `struct tm`

First, there were codes in locale C++ standard library implementation
that used the tm fields directly in calls that would expect a specific
type as is defined in POSIX's `struct tm`. However, AVR libc uses a
non-standard `struct tm` implementation uses `int8_t` and `int16_t`
where `int` is expected.

The patch
[26b67e383f4](https://github.com/gcc-mirror/gcc/commit/26b67e383f4b1df812cd7ba33de43451aff883ba)
fixes this by using a temporary `int` object.

## Pointers with 16 bits size

In libstdc++v3, exception implementation uses a Copy-On-Write
implementation to avoid unnecessary reallocations, which could cause
failure if the allocation fails. By doing that, it must abstract
pointers in different integral types, which it was implemented only
for 32-bit and 64-bit architectures.

The patch
[f68963c0923](https://github.com/gcc-mirror/gcc/commit/26b67e383f4b1df812cd7ba33de43451aff883ba))
implements the same logic for 16-bit pointers.

## Add AVR targets to libstdc++v3 build

The last patch just adds to libstdc++v3's autotools the AVR target so
it can be compiled for the AVR platform.

# Conclusion

We've been using GCC with STL for AVR microcontrollers for years
now. Nobody seems to be aware that this is possible yet. Next post
I'll explain how to compile GCC with libstdc++v3 for AVR!

Let me know if you want to try in other 8-bit microcontrollers.
