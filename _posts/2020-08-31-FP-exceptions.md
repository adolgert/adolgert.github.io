---
layout: post
author: adolgert
title: Testing for Floating Point Exceptions
keywords: unit testing, scientific software
description: Unit testing for divide-by-zero behavior.
usemathjax: true
---

Acceptance testing for divide-by-zero behavior.

# Floating point isn't the real numbers

If you're used to writing scientific code in a language, then
you know the results of these calculations:

~~~
x = sqrt(-2)
y = 1 / 0
z = 2^(-50) / 2^10
~~~

These are all cases where something goes wrong, to a greater
or lesser extent, and we rely on however our language
of choice behaves.

The problem is that not all languages give the same
answers for these lines of code, and the same language
can give different answers for different installations
or operating system settings.


# Floating-point exceptions

Those lines of code are examples of three of the five
kinds of floating-point exceptions. The CPU will flag
these exceptions when they happen.

* *Invalid operation*---These include $$\sqrt{-2}$$ and
  $$0 / 0$$. If the exception is not caught, they result
  in a NaN value.

* *Division by zero*---These result in either an exception
  or Inf or -Inf.

* *Overflow*---When an operation results in a number too large to represent, this can result
either in infinity or the largest representable number, depending on settings.

* *Underflow*---When an operation results in a number that isn’t zero but is too small to
represent, it can result in plus or minus zero, or the smallest representable number, or a
special value called a subnormal.

* *Inexact*---The result can’t be represented exactly with the ieee standard. For example,
$$2.0 / 3.0$$. This happens a lot.

An inexact result is hardly unusual, given the binary representation
of floating point, but it is still possible to catch this as an exception.
The floating-point standard, IEEE 754, is designed to be configurable (Overton 2001),
and exceptions raised on the CPU are the first link in a chain.

When the CPU sets a flag indicating that an exception has
happened, the operating system has to check for that flag
and clear it. The operating system then decides whether to
raise an interrupt.

Each language, from C++ to Julia, has a runtime that receives
interrupts from the operating system. That runtime then decides
whether to generate a language-level exception or whether to
ignore it.


# Acceptance testing

It used to be important to include tests of floating-point exceptions
in C++ because settings in the operating system or installation could
change behavior. I don't know to what extent languages are
able to provide stable behavior for divide-by-zero and NaN-generating
calls, but it seems like a good spot for an acceptance test during
installation.


# Bibliography

Overton, Michael L. Numerical computing with IEEE floating point arithmetic. Society for Industrial and Applied Mathematics, 2001.
