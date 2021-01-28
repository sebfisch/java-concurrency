---
title: Chat Application
weight: 660
---

# Chat Application

In this section we implement a more complex application using actors.
The application can be used to chat on the command line
allowing clients on different computers to connect to a remote chat server.

The messages for the protocol are defined in the classes `ChatClient` and `ChatServer`.
We will only discuss selected parts of the chat client implementation.
Implementing the chat server is left as an exercise.
The command line applications for the client and server
can be started using `RemoteChatClient` and `RemoteChatServer`, respectively.

## Implementing the chat client

Chat clients can be created by passing a user name and the path to a remote chat server.

```java
public static Behavior<Event> create(String name, String path) {
    return Behaviors.setup(ctx -> new ChatClient(ctx, name, path));
}

private String name;
private final String path;
private ActorRef<ChatServer.Request> server;

private ChatClient(ActorContext<Event> ctx, String name, String path) {
    super(ctx);
    this.name = name;
    this.path = path;
    ctx.getSelf().tell(Created.INSTANCE);
}
```

When a chat client is created (or restarted)
it sends itself a `Created` message, 
which is defined as an enum because it does not have any contents.

```java
private enum Created implements Event { INSTANCE }
```

As it is only used internally, the type is declared private.
The following method handles such messages by trying to login to the server.

```java
private ChatClient respond(Created msg) {
    ActorContext<Event> ctx = getContext();
    ActorRef<Event> self = ctx.getSelf();
    ctx.getSystem().classicSystem().actorSelection(path) //
        .tell(new ChatServer.Login(name, self), Adapter.toClassic(self));
    return this;
}
```

We use actor selection again, to communicate with a remote actor.
This time however, we do not resolve an actor reference immediately.
Instead we use `ActorSelection.tell` to send a message to the server
without resolving its reference.
The server will respond with a message containing its reference
so we can store it in the local state.

Another interesting aspect is when to logout from the server.
Actors can not only respond to messages but also to signals
used internally by Akka.
In its `createReceive` method, 
the chat client uses the following calls on the receive builder.

```java
    .onSignal(PreRestart.class, signal -> quit())
    .onSignal(PostStop.class, signal -> quit())
```

Those signals are sent before the actor restarts and after it was stopped.
They are a good place to perform cleanup actions.
We use them here to logout from the chat server.


## Task: Implement the chat server

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
The latter can be given a user name and a port as command line arguments
to avoid conflicts.
Are existing users notified about new users?
What happens, if a client tries to login with an already exiting name?
Are messages sent by one user displayed to other users correctly?
Are existing users notified about leaving users?

As a bonus, write unit tests that document
that the chat server responds to messages as intended.

