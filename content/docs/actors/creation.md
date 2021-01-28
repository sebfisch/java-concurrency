---
title: Creating Actors
weight: 610
---

# Creating Actors

Akka provides different APIs to program with actors.
In this section, we look at the typed object-oriented API
by discussing unit tests defined in `ObjectOrientedActorTests`
that demonstrate how to use it.
Akka provides a helper class to manage actors in tests.
We instantiate it as follows.

```java
static final TestKitJunitResource KIT = new TestKitJunitResource();
```    

## Sending and receiving messages

Our first test uses the test kit to spawn an actor 
that stops after receiving the first message.

```java
@Test
public void testThatTerminatorStopsOnMessage() {
    ActorRef<Object> terminator = KIT.spawn(Terminator.create());
    terminator.tell("");
    KIT.createTestProbe().expectTerminated(terminator);
}
```

In Akka, actors are created from `Behavior` instances
and the method `tell` on actor references is used
to send a message to an actor.
In object-oriented style,
`Behavior`s are defined as sub-classes of `AbstractBehavior`.
For simplicity, we define a static nested class.

```java
static class Terminator extends AbstractBehavior<Object> {
    static Behavior<Object> create() {
        return Behaviors.setup(Terminator::new);
    }

    Terminator(ActorContext<Object> ctx) {
        super(ctx);
    }

    @Override
    public Receive<Object> createReceive() {
        return newReceiveBuilder() //
            .onAnyMessage(msg -> Behaviors.stopped()) //
            .build();
    }
}
```

The type parameter is for corresponding messages.
For this test, we do not care about this type and use `Object`.
By convention, behaviour classes define a static factory method `create`
which is used to instantiate behaviors.
It should be defined using `Behaviors.setup`
to call the super-class constructor with the provided `ActorContext`.

Object-oriented behaviors specify how to respond to messages
by overriding the `createReceive` method,
which uses the builder pattern to register message handlers.
Handlers can return different behaviors
that are used subsequently.
In this case we use `Behaviors.stopped` to stop responding.

## Custom message types

As a best practice, behavior classes should define types
for corresponding messages as nested classes.
The following behavior defines two such types.

```java
static class Bouncer extends AbstractBehavior<Ping> {
    static class Ping {
        final ActorRef<Pong> sender;

        Ping(ActorRef<Pong> sender) {
            this.sender = sender;
        }
    }

    static class Pong {
    }

    static Behavior<Ping> create() {
        return Behaviors.setup(Bouncer::new);
    }

    Bouncer(ActorContext<Ping> ctx) {
        super(ctx);
    }

    @Override
    public Receive<Ping> createReceive() {
        return newReceiveBuilder().onAnyMessage(ping -> {
            ping.sender.tell(new Pong());
            return this;
        }).build();
    }
}
```

The `Ping` message wraps a reference to the actor that sent the message.
The `Pong` message is used to respond to pings.
When receiving a ping, actors with this behavior
respond to the actor wrapped in the message.
Instead of stopping, the handler returns the same behavior
in order to continue processing pings indefinitely.

The following test spawns an actor with this behavior.

```java
@Test
public void testThatBouncerBounces() {
    ActorRef<Ping> bouncer = KIT.spawn(Bouncer.create());
    TestProbe<Pong> probe = KIT.createTestProbe();
    IntStream.range(0, 5).forEach(i -> {
        bouncer.tell(new Ping(probe.getRef()));
        assertNotNull(probe.receiveMessage());
    });
}
```

Several messages are sent to the actor,
and each time the sender gets a response.

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
In object-oriented behaviors,
the own actor reference is available via `getContext().getSelf()`.

Write a test demonstrating that handshakes are answered.


