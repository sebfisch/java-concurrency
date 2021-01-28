---
title: Actor Systems
weight: 620
---

# Actor Systems

We now look at how actors can be started outside of tests.
All actors started in a program form a hierarchy.
The root of this hierarchy is a behavior started by an actor system.
Typically, there is only one actor system per application.

## Echoing messages

As a simple example of an application with multiple actors,
we write a program where one actor echos messages back to another.
The messages contain lines read from standard input,
and the sender prints messages that bounced back from the other actor.

Here is an example interaction with our program.
Every second line is input, the rest is output.

```
Type 'quit' to exit, something else to send
hi
received: hi
ok
received: ok
quit
```

Here is the first part of the implementation 
with the main method creating the actor system.

```java
public class EchoLoop extends AbstractBehavior<Echo> {
    private static final BufferedReader STDIN = 
        new BufferedReader(new InputStreamReader(System.in));

    public static void main(String[] args) {
        ActorSystem<Echo> echoLoop = //
            ActorSystem.create(EchoLoop.create(), "echo-loop");
        
        System.out.println("Type 'quit' to exit, something else to send");
        try(Stream<String> lines = STDIN.lines()) {
            lines
                .takeWhile(Predicate.not("quit"::equals))
                .forEach(line -> echoLoop.tell(new Echo(line)));
        } finally {
            echoLoop.terminate();
        }
    }
    // to be continued
}
```

The program uses `ActorSystem.create` with an `EchoLoop` behavior
(continued below)
to start all used actors.
Every line read from standard input is sent to the root actor
wrapped inside an `Echo` message.
When the user types `quit`, we shut down the actor system using `terminate`.
Otherwise, the program would continue to run because the actors would still run.

The `Echo` message, defined as a nested class, wraps text.

```java
public static class Echo {
    public final String text;
    
    public Echo(String text) {
        this.text = text;
    }
}
```

The `EchoLoop` behavior stores references to other actors.

```java
public static Behavior<Echo> create() {
    return Behaviors.setup(EchoLoop::new);
}

private ActorRef<EchoServer.Request> server;
private ActorRef<EchoClient.Request> client;

private EchoLoop(ActorContext<Echo> ctx) {
    super(ctx);
    server = ctx.spawn(EchoServer.create(), "echo-server");
    client = ctx.spawn(EchoClient.create(), "echo-client");
}
```

In the constructor we use the provided actor context
to `spawn` the other actors.
When receiving an `Echo` message,
the wrapped text is forwarded to the client using its `Send` message.
Apart from the text, the `Send` message also contains
a reference to the actor the text should be sent to.

```java
@Override
public Receive<Echo> createReceive() {
    return newReceiveBuilder()
        .onAnyMessage(this::echo)
        .build();
}

private EchoLoop echo(Echo msg) {
    client.tell(new EchoClient.Send(msg.text, server));
    return this;
}
```

## Implementing the echo client

The definition of the echo client starts with the messages it can receive.

```java
public class EchoClient extends AbstractBehavior<Request> {

    public interface Request {}

    public static class Send implements Request {
        public final String text;
        public final ActorRef<EchoServer.Request> server;

        public Send(String text, ActorRef<EchoServer.Request> server) {
            this.text = text;
            this.server = server;
        }
    }

    public static class Log implements Request {
        public final EchoServer.Response response;

        public Log(EchoServer.Response response) {
            this.response = response;
        }
    }
    // to be continued
}
```

All possible message implement the empty interface `Request`
which we use as type parameter for the message type.
We have already seen that the `Send` message wraps text
and a reference to a server actor.
The `Log` message wraps a server response.

As message handlers we use method references to an overloaded respond method.
The builder method `onMessage` is used to restrict the handler
to specific message types.

```java
@Override
public Receive<Request> createReceive() {
    return newReceiveBuilder()
        .onMessage(Send.class, this::respond)
        .onMessage(Log.class, this::respond)
        .build();
}
```

Clients respond to `Log` messages by printing the wrapped text.

```java
private EchoClient respond(Log msg) {
    System.out.printf("received: %s%n", msg.response.text);
    return this;
}
```

The text wrapped in a `Send` message is forwarded
to the wrapped server actor using its `Request` message.
Like `Send`, the `EchoServer.Request` message also wraps
an actor reference in addition to the text.
Both messages are equivalent,
but we use different types for demonstration purposes.

```java
private EchoClient respond(Send msg) {
    final ActorRef<EchoServer.Response> logAdapter =
        getContext().messageAdapter(EchoServer.Response.class, Log::new);
    msg.server.tell(new EchoServer.Request(msg.text, logAdapter));
    return this;
}
```

The echo client does not handle server requests directly,
so it cannot pass a reference to itself inside the request to the server.
Server requests need to be wrapped inside the `Log` message
for the client to be able to handle them.
The `messageAdapter` method on the actor context handles the wrapping
and returns an actor reference of appropriate type.

The following test demonstrates the behavior of echo clients sending a message.

```java
@Test
public void testThatEchoClientSendsGivenMessage() {
    ActorRef<EchoClient.Request> client = KIT.spawn(EchoClient.create());
    
    String text = "hello world";
    TestProbe<EchoServer.Request> server = KIT.createTestProbe();
    client.tell(new EchoClient.Send(text, server.getRef()));

    EchoServer.Request request = server.receiveMessage();
    assertEquals(text, request.text);
}
```

## Task: Implement the echo server

The class `EchoServer` defines `Request` and `Response` types
for actors that echo sent text back to their sender.
Implement the corresponding behavior
receiving `Request` messages and sending back a `Response` with the same text
before waiting for new messages.

Add a test to the `EchoTests` class
demonstrating that the server behaves as intended.

