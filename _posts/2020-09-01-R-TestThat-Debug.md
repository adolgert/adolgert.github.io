---
layout: post
author: adolgert
title: Modifying testthat Unit Tests for Debugging
description: How to call testthat in order to debug the function
keywords: unit testing, R language, test-driven design, scientific software
---

How to call testthat under RStudio in order to debug the function.

## Introduction

When I see an error in a unit test, and I need to figure
out how to fix it, I'd like to have all of my tools
available. There is no reason to be stuck with print-statement
debugging.

This entry shows two techniques: how to add convenience
functions for calling debuggers and how to use editor
breakpoints to trace unit tests.

## Convenience functions

My projects always have a directory structure that places
tests for the source file `R/filename.R` under `tests/testthat/test-filename.R`. I make it easier to test a single file
by making functions that take `filename.R` as the argument.

~~~R
# Place in .Rprofile
test_file_path <- function(filepath) {
  test_file <- paste("test", filepath, sep = "-")
  rprojroot::find_package_root_file("tests", "testthat", test_file)
}

test_progress <- function(filepath, reporter) {
  testthat::test_file(
  	test_file_path(filepath),
  	reporter = reporter
  )
}

# This turns off Hadley's weird unit test messages.
testf <- function(filepath) {
  test_progress(
    filepath, testthat::ProgressReporter$new(show_praise = FALSE)
	)
}

# Drop into debug if the test fails.
testd <- function(filepath) {
  test_progress(filepath, testthat::DebugReporter$new())
}
~~~

They also turn off the infantilizing "praise" option.


## Debugging a unit test interactively

The code above also shows one way to look at stack frames
when a test fails. It calls `testthat::test_file` with
the `DebugReporter`. This will stop at the failure and
show variables that are defined in a given stack frame.
That's helpful, but it isn't as helpful as being able to
use an interactive debugger.

There are two ways to invoke an interactive debugger from
a unit test. One is to insert `browser()` commands into the
source code. You will need to execute the function again, or
rebuild the source, in order for the `browser()` command to
be invoked. The trouble with this is that you don't want to
leave `browser()` commands in the code by accident, and
accidents will happen.

I'd like to use [editor breakpoints](https://support.rstudio.com/hc/en-us/articles/205612627-Debugging-with-RStudio#stopping-on-a-line) instead.
The `testthat` library [won't be able to use editor
breakpoints](https://github.com/r-lib/testthat/issues/116) because it usually starts a separate session,
outside RStudio, to run its tests.
If we're willing to relax our requirement that unit tests
run in a pristine environment, then we can run those tests
ourselves.

In order to run the unit tests ourselves, we need to read
the file with unit tests and run those tests in the local
environment. A quick way to do this

~~~R
test_file_trace <- function(filename) {
  env <- new.env()

  test_that <- function(desc, code) {
    eval(substitute(code), env)
  }
  env$test_that <- test_that
  source(filename, env)
}
~~~

When we want to debug a particular function, we can
set a breakpoint by left-clicking to the left of
the line numbers in RStudio.  Then mark the function
for tracing
with [debugonce](https://stat.ethz.ch/R-manual/R-devel/library/base/html/debug.html). Finally run unit tests in that file
using the code above, `test_file_trace(filename)`.
That gives interactive debugging using the browser.

It's possible to modify this function further
so that it picks out tests that match a
regular expression.
