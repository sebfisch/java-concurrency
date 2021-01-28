---
title: Using Actors
weight: 600
---

# Using Actors

Actors are an alternative programming model for concurrent applications.
They avoid the need for synchronization
because mutable state is encapsulated inside actors and not shared.
Communication between actors happens asynchronously
avoiding confusing control flow across object boundaries like with synchronous method calls.
Actors process their messages sequentially,
avoiding the need for synchronizing access to local state.
They are much more lightweight than threads.
An application can contain millions of actors without performance problems.

Maybe the most important selling point for actors is their support for fault tolerance.
Actors can supervise each other, 
and there are easy to understand strategies for handling failures.
For example, actors can be restarted automatically, 
to keep the system running and in a valid state.

[Akka](https://doc.akka.io/docs/akka/current/typed/guide/index.html) 
provides a mature implementation of actors for Java.
In this part of the tutorial, we will discuss how to
[create actors](creation),
how to define
[systems](systems)
of multiple actors, and how to
[handle failures](failure)
to improve fault tolerance.
We will also see
[remote actors](remote),
discuss an alternative
[functional API](functional),
and finally implement a 
[chat application](chat)
as an example of a realistic application using actors.

