---
title: Parallel Streams
weight: 280
---

# Parallel Streams

[Java streams](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/stream/package-summary.html)
can be processed in parallel automatically.
The implementation of parallel streams is based on
work stealing with the common `ForkJoinPool`.

Our `StreamRenderer` is computing pixels using a stream.
Here is part of its implementation, again.

```java
box.pixels().forEach(pixel -> {
  // omitted
});
```

The following change turns this implementation
into a work stealing renderer 
using available processors efficiently.

```java
box.pixels().parallel().forEach(pixel -> {
  // omitted
});
```

The following picture has been created with the parallel version
of the `StreamRenderer`.

![Work stealing renderer with parallel stream](../stream.png)

The colors show that work is split differently
than with our custom implementation.
Nevertheless, many threads were involved in
rendering the bottom right corner of the picture.

