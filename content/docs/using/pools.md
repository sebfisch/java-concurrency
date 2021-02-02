---
title: Thread Pools
weight: 240
---

# Thread Pools

The program discussed in the previous section
might start more threads than processor cores are available.
It is often more efficient to create threads corresponding to
available cores in advance and reuse them.
So called thread pools are used to implement this approach.

Before implementing a renderer using a thread pool
we write tests to learn about their API.

## Testing the API

The following test creates a thread pool 
and submits a task for execution.

```java
void testSubmittingToAThreadPool() {
    final int threadCount = Runtime.getRuntime().availableProcessors();
    final ExecutorService pool = Executors.newFixedThreadPool(threadCount);

    final Future<Integer> future = pool.submit(() -> 42);

    try {
        assertEquals(42, future.get());
    } catch (InterruptedException | ExecutionException e) {}

    pool.shutdown();
}
```

The number of available processors is queried
and then used to create a thread pool of fixed size.
The `submit` method is used to pass an anonymous
instance of
[Callable](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/Callable.html)
to be executed by a thread in the pool.
The result of `submit` is an instance of
[Future](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/Future.html)
that can be used to check if the task is completed using `isDone`
and to query the task result using `get`.

The `get` method blocks until the task is completed
similar to the `join` method for threads.
It throws an `InterruptedException` if the thread
executing the task is interrupted
and an
[ExecutionException](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/ExecutionException.html)
if the task throws an exception.
There is another variant of `get` 
that accepts a time limit as argument 
and throws a
[TimeoutException](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/TimeoutException.html)
if the task does not finish before the given limit.

Thread pools can be terminated with the `shutdown` method.
After calling `shutdown`, the pool does not accept
more tasks to be submitted
(throwing a
[RejectedExecutionException](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/RejectedExecutionException.html)
instead)
but continues to execute running tasks.
There is a variant `shutdownNow` that attempts
to stop executing tasks and returns a list of waiting tasks.

The next test demonstrates how to cancel futures.

```java
void testCancellingAFuture() {
    final ExecutorService pool = Executors.newSingleThreadExecutor();
    final List<?> interruptions = new ArrayList<>();
    final Future<?> future = pool.submit(() -> {
        while (!Thread.interrupted()) {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                interruptions.add(null);
                return;
            }
        }
        interruptions.add(null);
    });

    try {
        TimeUnit.MILLISECONDS.sleep(100);
    } catch (InterruptedException e) {}

    future.cancel(true);
    assertTrue(future.isCancelled()); 

    boolean cancelled = false;
    try {
        future.get();
    } catch (InterruptedException | ExecutionException e) {
    } catch (CancellationException e) {
        cancelled = true;
    }

    assertTrue(cancelled);
    assertEquals(1, interruptions.size());
}
```

Just to demonstrate the use of a different executor,
we create a pool with a single thread.
This time we pass a task that does not return a result
as anonymous `Runnable` instance to `submit`.

After submitting the future,
we give the pool some time to start executing it
before calling `cancel` on the returned future.
The `cancel` method accepts a boolean argument.
If it is `true`, the thread executing the task is interrupted.
Calling `get` on a cancelled future
results in a
[CancellationException](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/CancellationException.html).

## Task: Write more tests

Study the documentation for
[ExecutorService](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/ExecutorService.html)
and
[Future](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/Future.html).
Write more tests that you find interesting.
Alternatively, write tests documenting the behaviour
of
[ScheduledExecutorService](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/ScheduledExecutorService.html)s.
The class
[Executors](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/Executors.html)
has static methods to create them.

## Rendering with a thread pool

We are now ready to implement a renderer
that reuses threads when rendering an image
with more parts than available processors.
Here is the `render` method
of the `ThreadPoolRenderer`.

```java
public boolean render(final Box box) {
    final InterruptFlag self = new InterruptFlag();
    return join(fork(ForkJoinPool.commonPool(), box, self), self);
}
```

We use `ForkJoinPool.commonPool` to access a thread pool
that is configured to run many non-blocking tasks
optimized for the available processors.
We also instantiate an `InterruptFlag` (discussed below)
and pass it to the `fork` and `join` methods.
Here is the implementation of `fork`.

```java
private List<Future<?>> fork(final ExecutorService pool, //
        final Box box, final Interruptible origin) {
    return box.split() //
        .map(part -> (Runnable) () -> renderer.render(part, origin)) //
        .map(pool::submit) //
        .collect(Collectors.toList());
}
```

It returns a list of futures
corresponding to rendering tasks
using an underlying `StreamRenderer`
that is passed a reference to the interrupt flag
to be tested for interruptions.

The `join` method waits for the tasks to complete
by calling `get` on the corresponding futures.

```java
private boolean join(final List<Future<?>> futures, //
        final InterruptFlag self) {
    return futures.stream().map(future -> {
        try {
            future.get();
            return true;
        } catch (InterruptedException | ExecutionException e) {
            self.wasInterrupted(true);
            return false;
        }
    }).allMatch(ok -> ok);
}
```

The result of `join` signifies
whether the calling thread waited successfully for all tasks
without being interrupted.
If the calling thread is interrupted,
it changes the interrupt flag passed as second argument.
We use our own interrupt flag,
because the one associated with threads in Java might be reset,
which could lead to started threads not being able to notice the interruption.

The following picture has been created using the `ThreadPoolRenderer`.

![Part of the Mandelbrot fractal](../pool.png)

This time parts colored with the same hue
have been rendererd by the same thread.

The picture has been created by zooming in
on some part of the Mandelbrot fractal.
Rendering is more costly when zoomed in,
and bright parts take significantly 
longer to compute than dark parts.
When observing the rendering process closely,
we see that the bottom right part of the picture
is completed last.
In the end, only a few threads are occupied
rendering the remaining parts of the image
even if more processors would be available.

