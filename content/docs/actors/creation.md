---
title: Creating Actors
weight: 610
---

# Creating Actors

## Task: Add handshake test

Add a new behavior `Handshake`
defined as a nested class inside `ObjectOrientedActorTests`.
Define a corresponding message type `Self`
wrapping a reference to an actor
participating in the handshake.
Actors using the `Handshake` behavior
should answer `Self` messages with another `Self` message,
sending their own reference to the wrapped reference
and then continue waiting for more handshakes.

Write a test demonstrating that handshakes are answered.


