---
title: Using Threads
weight: 200
---

# Using Threads

In this part we discuss different ways of using threads in Java.
Before we inspect a more practical application of each approach,
we write tests to learn about crucial parts of the API.
We will also discuss how to wait for threads to complete
as well as asking them to complete using interruptions.

We start with the 
[classic interface](classic)
for using threads,
then learn how to reuse threads in
[thread pools](pools),
before learning how to decompose computations recursively
and distribute created tasks with a 
[work stealing](work_stealing)
algorithm using fork join pools.
Finally, we learn how to use
[parallel streams](parallel_streams)
which are using a fork join pool with work stealing internally.

