---
title: Handling Failure
weight: 630
---

# Handling Failure

In this section we will give a first impression
how actor systems can be made resilient against failure
by supervising actors, for example, to restart them automatically.
We will use tests again to document the corresponding API.

## Failing behaviors

The Akka documentation cites a distinction between failures and validation errors:

> A __validation error__ means that the data of a command sent to an actor is not valid, 
> this should rather be modelled as a part of the actor protocol than make the actor throw exceptions.
>
> A __failure__ is instead something unexpected or outside the control of the actor itself, 
> for example a database connection that broke. 
> Opposite to validation errors, it is seldom useful to model failures as part of the protocol 
> as a sending actor can very seldomly do anything useful about it.

For simplicity, we will use examples of failures that would better be handled as validation errors.

To demonstrate supervision,
we define a calculator that may fail due to division by zero.
The calculator defines a message `GetTotal` to query the current total.
The message `AddFraction` is meant to provoke a complex computation,
dividing the wrapped numbers and adding the result to the current total.

```java
static class Calculator extends AbstractBehavior<Calculator.Cmd> {
    interface Cmd {
    }

    static class GetTotal implements Cmd {
        final ActorRef<Integer> sender;

        GetTotal(ActorRef<Integer> sender) {
            this.sender = sender;
        }
    }

    static class AddFraction implements Cmd {
        final int numerator;
        final int denominator;

        AddFraction(int numerator, int denominator) {
            this.numerator = numerator;
            this.denominator = denominator;
        }
    }
    // to be continued
}
```

The calculator stores the current total in local state.
The initial value must be passed to the `create` method.

```java
static Behavior<Cmd> create(int total) {
    return Behaviors.setup(ctx -> new Calculator(ctx, total));
}

private int total;

Calculator(ActorContext<Cmd> ctx, int total) {
    super(ctx);
    this.total = total;
}

@Override
public Receive<Cmd> createReceive() {
    return newReceiveBuilder().onMessage(GetTotal.class, msg -> {
        msg.sender.tell(total);
        return this;
    }).onMessage(AddFraction.class, msg -> {
        total += msg.numerator / msg.denominator;
        return this;
    }).build();
}
```

The messages are handled in the way described above.
An important detail is that we use integer division
to get an `ArithmeticException` on division by zero.

The following test shows how the calculator works 
when there is no exception.

```java
@Test
public void testThatCalculatorCalculates() {
    ActorRef<Calculator.Cmd> calculator = KIT.spawn(Calculator.create(0));
    TestProbe<Integer> probe = KIT.createTestProbe();

    calculator.tell(new AddFraction(5, 2));
    calculator.tell(new AddFraction(1, 2));
    calculator.tell(new GetTotal(probe.getRef()));

    assertEquals(2, (int) probe.receiveMessage());
}
```

After some complex computations
the total can be queried without issue.
In the next test we provoke division by zero,
and observe that typed actors are stopped on failure
when no supervision strategy is defined.

```java
@Test
public void testThatCalculatorStopsOnCrash() {
    ActorRef<Calculator.Cmd> calculator = KIT.spawn(Calculator.create(0));
    calculator.tell(new AddFraction(1, 0));
    KIT.createTestProbe().expectTerminated(calculator);
}
```

## Supervising behaviors

In order to maintain the calculator's responsiveness in the presence of failures,
we can associate supervision strategies before spawning an actor.
There are basically two alternatives to stopping: resuming and restarting.

The following test shows that a resumed actor acts as if nothing happened.

```java
@Test
public void testThatResumingCalculatorResumesAfterCrash() {
    Behavior<Calculator.Cmd> resuming = //
            Behaviors.supervise(Calculator.create(0)) //
                    .onFailure(SupervisorStrategy.resume());
    ActorRef<Calculator.Cmd> calculator = KIT.spawn(resuming);
    TestProbe<Integer> probe = KIT.createTestProbe();

    calculator.tell(new AddFraction(5, 2));
    calculator.tell(new AddFraction(1, 0));
    calculator.tell(new GetTotal(probe.getRef()));

    assertEquals(2, (int) probe.receiveMessage());
}
```

