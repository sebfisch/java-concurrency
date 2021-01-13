---
title: Throttling Tasks
weight: 480
---

# Throttling Tasks

Finally, we look at a class used by the grid coloring application
that is useful in other contexts too.
To implement the animation,
the coloring algorithm notifies the image canvas
to rerender the image when colors in the grid change.
However, when too many notifications arrive in a short time
the user interface thread becomes unresponsive.
To avoid overloading the user interface thread with notifications
we use a class for throttling tasks represented as `Runnable` instances.
Throttling is implemented by delaying executed tasks and ignoring
subsequent executions until the delay is over.

The `ColoringApp` uses the following line 
to register a throttled rendering task
to be executed by the coloring algorithm.

```java
coloring.onChange(new Throttled(canvas::rerender, 200, TimeUnit.MILLISECONDS));
```

The `rerender` method does not actually perform the rendering.
It only signals to the user interface thread
that rerendering is necessary.

The `Throttled` class stores the arguments passed in the costructor
and has a `run` method that is implemented as follows.

```java
public synchronized void run() {
    if (!delayed) {
        delayed = true;
        service.schedule(() -> {
            command.run();
            delayed = false;
        }, delay, unit);
    }
}
```
The `run` method is synchronized to prevent different threads
from executing it at the same time.
The `delayed` flag signifies whether the current invocation
should be ignored because a previous invocation
has already been delayed.
If the current invocation is not ignored
it is delayed according to the delay passed in the constructor.
The `service` used to execute the delayed task
is an instance of `ScheduledExecutionService`
initialized in the constructor.

