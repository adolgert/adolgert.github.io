---
layout: post
author: adolgert
title: Checkout Testing for Heterogeneous Clusters
keywords: optimization, compilers, unit testing, checkout testing, scientific software
description: Checkout testing for code compiled on a different architecture
---

Checkout testing for code compiled on a different architecture.

## Introduction

The nodes on this cluster have different architectures because they were bought
in small batches. When I compile research code on a node, it sometimes won't run
on another node. That's the best case scenario, and the worst is that it will
run but give wrong results.

If I compile code with full optimization on one node and run on a much older
node, it will sometimes stop with a Unix signal, SIGILL.

```
Illegal instruction
```

That means the compiler inserted a machine code that uses newer chip features,
and they aren't present on this chip.

I would find the oldest machine on the cluster and compile there, but
chip features aren't a march of progress. You can look in `/proc/cpuinfo`
for the flags, which are a rough measure of each chip's capabilities.
Here's an [Intel i7-8809G](https://ark.intel.com/content/www/us/en/ark/products/130409/intel-core-i7-8809g-processor-with-radeon-rx-vega-m-gh-graphics-8m-cache-up-to-4-20-ghz.html), model 158, stepping 9.

```
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov
pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb
rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology
nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx est tm2
ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt
tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch
cpuid_fault epb invpcid_single pti ssbd ibrs ibpb stibp tpr_shadow vnmi
flexpriority ept vpid ept_ad fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms
invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves
dtherm ida arat pln pts hwp hwp_notify hwp_act_window hwp_epp md_clear flush_l1d
```

We can avoid the problem by being conservative in our compiler flags.


## Compiling for safety

It's tempting to recompile with `-O2`, in case that's safer. The list of optimizations
gcc uses at `-O3` isn't as dangerous as it used to be, and reducing
the optimization level isn't exactly a direct approach to a problem
with target architecture.

The most basic target architecture is SSE2, which means choosing both `-march=SSE2 -mcpu=SSE2`.
That restricts the compiler to using features from 2000. That seems like it
would be a bad performance hit. Worse, it would require compiling all the
components at this level. Our toolchains, these days, are huge, and there's
not easy way to specify the architecture for all of them. I'd have
to go package by package, library by library, looking for how to configure it.


## Silent faults

The deeper problem is that I've seen code that runs on a different architecture
but gives wrong results. I've seen this twice in three years, that someone
noticed. I don't have the code as an example.

I interpret these two events to indicate that the same machine code,
on two different architectures, can give significantly different results.
In particular, machine code with one architecture target can run
on another and give wrong results without raising an exception.


## Checkout testing

Checkout testing examines behavior of a system that's installed in its
target environment. The term comes from engineering. While acceptance
testing verifies to a client, or user, that the software works, checkout
testing applies *in place.* I'm starting to use checkout testing
on this heterogeneous cluster.

I can't run the full suite for every job on the cluster. The full
suite serves many other purposes. It does longer runs for randomized
fault searches. It has test-driven design user tests. I need to
pick the checkout test.

Important architecture changes have advanced multimedia and
floating-point processing. These affect vectorized mathematical
loops, so it's these that I'll test, in the absence of evidence
for what causes problems.

That means I have a new unit testing flag, `--checkout`, that
exercises code with high math complexity but skips trying all
possible parameter values. I compile with `-O3`, use the
native architecture, and compile another version if the checkout
fails.
