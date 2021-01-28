---
title: Functional Interface
weight: 650
---

# Functional Interface

In this section we discuss an alternative API for creating behaviors for actors.
We use the same tests as before
and adjust the corresponding behavior classes to the functional actor API.

## Receiving messages

Here is an adjusted definition for the `Terminator` behaviour.

```java
static class Terminator {
    static Behavior<Object> create() {
        return Behaviors.receiveMessage(msg -> Behaviors.stopped());
    }
}
```

The class no longer extends `AbstractBehavior`
and uses `Behaviours.receiveMessage` to `create` an appropriate behaviour.

The adjusted `create` method for the `Bouncer` behavior
also uses `receiveMessage`.

```java
static Behavior<Ping> create() {
    return Behaviors.receiveMessage(ping -> {
        ping.sender.tell(new Pong());
        return Behaviors.same();
    });
}
```

We cannot `return this` from the message handler
and instead use `Behaviors.same`.

## Local state

The conversion of the `Calculator` is interesting
because it has local state for the total value.
In the functional style,
local state can be passed in arguments to methods creating behaviors.

```java { lineos=table, hl_lines=[8] }
static Behavior<Cmd> create(int total) {
    return Behaviors.receive(Cmd.class)
        .onMessage(GetTotal.class, msg -> {
            msg.sender.tell(total);
            return Behaviors.same();
        })
        .onMessage(AddFraction.class, msg -> {
            return create(total + msg.numerator / msg.denominator);
        })
        .build();
}
```

Note the recursive call to `create` 
with a new total value
in a message handler.
To register multiple handlers for different message types
we can the builder pattern with `Behaviors.receive`.

## Actor context

If we need to access the actor context,
for example to spawn child actors,
we can use `Behaviors.setup`.
Consider the `create` method for the `Computer` behavior.

```java
static Behavior<Request> create() {
    return Behaviors.setup(ctx -> {
        ActorRef<Calculator.Cmd> calc = ctx.spawnAnonymous( //
            Calculator.create(0));
        ctx.watch(calc); // death pact
        return computer(calc);
    });
}
```

The method `computer` expects a reference to a calculator
as argument representing its local state
and implements the `Computer` behavior.

```java
private static Behavior<Request> computer(ActorRef<Calculator.Cmd> calc) {
    return Behaviors.receive(Request.class)
        .onMessage(Forward.class, msg -> {
            calc.tell(msg.cmd);
            return Behaviors.same();
        })
        .onMessage(Compute.class, msg -> {
            calc.tell(new Calculator.AddFraction(100, msg.arg));
            return Behaviors.same();
        })
        .build();
}
```

## Task: Refactor echo loop

Modify the definition of `EchoLoop`
to use the functional instead of the object-oriented API
for the root behavior of the actor system.