Even after division by zero occured,
the actor responds to the message to query the total.
The response to this message is unaffected by the failure.

A restarted actor also continues to respond to messages.

```java
@Test
public void testThatRestartingCalculatorRestartsAfterCrash() {
    Behavior<Calculator.Cmd> restarting = //
            Behaviors.supervise(Calculator.create(0)) //
                    .onFailure(SupervisorStrategy.restart());
    ActorRef<Calculator.Cmd> calculator = KIT.spawn(restarting);
    TestProbe<Integer> probe = KIT.createTestProbe();

    calculator.tell(new AddFraction(5, 2));
    calculator.tell(new AddFraction(1, 0));
    calculator.tell(new GetTotal(probe.getRef()));

    assertEquals(0, (int) probe.receiveMessage());
}
```

But with the restart strategy,
the total queried after the failure is reset to the initial value.

## Watching actors

Actors can watch other actors to be notified when they stop.
An interesting side effect of watching an actor is the so called death pact
which means that the termination signal sent due to watching
causes an exception if it is not handled
which in turn causes the watching actor to stop if no supervision strategy is defined.

In order to demonstrate a death pact,
we define another behavior that spawns and watches a calculator.

```java { lineos=table, hl_lines=[24] }
static class Computer extends AbstractBehavior<Request> {
    interface Request {}

    static class Forward implements Request {
        final Calculator.Cmd cmd;

        public Forward(Calculator.Cmd cmd) {
            this.cmd = cmd;
        }
    }

    static class Compute implements Request {
        final int arg;

        public Compute(int arg) {
            this.arg = arg;
        }
    }

    static Behavior<Request> create() {
        return Behaviors.setup(ctx -> {
            ActorRef<Calculator.Cmd> calc = ctx.spawnAnonymous( //
                Calculator.create(0));
            ctx.watch(calc); // death pact
            return new Computer(ctx, calc);
        });
    }

    private final ActorRef<Calculator.Cmd> calc;

    Computer(ActorContext<Request> ctx, ActorRef<Calculator.Cmd> calc) {
        super(ctx);
        this.calc = calc;
    }

    @Override
    public Receive<Request> createReceive() {
        return newReceiveBuilder()
            .onMessage(Forward.class, msg -> {
                calc.tell(msg.cmd);
                return this;
            })
            .onMessage(Compute.class, msg -> {
                calc.tell(new Calculator.AddFraction(100, msg.arg));
                return this;
            })
            .build();
    }
}
```

A computer handles two types of messages.
`Forward` can be used to forward a calculator command to the wrapped calculator.
`Compute` causes a calculation that involves division by the wrapped argument.
The most interesting part is the call to the `watch` method on the actor context
which leads to a death pact because the corresponding termination signal is not handled.

The following test shows how the computer computes if there are no exceptions.

```java
@Test
public void testThatComputerComputes() {
    ActorRef<Computer.Request> computer = KIT.spawn(Computer.create());
    TestProbe<Integer> probe = KIT.createTestProbe();

    computer.tell(new Compute(20));
    computer.tell(new Compute(30));
    computer.tell(new Forward(new GetTotal(probe.getRef())));

    assertEquals(8, (int) probe.receiveMessage());
}
```

The final test shows that the computer is stopped with the calculator
after division by zero.

```java
@Test
public void testThatComputerDiesWithCalculator() {
    ActorRef<Computer.Request> computer = KIT.spawn(Computer.create());
    TestProbe<Integer> probe = KIT.createTestProbe();

    computer.tell(new Compute(0));
    probe.expectTerminated(computer);
}
```

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
Check that the echo loop is still working
even after provoking crashs in the server or client.

