---
layout: post
title: Not so obvious in fact
---

*Originally published on [Substack](https://zatrazz.substack.com/p/not-so-obvious-in-fact).*

Although the Twitter post is clear bait for discussion, the topic is interesting for discussion:

[Find the log base 2 of an N-bit integer in O(lg(N)) operations with multiply and lookup](https://x.com/ChShersh/status/1981390801553809426)

![A msb.cpp code snippet: a constexpr positions[] lookup table, a sequence of shift-or operations on v, and a final return positions[(v * 0x8CACDU) >> 27]]({{ site.baseurl }}/images/msb-debruijn.png)

This code snippet is a direct copy from [Bit Twiddling Hacks](https://graphics.stanford.edu/~seander/bithacks.html#IntegerLogObvious), which has very amusing low-level code snippets and some history on optimizing them (The [Hacker's Delight](https://en.wikipedia.org/wiki/Hacker%27s_Delight) is another source of such low-level tricks).

And this is provided as a [compiler builtin](https://gcc.gnu.org/onlinedocs/gcc/Bit-Operation-Builtins.html) for a long time by both gcc and clang as *__builtin_ctz*. The built-in has some inherent issues:

1. It maps to default C types, so using it with [fixed-width integer types](https://en.cppreference.com/w/cpp/types/integer.html) requires extra care from the user.

2. If the input is 0, the result is **undefined.** This is because some architectures return different values for *0*, and to avoid adding the burden of extra checks on a specific architecture, the simpler solution is to make undefined behavior.

The builtin was introduced in GCC version 3.0 (2001), and it took another 22 years for the C standard to address the two issues above with [stdbit.h](https://en.cppreference.com/w/c/header/stdbit). With a C23 runtime, one can just use *stdc_trailing_zeros* regardless of the input type and value.

C++ also provides similar [bit manipulation support](https://cppreference.net/cpp/utility/bit.html), and one can just use *std::countr_zero*. Also, C++26 will provide the C23 stdbit.h support through *cstdbit*.

But back to the [De Bruijn sequence](https://en.wikipedia.org/wiki/De_Bruijn_sequence) code snippet: why should one know or even use it nowadays? The odds are that the CPU your code targets should support the intrinsic. For the ABIs [glibc](https://sourceware.org/glibc/) supports:

**ABI support for __builtin_ctzl (using gcc 15)**

| Architecture | Instructions / libcall | Notes                          |
|--------------|------------------------|--------------------------------|
| aarch64      | rbit + clz             |                                |
| alpha        | __ctzdi2               | ev67 added cttz                |
| arc          | ffs                    |                                |
| arm          | rsb + clz              |                                |
| csky         | __ctzdi2               |                                |
| hppa         | __ctzdi2               |                                |
| i686         | rep bsfl               |                                |
| loongarch    | ctz.w                  |                                |
| m68k         | bfffo                  |                                |
| microblaze   | __ctzsi2               |                                |
| mips         | __ctzdi2               | mips32r5 / mips64 added clz    |
| or1k         | __ctzsi2               |                                |
| powerpc      | cntlzw                 |                                |
| riscv        | __ctzsi2               | zbb extension added ctzw       |
| s390         | __ctzsi2               | z9-ec/arch7 added flogr        |
| sh           | __ctzsi2               |                                |
| sparc        | __ctzdi2               | VIS3 added lzd                 |
| x86_64       | rep bsfl               |                                |

So for a handful of ABIs, the compiler emits library calls (the **__ctzsi2** and **__ctzdi2**) that implement the builtin operation. For **gcc**, this is implemented in **libgcc**, which is commonly linked for all binaries and provides some optimized implementations for different architectures.

The [generic implementation](https://gcc.gnu.org/git/?p=gcc.git;a=blob;f=libgcc/libgcc2.c;h=df99c78eb2043854ecf0424aab44ef0f4943cf24;hb=HEAD#l718) uses an internal macro names *count_trailing_zeros* defined in a [very convoluted header](https://gcc.gnu.org/git/?p=gcc.git;a=blob;f=include/longlong.h;h=5ae250f7192dc79372cb17ad12cf2e35ba4d0019;hb=HEAD) that also provides different optimizations for multiple targets. The generic implementation boils down to:

```c
#define count_trailing_zeros(count, x)                  \
   do {                                                 \
     UWtype __ctz_x = (x);                              \
     UWtype __ctz_c;                                    \
     count_leading_zeros (__ctz_c, __ctz_x & -__ctz_x); \
     (count) = W_TYPE_SIZE - 1 - __ctz_c;               \
   } while (0)
 #endif
```

And *count_leading_zero* to:

```c
#define count_leading_zeros(count, x)                                 \
  do {                                                                \
    UWtype __xr = (x);                                                \
    UWtype __a;                                                       \
                                                                      \
    if (W_TYPE_SIZE <= 32)                                            \
      {                                                               \
        __a = __xr < ((UWtype)1<<2*__BITS4)                           \
          ? (__xr < ((UWtype)1<<__BITS4) ? 0 : __BITS4)               \
          : (__xr < ((UWtype)1<<3*__BITS4) ?  2*__BITS4 : 3*__BITS4); \
      }                                                               \
    else                                                              \
      {                                                               \
        for (__a = W_TYPE_SIZE - 8; __a > 0; __a -= 8)                \
          if (((__xr >> __a) & 0xff) != 0)                            \
            break;                                                    \
      }                                                               \
                                                                      \
    (count) = W_TYPE_SIZE - (__clz_tab[__xr >> __a] + __a);           \
  } while (0)
```

A more readable code is:

```cpp
constexpr uint32_t W_TYPE_SIZE = 32;
constexpr uint32_t BITS4 = W_TYPE_SIZE/4;

constexpr uint32_t gcc_clz (uint32_t v)
{
  static constexpr uint8_t __clz_tab[256] =
  {
    0,1,2,2,3,3,3,3,4,4,4,4,4,4,4,4,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,
    6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,
    7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,
    7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,
    8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,
    8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,
    8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,
    8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8
  };

  uint32_t a = v < (1U<<2*BITS4)
               ? v < (1U<<BITS4)   ? 0         : BITS4
               : v < (1U<<3*BITS4) ? 2 * BITS4 : 3 * BITS4;
  return W_TYPE_SIZE - (__clz_tab[v >> a] + a);
}

uint32_t gcc_ctz (uint32_t v)
{
  return W_TYPE_SIZE - 1 - gcc_clz (v & -v);
}
```

And how does it compare to the initial implementation that motivates this post? [It depends](https://godbolt.org/z/T1MW7665v). On both x86_64 and aarch64, for instance, it generates **more** code, with changes, and using a large table. So why does **libgcc** use this implementation instead of the De Bruijn-based one?

Because the generic code avoids the need for a 32x32 bit multiplier, which on simple CPUs (that most likely **do not** provide bit manipulation instructions) might have higher latency and lower throughput. The generic implementation is a trade-off between good-enough performance and code complexity, and since performance varies by CPU core, both **gcc** and **libgcc** also provide arch-specific optimizations. The [compiler-rt](https://github.com/llvm/llvm-project/blob/main/compiler-rt/lib/builtins/ctzsi2.c) also uses a multiplication-free implementation.

The De Bruijn sequence is an interesting method for finding the least- or most-significant set bit in a word, but either the toolchain or the architectures will likely provide better, faster interfaces for this.
