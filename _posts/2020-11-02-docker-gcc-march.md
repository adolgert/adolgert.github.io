---
layout: post
author: adolgert
title: Native architecture for gcc on Docker
keywords: docker, gcc, architecture, optimization, compilers, scientific software
description: How gcc chooses the target architecture in a docker image
---

How gcc chooses the target architecture in a docker image

# Problem Statement

I've had trouble with libraries on our cluster because I compile on one
node and it doesn't run on another node. Worse, it runs but gives a wrong
answer. This makes sense because the nodes have different CPU architectures.
While they are all Intel CPUs, they are different models from different years.

I've been wondering how to guard against this problem and whether
using [Docker](www.docker.com) and [Singularity](singularity.lbl.gov/) images would help.


# Docker and the native architecture

On Linux, Docker doesn't change what CPU instructions are available. There
isn't an emulation layer for that, unless you're running ARM on a Linux
box. On Windows and Mac, it's different, but I'm focusing on Linux here.

There is nothing intrinsic to Docker that will fix my problem. If
I compile for the native architecture when I make an image,
that's the image I get. My machine at home is newer than some of
the machines at work, despite having seriously less memory and
memory bandwidth.

If Docker isn't smoothing over the problem of changes in the
target architecture, then how is this not a problem more
often? Why is it a problem for the code I compile on my
cluster?


# How Gcc chooses architectures

The compiler chain on Linux uses gcc as its backbone.
Architecture choices are made by gcc and how installations
call gcc. Somewhere in this chain, I've been running
into a choice of a too-specific architecture.

For gcc, the target architecture comes from a command-line
argument called `march`. The baseline for my cluster is `march=x86_64`.
Other options are processor-specific, such as `march=haswell`.
It will look at the current processor if you specify `march=native`.

If you don't give gcc an architecture, it chooses a default.
Here are [three ways to find the default architecture of
gcc](https://stackoverflow.com/questions/11727855/obtaining-current-gcc-architecture).

```bash
$ echo | gcc -v -E - 2>&1 | grep cc1
 /usr/lib/gcc/x86_64-unknown-linux-gnu/4.9.0/cc1 -E -quiet -v - -mtune=generic -march=x86-64
```

```bash
$ gcc -Q --help=target | grep march
  -march=                           x86-64
$ gcc -m32 -Q --help=target | grep march
  -march=                           i686
$ i686-w64-mingw32-gcc -Q --help=target | grep march
  -march=                           pentiumpro
```
Lastly, it defaults to the target machine, in the first
part of the dumpmachine triple.
```bash
gcc -dumpmachine
```

The short version is that the cluster machines report
`x86_64-redhat-linux`, so their default target is x86, which is
what I want.

# Conclusion

The problems I see come from individual packages using `march=native mtune=native`.
I need to find those and root them out.
I can try hinting by exporting `CFLAGS` and `CPPFLAGS`, but they don't always
work.
