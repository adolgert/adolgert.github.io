---
layout: post
author: adolgert
title: Finite state machines as a design pattern for scientific code
description: The mathematical definition of a finite state machine is a template for a design pattern for scientific simulation.
keywords: finite state machine, design patterns, scientific software
usemathjax: true
---

The mathematical definition of a finite state machine is a template for a design.

# Introduction

We write code all the time that does some sort of time step.
It could be a custom integration for a particular ordinary differential
equation. It could be a stochastic Markov method that carries state
from the last two steps. Every time we write code like this, there
are some parts that are general, such as multiplying a transformation
matrix by a vector, and there are some parts that are particular
to this problem, such as initializing the vector from a single
parameter. Each time we write code like this, we try
to make the code clearer and more useful by finding a structure
that balances the general and the particular.

Mathematicians use a model of computation called a finite state machine
(FSM) in order to understand how computation, itself works. We can
use this model of computation as a guide to writing all kinds
of time-stepping code. The mathematical background gives some reassurances
about how generally this guide applies, and it offers a way
to decompose a time-stepping function into composable parts
that have recognizable function.

More importantly, a function written as a FSM will have an 
interface that makes it easy for other code to call.
Our highest goal is to be able to write several simulations
which have different underlying time-stepping algorithms, but
our goal is to still be able to use the same code on top
of them for optimization or parameter tuning.

There are several definitions of finite state machines. 
I'll present two versions here, one common version, called the
Moore machine, and one less common, called a machine in a
category (Arbib and Manes 1974).

# Moore model


The Moore model is a sequential machine that has five parts.
Some of the five parts are data and some are functions.

* *State*---When we think of a FSM, we think of the state, $$Q$$, as being a known set of
	internal states.

* *Input Token*---Every time step, we feed some data to the
	finite state machine. We call each new value $x$, and each $$x$$ is from
	the class $$X_0$$ of allowed inputs. It's the domain of the FSM.

* *Output Token*---The output of the FSM is a value $$y$$ from the class $$Y$$
	of possible outputs.

* *Dynamics*---The heart of a FSM is a function that
	transforms the state and an input token into a new state.
	As an equation, it's $$\delta: Q \times X_0\rightarrow Q$$.

* *Output function*---The output function is equivalent to an
	observer pattern. It transforms the state and the input into the output,
	$$\lambda:Q\times X_0\rightarrow Y$$.

There is also an initial state that's part of the machine.

From this, we can see a software design pattern that suggests the following
structure.

~~~R
dynamics <- function(state, step_input) {
  ...
  new_state
}

output_function <- function(state, step_input) {
  ...
  fsm_output
}


fsm_step <- function(fsm, step_input) {
  output_function <- output_function(fsm$state, step_input)
  fsm$state <- dynamics(fsm$state, step_input)
}

y <- fsm_step(fsm, x)
~~~

It's a little weird, from a software perspective, that the Moore machine
calculates output of the FSM separately from calculating the next
state. There is another version of this called the Mealy machine
that doesn't have this behavior. People who work in linear control
have reasons for this structure, and I default to trusting them, in
the absence of further research.

This pattern implicitly organizes a time-stepping algorithm into
input, output, state, dynamics, and observer. That, itself, is
useful for clarifying when two time-stepping algorithms have the
same dynamics, but different observers. It can also force the author
to clarify what all of the inputs are. For instance, the random
number generator can be considered part of the state, or each
set of random numbers can be considered part of the input.


# Machines in a category

A paper by Arbib and Manes asks how to write a FSM such that it will
be possible to express not just linear control problems but also
stochastic automata and tree automata (Arbib and Manes 1974). They want to unify the interface
to these types so that it's possible to write an inverse problem on
top of them. An inverse problem asks which inputs would lead
to a given set of outputs. It's an example of a higher-level function
that would reuse a time-stepping function, if that function had
the right interface.

They suggest a generalization of the Moore model described above.
Instead of thinking of the input to the \fsm as a set of tokens
$x$ from $X_0$, they suggest thinking of the input as a set of
transformations which take $Q \rightarrow Q\times X_0$. In other
words, you don't pass in data. You pass in something that transforms
the internal state of the \fsm into a new internal state.

The machine in a category has seven parts.

* *State*---This is the same state, $$Q$$.

* *Input process*---Each input to the FSM is a functor, $$X$$,
	that transforms $$X:Q\rightarrow Q\times X$$.

* *Output token*---Still a token $$y$$ from $$Y$$.

* *Dynamics*---The dynamics now acts on the state, $$Q$$,
	not the state and a new token.

* *Initial state object*---The initial state object is any
	representation of the state, but it has help from the next in the list.

* *Initial state process*---Like the input process, the initial state
	is a function that transform the initial state object into a valid
	internal state, $$Q$$.

* *Output token*---This is a value $$y$$ in the class $$Y$$.

* *Output process*---This observer transforms the state into
	an output token, $$\beta:Q\rightarrow Y$$. It doesn't take the input
	token as an argument.

This list is similar to the Moore machine above, but it gives us more
ways to differentiate among time-stepping algorithms. The input process
can be, for code, a strategy pattern that accepts different kinds of inputs,
as long as they can be combined with the FSM state, $$Q$$. The initial
state object could be some simple parameterization which the initial state
process turns into a state, $$Q$$. That makes the initial state process
a builder pattern.

~~~R
dynamics <- function(state) {
  ...
  new_state
}

output_function <- function(state) {
  ...
  fsm_output
}

input_process <- function(input_token) {
  function(state) {
    state + input_token
  }
}

fsm_step <- function(fsm, step_input) {
  state <- step_input(fsm$state)
  output_function <- output_function(state)
  fsm$state <- dynamics(state)
}

y <- fsm_step(fsm, input_process(x))
~~~

# Conclusion

This design pattern is a central structure for time-stepping
algorithms. It separates the more pure math of the dynamics
from translations of input and output parameters and data.
