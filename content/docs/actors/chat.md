---
title: Chat Application
weight: 660
---

# Chat Application

## Task: Implement chat server

The `ChatServer` class defines messages
that clients can send to a chat server.
Implement a corresponding behavior
after a conscious decision for the functional or object-oriented API.

The behavior should react to each possible message
with appropriate responses to clients as well as local state changes.
More specifically, the behaviour should
capture local state associating client references with names.
`Login` requests should be answered with a `LoggedIn` event
or with an `Error` event if the chosen name is already taken.
Additionally, other users should be alerted to successful registrations 
via `UserJoined`.
When a registered client requests to `Send` a message,
other clients should be notified via `NewMessage`.
Unregistered clients should not be able to send messages
but receive an `Error` event.
When a registered client quits,
others should be notified via the `UserLeft` event.

In order to test your implementation,
start a `RemoteChatServer` and several `RemoteChatClient`s.
The latter can be given a name and a port as command line arguments
to avoid conflicts.
Are existing users notified about new users?
What happens, if a client tries to login with an already exiting name?
Are messages sent by one user displayed to other users correctly?
Are existing users notified about leaving users?

As a bonus, write unit tests that document
that the chat server responds to messages as intended.

