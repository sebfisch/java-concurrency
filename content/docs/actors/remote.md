---
title: Remote Actors
weight: 640
---

# Remote Actors

## Task: Implement remote echo client

Complete the definition of `RemoteEchoClient`
which can be used as a counterpart for `RemoteEchoServer`.
When started, the application should start an actor
with the behavior `EchoClient` as root of the actor system.
The application should read lines from standard input,
send them to the started `EchoClient` actor via the `SendRemote` message
using the path of the `RemoteEchoServer`,
and exit when the line contains `quit`.

Start `RemoteEchoServer` and `RemoteEchoClient`
to test if messages are sent back to the client.

