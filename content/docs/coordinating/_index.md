---
title: Coordinating Threads
weight: 400
---

# Coordinating Threads

In the previous part of this tutorial
we have used threads to make use of available processors
via parallel execution when rendering images of fractals.
We have not used language features for thread coordination.
Threads were used only to compute images faster.

However, the `FractalApp` actually does use 
a simple form of thread coordination
because rendering threads access and manipulate
shared mutable state to store computed pixels.

Running computations using threads is also called
asynchronous computation because by default
we do not know in which order statements of different threads
are executed.
Using primitives for thread coordination
we can establish a known order
at least for some of the executed statements
of different threads.
Coordinating threads in such a way is, therefore,
also called synchronization.

In this part we will discuss both simple and more involved
forms of thread coordination.
After learning about
[classic synchronization](synchronization)
primitives we discuss alternative ways of
[flexible locking](locking)
by means of another example application.
Finally, we will learn about
[completable futures](completable_futures)
that can be used to combine asynchronous computations
and discuss how to use scheduled executors
for 
[throttling tasks](throttling).

