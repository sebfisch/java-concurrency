---
title: Completable Futures
weight: 460
---

# Completable Futures

Completable futures extend futures with numerous methods
to create more complex futures from simpler ones.
We will discuss selected parts of their API using tests.

## Creating futures

Completed futures can be created from the value they return.

```java
void testCreatingCompletedFuture() {
    final CompletableFuture<Integer> future = //
	CompletableFuture.completedFuture(42);
    assertEquals(42, future.join());
}
```

Here, the `join` method is like `get` but does not throw
an `InterruptedException`.
We can use `supplyAsync` to create a future
from an asynchronous computation.

```java
void testCompletingFutureAsynchronously() {
    final CompletableFuture<Integer> future = //
        CompletableFuture.supplyAsync(() -> 42);
    assertEquals(42, future.join());
} 
```

The computation is executed on the common `ForkJoinPool`
by default.
An alternative executor can be passed as second argument.
An even more manual way to create a completed future is
to call `complete` from an arbitrary thread.

```java
void testCompletingFutureManually() {
    final CompletableFuture<Integer> future = new CompletableFuture<>();
    Executors.newSingleThreadExecutor().execute(() -> future.complete(42));
    assertEquals(42, future.join());
}
```

## Manipulating futures

The interface `CompletionStage` implemented by completable futures
defines many operations for manipulating completable futures.
We only show two methods here, one that is like `map` on streams
and another that is like `flatMap`.

```java
void testThatThenApplyIsLikeMap() {
    final CompletableFuture<String> stringFuture = //
        CompletableFuture.completedFuture("hello");
    final CompletableFuture<Integer> intFuture = //
        stringFuture.thenApply(String::length);
    assertEquals(5, intFuture.join());
}

void testThatThenComposeIsLikeFlatMap() {
    final CompletableFuture<String> stringFuture = //
        CompletableFuture.completedFuture("ha");
    final CompletableFuture<String> combinedFuture = //
        stringFuture.thenCompose(string -> //
        CompletableFuture.completedFuture(string+string));
    assertEquals("haha", combinedFuture.join());
}
```

The second test could also be implemented using `thenApply`
rather than `thenCompose`.
We can see that just like `map` is a special case of `flatMap`
for streams (and optionals, too)
`thenApply` is a special case of `thenCompose`
for completable futures.

## Task: Handling exceptions

Write more tests for operations involving completable futures
that you find interesting.
Include tests that show operations involving exceptions
and how to handle them.
Can you express something like `filter` for streams
using `thenCompose` and exceptional futures?

