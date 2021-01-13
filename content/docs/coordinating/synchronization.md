---
title: Classic Synchronization
weight: 420
---

# Classic Synchronization

In this section we discuss traditional synchronization primitives
that have been provided in Java from the beginning.

## Thread Interference

Before discussing synchronization,
we look at what can happen if different threads lack coordination.
We will use a shared mutable object of the following class
to demonstrate
[thread interference](https://docs.oracle.com/javase/tutorial/essential/concurrency/interfere.html).

```java
class Count {
    public int count = 0;
}
```

The following test creates two threads
that manipulate a `Count` instance.

```java
void testThreadInterference() {
    final Count count = new Count();

    final Runnable inc = () -> {
        final int val = count.value;
        randomSleep();
        count.value = val + 1;
    };

    final Runnable dec = () -> {
        final int val = count.value;
        randomSleep();
        count.value = val - 1;
    };

    final IntSupplier race = () -> {
        final Thread incThread = new Thread(inc);
        final Thread decThread = new Thread(dec);
        incThread.start();
        decThread.start();
        try {
            incThread.join();
            decThread.join();
        } catch (InterruptedException e) {}
        return count.value;
    };

    assertTrue(IntStream.generate(race).limit(100).anyMatch(n -> n != 0));
}
```

The definitions of `inc` and `dec` 
to increment and decrement the shared `count` are only correct,
if the count is not changed by another thread during the `randomSleep`.
However, when executing the `race` many times,
it is highly likely that the count is manipulated by the other thread
at least in some executions.

## Intrinsic Locks

In Java, every object has a built-in lock.
We can modify the definitions of `inc` and `dec`
to use the `synchronized` keyword as follows.

```java
final Runnable inc = () -> {
    synchronized (count) {
        final int val = count.value;
        randomSleep();
        count.value = val + 1;
    }
};

final Runnable dec = () -> {
    synchronized (count) {
        final int val = count.value;
        randomSleep();
        count.value = val - 1;
    }
};
```

When executing the synchronized block,
no other thread can enter a block
synchronized on the same object.
The behaviour of the intrinsic lock is re-entrant,
meaning that a thread that is already holding a lock of an object
can enter other blocks synchronized on the same object.

Now, the assertion in the end can be rewritten
to demonstrate that no thread interferes with the other
between reading and writing the count value.

```java
assertTrue(IntStream.generate(race).limit(100).allMatch(n -> n == 0));
```

The `synchronized` keyword can also be used in method signatures,
meaning that the whole method definition is wrapped in a block
synchronized on the object the method is called on.

The `FractalApp` uses intrinsic locks 
to synchronize read and write access to a buffered image.
Interested readers are encouraged to look for the 
`synchronized` keyword in the `ImageCanvas` class.

## Releasing locks

Locks are released when a thread exits a synchronized block.
Alternatively, threads can release a lock they are holding
by calling `wait` on the corresponding object.
The `wait` method blocks until another thread notifies
waiting threads, usually with `notifyAll`.
The following test demonstrates how threads can wait for
and notify each other.

```java
void testIntrinsicLockRelease() {
    final Count count = new Count();

    final Runnable consume = () -> {
        synchronized (count) {
            while (count.value == 0) {
                try {
                    count.wait();
                } catch (InterruptedException e) {}
            }
            count.value = 0;
            count.notifyAll();
        }
    };

    final Runnable produce = () -> {
        synchronized (count) {
            while (count.value != 0) {
                try {
                    count.wait();
                } catch (InterruptedException e) {}
            }
            count.value = 10;
            count.notifyAll();
        }
    };

    final IntSupplier handshake = () -> {
        final Thread consumer = new Thread(consume);
        final Thread producer = new Thread(produce);
        consumer.start();
        producer.start();
        try {
            consumer.join();
            producer.join();
        } catch (InterruptedException e) {}
        return count.value;
    };

    assertTrue(IntStream.generate(handshake).limit(100).allMatch(n -> n == 0));
}
```

The consumer waits as long as the count value is zero,
and the producer waits as long as it is non-zero.
Regardless of which thread enters the synchronized block first,
the producer will be the first to exit its waiting loop,
write a non-zero count value,
and notify the consumer,
which will then continue to reset the count value to zero again.

Waiting threads should not assume 
that the condition they are waiting for
is satisfied when they are notified.
It is common practice to test it in a loop.

