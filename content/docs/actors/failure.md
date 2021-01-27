---
title: Handling Failure
weight: 630
---

# Handling Failure

## Task: More robust echos

When actors that have spawned other actors are restarted,
those child actors are stopped by default.
As a consequence, the parent actor can respawn the children
without polluting the system with unused actors.

One possible strategy to react to the failure of a child
is for the parent to be restarted itself
stopping all other children.
(Whether this is the preferrable option
depends on the circumstances.)

First, modify the definition of the `EchoLoop` behavior
such that it enters a death pact with the spawned server and client,
meaning that it fails itself when one of its children fails.
Then, add supervision when starting the parent behavior
to cause a restart in case of failure.

In order to test the gained robustness,
modify the server and client to crash on certain messages.
Verify that the echo loop is still working
even after provoking crashs in the server or client.

