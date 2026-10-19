---
layout: post
title: The recent glibc FMA optimization
---

*And why the commit message alone does not give all the context. Originally published on [Substack](https://zatrazz.substack.com/p/the-recent-glibc-fma-optimization).*

A recent [Phoronix article](https://www.phoronix.com/news/Glibc-4x-FMA-Improvement-Zen) brought some attention to **glibc's FMA** (fused multiply-addition) math optimisations on **x86**, and, based on the comments, some people are confused about what it means and why it matters.

I added some context to the [cover letter](https://sourceware.org/pipermail/libc-alpha/2025-November/172447.html), and I will provide a more thorough explanation of why the current FMA implementation in glibc is really slow, why it might become important to have a faster **FMA** implementation, and where this improvement could make a difference.

The [Wikipedia article](https://en.wikipedia.org/wiki/Multiply–accumulate_operation) provides an extensive explanation of FMA. What is important to note is that all the C standards specify that the **fma** functions should be correctly rounded in all supported rounding modes, and that glibc assumes and requires this. Any precision issue is considered a bug, and any new implementation is supposed to follow the C standard (there are still some [discussions about errno handling](https://sourceware.org/bugzilla/show_bug.cgi?id=6801) though).

The next glibc version, 2.43, imported even more implementations from [CORE-MATH](https://core-math.gitlabpages.inria.fr/) to provide correctly rounded implementations of the high-level math functions **acosh**, **asinh**, **atanh**, **erf**, **erfc**, **lgamma**, and **tgamma**. These are the functions that showed the worst precision, with some results differing by **7 ulps** from the expected result.

And some CORE-MATH implementations rely on the **FMA** operation on the fast path to provide a better estimate and avoid the fallback code (which can be slow for some inputs). The implementations were originally designed for recent hardware and assumed that a lot of floating-point support was available as *hardware instructions* rather than *functions from the C runtime*.

And this is true for the most glibc targets: **aarch64** supports in the baseline ABI, **arm** since around 2013, **powerpc** since 1990, **RISC-V** includes in both the **F** (Single-Precision) and **D** (Double-Precision), and **x86_64** since 2011 (AMD Bulldozer) or 2013 (Intel Haswell).

The issue is how you deploy the **glibc** and which target you use. For instance, on a generic arm build (*--target=arm-linux-gnueabihf* and compiler targeting *armv5*), the glibc **acosh** benchmark profile shows:

```
$ perf report --stdio
[...]
# Overhead  Command      Shared Object      Symbol
# ........  ...........  .................  .................................
#
    60.53%  bench-acosh  libm.so            [.] __fmal
    17.64%  bench-acosh  libm.so            [.] feclearexcept
    11.79%  bench-acosh  bench-acosh        [.] bench_start
     8.99%  bench-acosh  libm.so            [.] __acosh_finite@GLIBC_2.15
     0.56%  bench-acosh  libm.so            [.] __acoshl
     0.39%  bench-acosh  bench-acosh        [.] main
     0.07%  bench-acosh  libm.so            [.] as_acosh_refine
```

The fma support was added only with **VFPv4** (Vector Floating Point version 4) on **armv7-a**. Building **glibc** targeting *armv7a-linux-gnueabihf* shows a different profile:

```
$ perf report --stdio
[...]
# Overhead  Command      Shared Object      Symbol
# ........  ...........  .................  ...............................
#
    70.84%  bench-acosh  libm.so            [.] __acosh_finite@GLIBC_2.15
    19.49%  bench-acosh  bench-acosh        [.] bench_start
     4.33%  bench-acosh  libm.so            [.] __acoshl
     1.57%  bench-acosh  bench-acosh        [.] main
     1.08%  bench-acosh  libm.so            [.] 0x000000000000af90
     0.78%  bench-acosh  libm.so            [.] 0x000000000000af94
     0.74%  bench-acosh  libm.so            [.] __fma_vpfv4
```

The **__fma_vpfv4** was added in **glibc 2.43** to allow **armv7-a** to use the **FMA** instruction when available. It uses Linux kernel *hwcap* along with *iFUNC* to issue the **FMA** instruction; if glibc is built with *VPFv4* as the default, *iFUNC* usage is also elided, and the **FMA** instruction is called directly.

Now, back on the original x86 patch, the issue is that both [**x86_64-v1** and **x86_64-v2**](https://en.wikipedia.org/wiki/X86-64#Microarchitecture_levels) ISAs do have the **FMA** instruction. And the old algorithm, named **ldbl-96**, was used on processors that lack the **FMA** instruction. The **ldbl-96** implementation uses **long double** operations, which on x86 will be translated to **x87** instructions, and these have a series of *bottlenecks*: they are stack-backed (the compiler has to generate dozens of "shuffling" instructions to get the inputs on the right place), they do not allow vectorisation, it operates internally on a different precision (80-bits double extended, which for this specific usage is required), and it requires extra round/truncate operations, and recent x86 chips do not optimize such instructions.

However, even the current **dbl-64** implementation (now used on all targets) has a significant drawback: it relies on setting the rounding mode for different calculations (first to FE_TONEAREST and then to FE_TOWARDZERO) to obtain correctly rounded results. For most CPUs, this adds **significant** performance overhead because it requires executing a typically slow instruction (to get/set the floating-point status), necessitates flushing the pipeline, and breaks some compiler assumptions/optimisations.

The [new proposed implementation](https://sourceware.org/pipermail/libc-alpha/2025-November/172449.html), originally written by Szabolcs for **musl**, utilises mostly integer arithmetic, with floating-point arithmetic used to raise expected exceptions, without the need for **fenv.h** operations. This shows a significant improvement over [most hardwares](https://sourceware.org/pipermail/libc-alpha/2025-November/172449.html), although most of them will never be used because the architecture already supports the **FMA** instruction.

The whole idea of optimising the generic **fma/fmaf** implementation was to improve the performance of the new *CORE-MATH* implementation on **old hardware** and on **ABIs that lack the fma operation**. And at least on armhf, it improved performance twofold.
