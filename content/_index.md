---
title: Parallelism and Concurrency in Java
---

# Parallelism and Concurrency in Java

Computers can do many things at the same time.
Operating systems run multiple programs simultaneously,
so we can listen to music while browsing the web.
A text editing program might search through one file
while we edit another.
Computers can also do the same thing on many processors.
Image manipulation software often uses multiple cores
when applying costly operations to large images.

Parallelism and concurrency are related but separate concepts.
Parallelism is about how a program is executed,
concurrency is about how a program is written.
In traditional programming languages like Java,
that support statements mutating memory explicitly,
how a program is written is often close to
how it is executed.
It is still useful to distinguish between these concepts.

Using parallelism means to execute a program 
using multiple processors or cores
(or, with hyper-threading, using multiple logical threads.)
Programs can be executed in parallel even if they are written 
without primitives for concurrent programming.
Different sub-expressions of an expression could be
evaluated on different cores if evaluation is costly.
In Java, stream pipelines can be processed sequentially
or in parallel with minimal changes to the underlying code.
Some programs do not require doing multiple things at the same time
but still benefit from parallel execution on many cores.

Other programs do require an explicit notion
of doing multiple things at the same time.
Such programs must be written with special support
from programming languages for managing
multiple so called threads of execution.
Different threads of a program can be regarded as seperate programs
accessing the same memory and files.
They can be (and usually are) coordinated,
for example to wait for and consume results
computed by other threads.
Usind and coordinating multiple threads of execution
in a program is called concurrent programming.

Concurrent programs can be executed using parallelism
if the executing computer has multiple cores.
They can also be executed sequentially
where a scheduler switches between different threads
usually in quick succession.
The order in which statements in different threads
are executed is generally unspecified.
Executing a concurrent program can have non-deterministic effects,
meaning that different runs may behave differently,
regardless of whether they are executed using parallelism or not.

In this tutorial, we will discuss different ways for
[using](docs/using/)
and 
[coordinating](docs/coordinating/)
threads in Java.
First, we will compare different ways of using threads
in a program for rendering images of the
[Mandelbrot fractal](https://en.wikipedia.org/wiki/Mandelbrot_set).
The rendering of such images is costly and can benefit from
parallel execution.
Later, we will discuss how to coordinate different threads
using locks in a concurrent algorithm for coloring grids.
Coordinating threads can be tricky and we will discuss
how to avoid blocking computations 
by different threads waiting for each other.

The underlying source code is available online, 
and this tutorial includes tasks to extend it.
If you want to follow along, you can use your own Java development environment
or install
[Docker](https://docs.docker.com/get-docker/)
and
[VS Code](https://code.visualstudio.com/download)
with the
[Remote Development Extension Pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)
to use a predefined environment without creating (or adjusting) your own.
To use the predefined environment in VS Code
download and unpack the
[source code](https://github.com/sebfisch/java-concurrency-code/archive/main.zip)
(or clone the
[repository](https://github.com/sebfisch/java-concurrency-code)
using `git`),
open the repository folder in VS Code,
click on the Remote Containers icon in the bottom-left corner,
and select *Reopen Folder in Container*.

