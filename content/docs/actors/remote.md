---
title: Remote Actors
weight: 640
---

# Remote Actors

Akka supports communication with actors on remote machines.
Corresponding actor references can be used just like those for local actors.
From a programming point of view
the location of an actor is hardly visible
which helps develop distributed systems.

We will use a low level API provided by Akka to demonstrrate remote actors.
Although the documentation advises against its direct use,
more advanced APIs for distributed actors are out of scope for this tutorial.

## Configuration

To enable remote communication,
we need to configure Akka in the `application.conf` file.

```
akka {
  actor {
    provider = remote
    serialization-bindings {
        "sebfisch.actors.JsonSerializable" = jackson-json
    }
  }
  remote {
    artery {
      transport = tcp
    }
  }
}
```

Apart from selecting a provider that supports remote communication
and selecting a transport protocol,
we define a marker interface `JsonSerializable` for types of messages
that are exchanged via the remote mechanism
and register it for automatic conversion.

## Remote echos

We now extend the echo application to work with client and server
running in separate processes possibly on different computers.
The first step is to add a new type of message to the `EchoClient`.

```java
public static class SendRemote implements Request {
    public final String text;
    public final String path;
    
    public SendRemote(String text, String path) {
        this.text = text;
        this.path = path;
    }
}
```

This message wraps a text to be sent
and a path to the remote echo server.
Actors can be accessed via their path which is a URL with the following format.

```
akka://systemname@hostname:port/actorpath
```

The `actorpath` is nested according to the hierarchical structure of the actor system.
The root of the actor system has the path `/user`
because there is also another root `/system` used internally by Akka.

The `SendRemote` message is handled as follows.

```java
private EchoClient respond(SendRemote msg) {
    final int timeout = 10;
    final ActorContext<Request> ctx = getContext();

    ctx.getSystem().classicSystem().actorSelection(msg.path) //
        .resolveOne(Duration.ofSeconds(timeout)) //
        .thenApply(this::toTyped) //
        .thenAccept(server -> {
            ctx.getSelf().tell(new Send(msg.text, server));
        });
    
    return this;
}
```

We use `ActorSelection.resolveOne` 
to asynchronously retrieve an actor reference for the remote server
which we then use inside a `Send` message to the client itself.
The methods `classicSystem` and `toTyped` are used
to convert between the typed und untyped API 
because actor selection is only available in the untyped version.

## Implementing a remote echo server

The class `RemoteEchoServer` can be run
to start an `EchoServer` that is available for remote access.

```java
public class RemoteEchoServer {
    public static final String HOST = "127.0.0.1";
    public static final int PORT = 25520;
    public static final String NAME = "echo-server";
    public static final String PATH =
        "akka://" + NAME + "@" + HOST + ":" + PORT + "/user";

    private static final BufferedReader STDIN = 
        new BufferedReader(new InputStreamReader(System.in));

    public static void main(String[] args) {        
        ActorSystem<EchoServer.Request> echoServer =
            ActorSystem.create(EchoServer.create(), NAME, remoteConf());
        
        System.out.println("Type 'quit' to exit");
        try (Stream<String> lines = STDIN.lines()) {
            lines.filter("quit"::equals).findFirst();
        } finally {
            echoServer.terminate();
        }
    }

    private static Config remoteConf() {
        Map<String, Object> overrides = new HashMap<>();
        overrides.put("akka.remote.artery.canonical.hostname", HOST);
        overrides.put("akka.remote.artery.canonical.port", PORT);
        return ConfigFactory
            .parseMap(overrides)
            .withFallback(ConfigFactory.load());
    }
}
```

We override the default hostname and port with known values
and pass the resulting configuration to `ActorSystem.create`.
The program terminates when `quit` is read from standard input.

## Task: Implement a remote echo client

Complete the definition of `RemoteEchoClient`
which can be used as a counterpart for `RemoteEchoServer`.
When started, the application should start an actor
with the behavior `EchoClient` as root of the actor system.
Note that it is not necessary to override the default configuration
because echo clients do not need to be discoverable.
The application should read lines from standard input,
send them to the started `EchoClient` actor via the `SendRemote` message
using the path of the `RemoteEchoServer`,
and exit when the line contains `quit`.

Start `RemoteEchoServer` and `RemoteEchoClient`
to test if messages are sent back to the client.

