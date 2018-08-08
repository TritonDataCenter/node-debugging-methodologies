# Investigating a performance issue

By "performance" we usually mean latency and/or throughput of a service provided
by a program to another program. Performance related issues can be some of the
most difficult issues to investigate.

The performance of a Node program can be directly impacted by a lot of different
factors, both internal and external to the program, including the state of
hardware and external systems. As a result, it is very challenging to come up
with a step by step guide that will guarantee to help get closer to the root
cause of the problem.

Therefore, this document aims at describing a process that is very general, and
whose only goal is to start the investigation on a track that is hopefully not
misleading.

## Broad system analysis

The first step when investigating a performance issue is to get an
overview of the architecture of the application, document it, and identify which
individual components contribute the most to its overall performance.

### Formulate the problem statement

For performance related issues, it is very important to come up with a
problem statement that has three key characteristics. It must be measurable,
have a goal, and be scoped to the smallest functional dimension of the system
under investigation.

Having a problem statement that is measurable means that performance
improvements can be measured and progress can be tracked. Having a goal allows
us to determine when the problem is solved. Finally, having it scoped to the
smallest functional dimension of the system under investigation prevents the
task from being too large.

For instance, the following problem statements do not satisfy these three
characteristics:

1. "My API server has a high latency, how can I make it have lower latency?"

2. "My websocket client is too slow, how can I make it perform better?"

The following equivalent statements satisfy them:

1. "The `/create` endpoint of my API server has a latency of 1 second at
the 99% percentile, but my SLA is .5 seconds at the 99.99% percentile, how can
I get closer to this SLA?"

2. "My websocket client is able to process 100 subscription events at the 99%
percentile, but I need it to proces 1,000 events at the same percentile, how can
I make it perform better?"

### Documenting the architecture of the system under investigation

The design of an application has a direct impact on its performance. For
instance, if a given API server's endpoints sends multiple requests to other
hosts on the network to generate a response, its performance will be affected by
a variety of factors including networking performance and the remote systems'
performance.

Documenting the architecture (invidivual components, how they communicate
between them, etc.) is thus a key step in determining an appropriate
investigation strategy.

### Identifying hot individual components

From the application's architecture's documentation, we can determine which
components of the system are in the critical path to provide the service under
investigation, and how each of them contribute to the overall performance. Once
the largest contributors are identified, we can investigate the performance of
each individual component separately.

## Single component analysis

By now, we should understand better what individual components of the
application contribute the most to its overall performance, and it's time to
perform a single component analysis on each of them.

For each component, we need to identify:

1. Whether its performance is CPU and/or I/O bound.
2. What causes it to spend too much time using the CPU or performing I/O.

### Identifying whether the workload is I/O or CPU bound

#### Determining CPU usage for a Node.js process

Running the following command:

```
$ prstat -Lmc -p pid
```

will continuously output process statistics for the process with PID `pid`, one
page after another. For a Node.js process, the output for a single page will
look like the following:

```
 PID USERNAME USR SYS TRP TFL DFL LCK SLP LAT VCX ICX SCL SIG PROCESS/LWPID 
 96006 root     3.4 2.8 0.0 0.0 0.0 0.0  94 0.0  11  17 663   0 node/1
 96006 root     0.0 0.0 0.0 0.0 0.0 100 0.0 0.0  11   0  31   0 node/10
 96006 root     0.0 0.0 0.0 0.0 0.0 100 0.0 0.0   3   0   5   0 node/3
 96006 root     0.0 0.0 0.0 0.0 0.0 100 0.0 0.0   2   0   4   0 node/4
 96006 root     0.0 0.0 0.0 0.0 0.0 100 0.0 0.0   3   0   5   0 node/5
 96006 root     0.0 0.0 0.0 0.0 0.0 100 0.0 0.0   2   0   4   0 node/6
 96006 root     0.0 0.0 0.0 0.0 0.0 100 0.0 0.0   1   0   4   0 node/2
 96006 root     0.0 0.0 0.0 0.0 0.0 100 0.0 0.0   2   0   4   0 node/9
 96006 root     0.0 0.0 0.0 0.0 0.0 100 0.0 0.0   1   0   3   0 node/8
 96006 root     0.0 0.0 0.0 0.0 0.0 100 0.0 0.0   1   0   3   0 node/7
Total: 1 processes, 10 lwps, load averages: 0.02, 0.07, 0.04
```

To determine CPU usage, we can take a look at columns named `USR` and `SYS`.
Values in these two columns are between 0 and 100. If either or both these
values are consistently significant (above `10`), then the corresponding thread
spends a significant amount of time on CPU.

The `VCX` and `ICX` columns respectively indicate voluntary and involuntary
context switches, and thus are good indicators of whether a process/thread is
CPU bound. If a given thread has a significant number of involuntary context
switches, it is likely CPU bound.

Note that in the output above, the thread identified by `node/1` is the main
execution thread where the application's JavaScript code runs. All other threads
are part of the following groups of threads:

* Libuv's thread pool. For a Node.js program, these threads are responsible for
  running filesystem I/O operations and asynchronous `getaddrinfo` calls.
* V8's platform's thread pool. These threads are responsible for running various
  internal components of the V8 runtime, such as the runtime profiler.

### Gathering data about time spent on CPU

#### Generating a CPU flamegraph

While `prstat`'s output allows to identify a process spending a lot of time on
CPU, it doesn't help identifying what part of the program is responsible for
most CPU usage.

[Generating a CPU flamegraph](../debugging-tools#generating-cpu-flamegraphs)
uses DTrace's sampling profiler and the Node.js ustack helper to identify which
part of the application's code are the biggest contributors to time spent on
CPU.

When generating a CPU flamegraph, make sure that the Node.js program that is
being profiled is under a load that is representative of the application's load.

#### Common issues with Node.js applications

##### Garbage collection pauses

V8's garbage collector can impact JavaScript code's performance. Thus,
__when CPU usage is high, we must make sure that it is not a symptom of a
memory leak__.

One way to do that is to look for call stacks that are related to V8's garbage
collector in a CPU flamegraph. Another way, which unfortunately requires to
change the way the program runs and thus a restart, is to run the program with
the `--trace-gc` command line option. This command line option makes V8 output
debug informatiom about the work performed by the garbage collector, including
how much time was spent.

### Gathering data about time spent doing I/O

#### Tracing I/O related system calls

#### Tracing HTTP requests/responses latencies

