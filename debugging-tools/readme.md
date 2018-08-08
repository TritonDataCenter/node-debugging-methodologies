# Node.js Core Debugging Tools

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Node.js Core Debugging Tools](#nodejs-core-debugging-tools)
  - [Introduction](#introduction)
  - [Platform-specific considerations](#platform-specific-considerations)
  - [SmartOS debugging tools](#smartos-debugging-tools)
    - [Process state examination tools](#process-state-examination-tools)
      - [Proc tools](#proc-tools)
        - [pstack](#pstack)
        - [pmap](#pmap)
        - [pfiles](#pfiles)
        - [penv](#penv)
        - [pargs](#pargs)
    - [Tracing](#tracing)
      - [DTrace](#dtrace)
        - [DTrace's `ustack` and `jstack` functions](#dtraces-ustack-and-jstack-functions)
        - [Generating a core dump on a specific event](#generating-a-core-dump-on-a-specific-event)
        - [Generating CPU flamegraphs](#generating-cpu-flamegraphs)
        - [Static DTrace probes](#static-dtrace-probes)
        - [DTrace scripts](#dtrace-scripts)
    - [Post-mortem debugging](#post-mortem-debugging)
      - [mdb_v8](#mdb_v8)
    - [Live debugging](#live-debugging)
  - [Linux debugging tools](#linux-debugging-tools)
    - [Process state examination tools](#process-state-examination-tools-1)
    - [Tracing](#tracing-1)
    - [Post-mortem debugging](#post-mortem-debugging-1)
    - [Live debugging](#live-debugging-1)
  - [Multi-platform debugging tools](#multi-platform-debugging-tools)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

This page describes the tools that can be used to root cause Node.js problems,
such as:

* tracing tools like DTrace and perf.
* post-mortem debugging tools like mdb, mdb_v8 and llnode.
* live debugging tools like node-heapdump and llnode.

## Platform-specific considerations

A lot of the debugging tools that are available are specific to a given set of
platforms. This page is divided into three sections:

1. Debugging tools specific to the SmartOS platform.
2. Debugging tools specific to the Linux platform.
3. Debugging tools that can be considered as similar on both the Linux and
SmartOS platforms.

## SmartOS debugging tools

These debugging tools can be used unless specified otherwise for any Node.js
process running within any SmartOS container.

### Process state examination tools

#### Proc tools

SmartOS comes with several tools that allow to inspect the state of a running
process or the state represented by a core file generated from a process. As a
result, these tools fit both in the live and post-mortem debugging category.

These tools are usually referred to as "Proc tools", as they use the facilities
exposed by the process file system mounted in `/proc`.

In the remainder of this section, examples of proc tools will be given using
core files or running processes. Unless specified otherwise, the output of these
commands can be interpreted in the same way for both use cases.

##### `pstack`

Gets the callstack of a process. For Node.js processes, it will display
hexadecimal addresses in lieu of function names for frames that represent calls
to JavaScript functions. In order to get the actual names of JavaScript
functions, see [mdb_v8's documentation about the `::jsstack`
command](https://github.com/joyent/mdb_v8/blob/master/docs/usage.md#jsstack).

Following is a typical output when running pstack on a core file generated from
a process running node:

```
[root@0c933410-7c9b-4bea-96b6-2db35946bc95 ~]# pstack zd200714/core.49394.49394
core '/zd200714/core.49394.49394' of 49394:      node --nouse-idle-notification /server/router.j
-----------------  lwp# 1  --------------------------------
 fffffd7fff22b56a portfs   (6, 5, fffffd7fffdf99f0, 400, 1, fffffd7fffdf99d0)
 0000000000ba8064 uv__io_poll () + 144
 0000000000b98dae uv_run () + 15e
 00000000009e64f8 _ZN4node5StartEiPPc () + 588
 000000000086b36c _start () + 6c
-----------------  lwp# 2  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff2177e3 sema_wait (1a12500) + 13
 fffffd7fff20bfd8 sem_wait (1a12500) + 38
 0000000000ba51e8 uv_sem_wait () + 18
 0000000000871c42 _ZN4node21DebugSignalThreadMainEPv () + 12
 fffffd7fff224d9a _thrp_setup (fffffd7fff050240) + 8a
 fffffd7fff2250b0 _lwp_start ()
-----------------  lwp# 3  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff2177e3 sema_wait (1ab8e28) + 13
 fffffd7fff20bfd8 sem_wait (1ab8e28) + 38
 0000000000fef8d8 _ZN2v84base9Semaphore4WaitEv () + 18
 0000000000a34ef9 _ZN2v88platform9TaskQueue7GetNextEv () + 29
 0000000000a3504c _ZN2v88platform12WorkerThread3RunEv () + 2c
 0000000000ff0249 _ZN2v84baseL11ThreadEntryEPv () + 39
 fffffd7fff224d9a _thrp_setup (fffffd7fff050a40) + 8a
 fffffd7fff2250b0 _lwp_start ()
-----------------  lwp# 4  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff2177e3 sema_wait (1ab8e28) + 13
 fffffd7fff20bfd8 sem_wait (1ab8e28) + 38
 0000000000fef8d8 _ZN2v84base9Semaphore4WaitEv () + 18
 0000000000a34ef9 _ZN2v88platform9TaskQueue7GetNextEv () + 29
 0000000000a3504c _ZN2v88platform12WorkerThread3RunEv () + 2c
 0000000000ff0249 _ZN2v84baseL11ThreadEntryEPv () + 39
 fffffd7fff224d9a _thrp_setup (fffffd7fff051240) + 8a
 fffffd7fff2250b0 _lwp_start ()
-----------------  lwp# 5  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff2177e3 sema_wait (1ab8e28) + 13
 fffffd7fff20bfd8 sem_wait (1ab8e28) + 38
 0000000000fef8d8 _ZN2v84base9Semaphore4WaitEv () + 18
 0000000000a34ef9 _ZN2v88platform9TaskQueue7GetNextEv () + 29
 0000000000a3504c _ZN2v88platform12WorkerThread3RunEv () + 2c
 0000000000ff0249 _ZN2v84baseL11ThreadEntryEPv () + 39
 fffffd7fff224d9a _thrp_setup (fffffd7fff051a40) + 8a
 fffffd7fff2250b0 _lwp_start ()
-----------------  lwp# 6  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff2177e3 sema_wait (1ab8e28) + 13
 fffffd7fff20bfd8 sem_wait (1ab8e28) + 38
 0000000000fef8d8 _ZN2v84base9Semaphore4WaitEv () + 18
 0000000000a34ef9 _ZN2v88platform9TaskQueue7GetNextEv () + 29
 0000000000a3504c _ZN2v88platform12WorkerThread3RunEv () + 2c
 0000000000ff0249 _ZN2v84baseL11ThreadEntryEPv () + 39
 fffffd7fff224d9a _thrp_setup (fffffd7fff052240) + 8a
 fffffd7fff2250b0 _lwp_start ()
-----------------  lwp# 7  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff21e67b cond_wait_queue (1a16f40, 1a16f20, 0) + 5b
 fffffd7fff21eb21 cond_wait_common (1a16f40, 1a16f20, 0) + 1d1
 fffffd7fff21ecd2 __cond_wait (1a16f40, 1a16f20) + 52
 fffffd7fff21ed6a cond_wait (1a16f40, 1a16f20) + 2a
 fffffd7fff21eda5 pthread_cond_wait (1a16f40, 1a16f20) + 15
 0000000000ba5359 uv_cond_wait () + 9
 0000000000b96438 worker () + 48
 0000000000ba4e62 uv__thread_start () + 22
 fffffd7fff224d9a _thrp_setup (fffffd7fff052a40) + 8a
 fffffd7fff2250b0 _lwp_start ()
-----------------  lwp# 8  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff21e67b cond_wait_queue (1a16f40, 1a16f20, 0) + 5b
 fffffd7fff21eb21 cond_wait_common (1a16f40, 1a16f20, 0) + 1d1
 fffffd7fff21ecd2 __cond_wait (1a16f40, 1a16f20) + 52
 fffffd7fff21ed6a cond_wait (1a16f40, 1a16f20) + 2a
 fffffd7fff21eda5 pthread_cond_wait (1a16f40, 1a16f20) + 15
 0000000000ba5359 uv_cond_wait () + 9
 0000000000b96438 worker () + 48
 0000000000ba4e62 uv__thread_start () + 22
 fffffd7fff224d9a _thrp_setup (fffffd7fff053240) + 8a
 fffffd7fff2250b0 _lwp_start ()
-----------------  lwp# 9  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff21e67b cond_wait_queue (1a16f40, 1a16f20, 0) + 5b
 fffffd7fff21eb21 cond_wait_common (1a16f40, 1a16f20, 0) + 1d1
 fffffd7fff21ecd2 __cond_wait (1a16f40, 1a16f20) + 52
 fffffd7fff21ed6a cond_wait (1a16f40, 1a16f20) + 2a
 fffffd7fff21eda5 pthread_cond_wait (1a16f40, 1a16f20) + 15
 0000000000ba5359 uv_cond_wait () + 9
 0000000000b96438 worker () + 48
 0000000000ba4e62 uv__thread_start () + 22
 fffffd7fff224d9a _thrp_setup (fffffd7fff053a40) + 8a
 fffffd7fff2250b0 _lwp_start ()
-----------------  lwp# 10  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff21e67b cond_wait_queue (1a16f40, 1a16f20, 0) + 5b
 fffffd7fff21eb21 cond_wait_common (1a16f40, 1a16f20, 0) + 1d1
 fffffd7fff21ecd2 __cond_wait (1a16f40, 1a16f20) + 52
 fffffd7fff21ed6a cond_wait (1a16f40, 1a16f20) + 2a
 fffffd7fff21eda5 pthread_cond_wait (1a16f40, 1a16f20) + 15
 0000000000ba5359 uv_cond_wait () + 9
 0000000000b96438 worker () + 48
 0000000000ba4e62 uv__thread_start () + 22
 fffffd7fff224d9a _thrp_setup (fffffd7fff054240) + 8a
 fffffd7fff2250b0 _lwp_start ()
-----------------  lwp# 11  --------------------------------
 fffffd7fff2250f7 lwp_park (0, fffffd7ffd6ece70, 0)
 fffffd7fff21e67b cond_wait_queue (fffffd7ffeca4840, fffffd7ffeca4860, fffffd7ffd6ece70) + 5b
 fffffd7fff21eb21 cond_wait_common (fffffd7ffeca4840, fffffd7ffeca4860, fffffd7ffd6ece70) + 1d1
 fffffd7fff21ee2d __cond_timedwait (fffffd7ffeca4840, fffffd7ffeca4860, fffffd7ffd6ecf40) + 6d
 fffffd7fff21eef6 cond_timedwait (fffffd7ffeca4840, fffffd7ffeca4860, fffffd7ffd6ecf40) + 36
 fffffd7ffec487bc umem_update_thread (0) + 1cc
 fffffd7fff224d9a _thrp_setup (fffffd7fff054a40) + 8a
 fffffd7fff2250b0 _lwp_start ()
[root@0c933410-7c9b-4bea-96b6-2db35946bc95 ~]#
```

Even though Node.js is sometimes described as a single-threaded runtime, it is
not an accurate description. It runs the application's JavaScript code on one
single main thread, but its implementation uses several threads in order to
perform various tasks, including managing the asynchronous operations scheduled
by the application's code running on the main thread.

As a result, and because by default, `pstack` dispays the call stack for _all_
threads (or LWP for Light-Weight Process), the output above contains several
threads.

LWP or thread \#1 is the main thread, where the node, libuv and V8 event loops
run:

```
-----------------  lwp# 1  --------------------------------
 fffffd7fff22b56a portfs   (6, 5, fffffd7fffdf99f0, 400, 1, fffffd7fffdf99d0)
 0000000000ba8064 uv__io_poll () + 144
 0000000000b98dae uv_run () + 15e
 00000000009e64f8 _ZN4node5StartEiPPc () + 588
 000000000086b36c _start () + 6c
```

We can see that node's main thread was waiting on I/O using event ports. Unless
a node application is CPU bound (which a server application shouldn't
be), this is what a call stack for the main thread will often look like.

Thread \#2 is the thread used for the debugger agent, so that live debuggers can
connect to the node process in our to be able to send commands (in order to e.g
set breakpoings) and receive results:

```
-----------------  lwp# 2  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff2177e3 sema_wait (1a12500) + 13
 fffffd7fff20bfd8 sem_wait (1a12500) + 38
 0000000000ba51e8 uv_sem_wait () + 18
 0000000000871c42 _ZN4node21DebugSignalThreadMainEPv () + 12
 fffffd7fff224d9a _thrp_setup (fffffd7fff050240) + 8a
 fffffd7fff2250b0 _lwp_start ()
```

Threads \#3 to \#6 are V8's platform threads. This is a thread pool maitained by
V8 so that it can run the various tasks it needs to run, such as the tasks in
the microtask queues (e.g to execute promises):

```
-----------------  lwp# 3  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff2177e3 sema_wait (1ab8e28) + 13
 fffffd7fff20bfd8 sem_wait (1ab8e28) + 38
 0000000000fef8d8 _ZN2v84base9Semaphore4WaitEv () + 18
 0000000000a34ef9 _ZN2v88platform9TaskQueue7GetNextEv () + 29
 0000000000a3504c _ZN2v88platform12WorkerThread3RunEv () + 2c
 0000000000ff0249 _ZN2v84baseL11ThreadEntryEPv () + 39
 fffffd7fff224d9a _thrp_setup (fffffd7fff050a40) + 8a
 fffffd7fff2250b0 _lwp_start ()
-----------------  lwp# 4  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff2177e3 sema_wait (1ab8e28) + 13
 fffffd7fff20bfd8 sem_wait (1ab8e28) + 38
 0000000000fef8d8 _ZN2v84base9Semaphore4WaitEv () + 18
 0000000000a34ef9 _ZN2v88platform9TaskQueue7GetNextEv () + 29
 0000000000a3504c _ZN2v88platform12WorkerThread3RunEv () + 2c
 0000000000ff0249 _ZN2v84baseL11ThreadEntryEPv () + 39
 fffffd7fff224d9a _thrp_setup (fffffd7fff051240) + 8a
 fffffd7fff2250b0 _lwp_start ()
-----------------  lwp# 5  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff2177e3 sema_wait (1ab8e28) + 13
 fffffd7fff20bfd8 sem_wait (1ab8e28) + 38
 0000000000fef8d8 _ZN2v84base9Semaphore4WaitEv () + 18
 0000000000a34ef9 _ZN2v88platform9TaskQueue7GetNextEv () + 29
 0000000000a3504c _ZN2v88platform12WorkerThread3RunEv () + 2c
 0000000000ff0249 _ZN2v84baseL11ThreadEntryEPv () + 39
 fffffd7fff224d9a _thrp_setup (fffffd7fff051a40) + 8a
 fffffd7fff2250b0 _lwp_start ()
-----------------  lwp# 6  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff2177e3 sema_wait (1ab8e28) + 13
 fffffd7fff20bfd8 sem_wait (1ab8e28) + 38
 0000000000fef8d8 _ZN2v84base9Semaphore4WaitEv () + 18
 0000000000a34ef9 _ZN2v88platform9TaskQueue7GetNextEv () + 29
 0000000000a3504c _ZN2v88platform12WorkerThread3RunEv () + 2c
 0000000000ff0249 _ZN2v84baseL11ThreadEntryEPv () + 39
 fffffd7fff224d9a _thrp_setup (fffffd7fff052240) + 8a
 fffffd7fff2250b0 _lwp_start ()
```

Threads \#7 to \#10 represent libuv's thread pool:

```
-----------------  lwp# 7  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff21e67b cond_wait_queue (1a16f40, 1a16f20, 0) + 5b
 fffffd7fff21eb21 cond_wait_common (1a16f40, 1a16f20, 0) + 1d1
 fffffd7fff21ecd2 __cond_wait (1a16f40, 1a16f20) + 52
 fffffd7fff21ed6a cond_wait (1a16f40, 1a16f20) + 2a
 fffffd7fff21eda5 pthread_cond_wait (1a16f40, 1a16f20) + 15
 0000000000ba5359 uv_cond_wait () + 9
 0000000000b96438 worker () + 48
 0000000000ba4e62 uv__thread_start () + 22
 fffffd7fff224d9a _thrp_setup (fffffd7fff052a40) + 8a
 fffffd7fff2250b0 _lwp_start ()
-----------------  lwp# 8  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff21e67b cond_wait_queue (1a16f40, 1a16f20, 0) + 5b
 fffffd7fff21eb21 cond_wait_common (1a16f40, 1a16f20, 0) + 1d1
 fffffd7fff21ecd2 __cond_wait (1a16f40, 1a16f20) + 52
 fffffd7fff21ed6a cond_wait (1a16f40, 1a16f20) + 2a
 fffffd7fff21eda5 pthread_cond_wait (1a16f40, 1a16f20) + 15
 0000000000ba5359 uv_cond_wait () + 9
 0000000000b96438 worker () + 48
 0000000000ba4e62 uv__thread_start () + 22
 fffffd7fff224d9a _thrp_setup (fffffd7fff053240) + 8a
 fffffd7fff2250b0 _lwp_start ()
-----------------  lwp# 9  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff21e67b cond_wait_queue (1a16f40, 1a16f20, 0) + 5b
 fffffd7fff21eb21 cond_wait_common (1a16f40, 1a16f20, 0) + 1d1
 fffffd7fff21ecd2 __cond_wait (1a16f40, 1a16f20) + 52
 fffffd7fff21ed6a cond_wait (1a16f40, 1a16f20) + 2a
 fffffd7fff21eda5 pthread_cond_wait (1a16f40, 1a16f20) + 15
 0000000000ba5359 uv_cond_wait () + 9
 0000000000b96438 worker () + 48
 0000000000ba4e62 uv__thread_start () + 22
 fffffd7fff224d9a _thrp_setup (fffffd7fff053a40) + 8a
 fffffd7fff2250b0 _lwp_start ()
-----------------  lwp# 10  --------------------------------
 fffffd7fff2250f7 lwp_park (0, 0, 0)
 fffffd7fff21e67b cond_wait_queue (1a16f40, 1a16f20, 0) + 5b
 fffffd7fff21eb21 cond_wait_common (1a16f40, 1a16f20, 0) + 1d1
 fffffd7fff21ecd2 __cond_wait (1a16f40, 1a16f20) + 52
 fffffd7fff21ed6a cond_wait (1a16f40, 1a16f20) + 2a
 fffffd7fff21eda5 pthread_cond_wait (1a16f40, 1a16f20) + 15
 0000000000ba5359 uv_cond_wait () + 9
 0000000000b96438 worker () + 48
 0000000000ba4e62 uv__thread_start () + 22
 fffffd7fff224d9a _thrp_setup (fffffd7fff054240) + 8a
 fffffd7fff2250b0 _lwp_start ()
```

These threads are used by libuv to run concurrent asynchronous operations that
are not implemented in terms of async I/O primitives, such as file system
operations and DNS lookup requests.

Finally, thread \#11 is used for libumem's bookkeeping:

```
-----------------  lwp# 11  --------------------------------
 fffffd7fff2250f7 lwp_park (0, fffffd7ffd6ece70, 0)
 fffffd7fff21e67b cond_wait_queue (fffffd7ffeca4840, fffffd7ffeca4860, fffffd7ffd6ece70) + 5b
 fffffd7fff21eb21 cond_wait_common (fffffd7ffeca4840, fffffd7ffeca4860, fffffd7ffd6ece70) + 1d1
 fffffd7fff21ee2d __cond_timedwait (fffffd7ffeca4840, fffffd7ffeca4860, fffffd7ffd6ecf40) + 6d
 fffffd7fff21eef6 cond_timedwait (fffffd7ffeca4840, fffffd7ffeca4860, fffffd7ffd6ecf40) + 36
 fffffd7ffec487bc umem_update_thread (0) + 1cc
 fffffd7fff224d9a _thrp_setup (fffffd7fff054a40) + 8a
 fffffd7fff2250b0 _lwp_start ()
```

##### `pmap`

`pmap` is the abbreviation for `Process' Mappings`. It displays the memory
mappings that a process has mapped. What is generally interesting when looking
at process mappings in the context of a Node.js bug is to determine
the number and size of mappings (and thus the amount of memory used) for the
`heap` and `anon` types of mappings.

A `heap` mapping type represents a memory mapping used to store memory for the
native heap, or in other words, memory allocated with the native allocators used
by e.g `malloc` and `new`. For Node.js processes, these mappings are used by:

* allocation made by V8's, libuv and Node.js' runtimes.
* allocation made by native modules that use native allocators.

An `anon` mapping type represents an anonymous memory mapping created with
`mmap`. V8 uses anonymous mappings to allocate memory that is used to store
JavaScript objects. V8 calls the collection of these mappings a "heap", so it
can be confusing when others refer to a "heap" that is _not_ the _native_ heap
but a _custom V8 JavaScript heap_.

Given that, when looking at `pmap`'s output we can roughly determine the sources
of a node process' memory footprint. We can compute how much memory is used on
the native heap:

```
[root@0c933410-7c9b-4bea-96b6-2db35946bc95 ~]# pmap zd200714/core.49394.49394 | grep anon | tr -s ' ' | cut -d ' ' -f 2 | tr -d 'K' | awk '{s+=$1} END {print s}'
568312
```

and how much memory is used by V8's JavaScript heap:

```
[root@0c933410-7c9b-4bea-96b6-2db35946bc95 ~]# pmap zd200714/core.49394.49394 | grep heap | grep -v heapdump | tr -s ' ' | cut -d ' ' -f 2 | tr -d 'K' | awk '{s+=$1} END {print s}'
786432
```

From the output of the two `pmap` commands above, we can determine that ~586 MBs
of memory is used by V8's JavaScript heap, and ~786 MBs is used by the native
heap.

##### `pfiles`

`pfiles` can be used to list the file descriptors that are open by a given
process. File descriptors are identifiers that represent resources, so the number
of file descriptors that a process has open can be a good indication of what
resources are used.

Each file descriptor has a type, and so it can be easy to filter the file
descriptors by type to get an idea of e.g the number of file descriptors used
for network connections:

```
[root@0c933410-7c9b-4bea-96b6-2db35946bc95 ~]# pfiles zd200714/core.49394.49394 | grep '^[ ]*[0-9]\+\:' | grep IFSOCK | wc -l
1825
```

From the output above we can determine that this process had 1825 network socket
file descriptors opened when the core file was generated. If this number is too
large compared to the expected number of concurrent connections, this can be an
indicator of a problem.

##### `penv`

The `penv` command can be used to output the environment of a process. It's very
important to take a look at the process' environment before starting any
investigation, as a process' environment can have an impact on the problem
being investigated.

##### `pargs`

`pargs` displays the arguments of a process. Similarly to `penv`, it is very
important to take a look at a process' arguments before starting any
investigation, as command line arguments can have an impact on the observed
behavior of a process.

### Tracing

#### DTrace

DTrace is tracing framework that can be used to safely (unless safeties are
explicitly disabled) trace almost anything on a running system with mimimal
performance overhead.

Some DTrace tools are especially useful when tracing Node.js programs.

##### DTrace's `ustack` and `jstack` functions

Using the `stack` function, DTrace can generate a string that represents the
call stack at the time a probe fires. However, this function will not output the
name of JavaScript functions when tracing a node process.

In order to get JavaScript function names, one has to use the `ustack` or
`jstack` functions. See the documentation about these two functions in [Dtrace's
users guide](http://dtrace.org/guide/chp-actsub.html).

##### Generating a core dump on a specific event

Sometimes it can be very useful to generate a core dump when a specific event
happens. For instance, while the `stack`, `ustack` and `jstack` functions allow
for generating a call stack at the time a probe fires, inspecting the content of
e.g function calls arguments and what JavaScript objects they reference
currently requires a lot of work.

Instead, we could leverage the JavaScript tools provided by `mdb_v8` by
generating a core file when a probe fire and then get access to the whole
process, including JavaScript objects, using mdb and the tools provided by
mdb_v8.

In order to generate a core file from a running process, we'll need to do the following:

1. Stop the process.
2. Execute the `gcore` command.
3. Resume the process.

DTrace is designed to be safe to be used in production. Stopping a process is
not considered safe as this can potentially make the service that is provided by
this process unavailable. As a result, we'll need to enable "destructive
operations" with a command line switch. That translates to the following DTrace
script.

```
[root@0c933410-7c9b-4bea-96b6-2db35946bc95 ~]# cat gcore-on-open-file.d
#!/usr/sbin/dtrace -ws

syscall::open:entry
/execname == "node"/
{
    stop();
    system("gcore %d; prun %d", pid, pid);
}
```

Now runnning this script:
```
[root@0c933410-7c9b-4bea-96b6-2db35946bc95 ~]# ./gcore-on-open-file.d
dtrace: script './gcore-on-open-file.d' matched 1 probe
dtrace: allowing destructive actions
```

and opening a file in node's REPL in another shell on the same machine:

```
> [root@0c933410-7c9b-4bea-96b6-2db35946bc95 ~]# node
> fs.readFileSync('/tmp/bar', {}, function onRead(err) { console.log('error:', err); });
```

makes DTrace output the following:
```
CPU     ID                    FUNCTION:NAME
 31   6158                       open:entry gcore: core.36135 dumped

^C
```

which means that DTrace ran `gcore`, which generated a file named `core.36135`.
That core file can now be inspected using any of the proc tools or the
post-mortem debugging tools described in this document.

##### Generating CPU flamegraphs

CPU flame graphs were invented by Brendan Gregg, and are [documented in details
on his website](http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html).
Since their creation, several different tools have been written to generate CPU
flamegraphs for Node.js programs.

Typically, perf traces are generated by sampling user level stacks using DTrace,
and these stacks are turned into a SVG document that can be rendered in a web
browser by [stackvis](https://www.npmjs.com/package/stackvis).

Detailed instructions are [available on Node.js'
blog](https://nodejs.org/en/blog/uncategorized/profiling-node-js/).

##### Static DTrace probes

Node.js' core and several popular modules of the Node.js ecosystem make use of
DTrace static probes to provide useful tracing facilities.

Contrary to dynamic probes, static probes are useful because they provide a
stable interface that we can use to trace an application.

For instance, Node.js' core has a static probe named `http-client-request`. This
static probe's name does not change from one version of node to the next and
always fires when a HTTP request is received by a HTTP server. On the other
hand, a dynamic probe may not be present in all versions of the software that
provides it, and may fire under different circumstances across versions.

###### Node.js' core static probes

Node.js has a small number of static probes built-in. They can be listed with
the following command line:

```
dtrace -l -n 'node*:::*'
```

Using `c++filt` can be useful to demangle C++ function names and get an output that is easier to read:

```
[root@0c933410-7c9b-4bea-96b6-2db35946bc95 ~]# dtrace -l -n 'node*:::*' | c++filt
   ID   PROVIDER            MODULE                          FUNCTION NAME
 2814  node86755              node node::dtrace_gc_done(v8::Isolate*, v8::GCType, v8::GCCallbackFlags) gc-done
 4888  node86755              node node::dtrace_gc_start(v8::Isolate*, v8::GCType, v8::GCCallbackFlags) gc-start
13057  node86755              node node::DTRACE_HTTP_CLIENT_REQUEST(v8::FunctionCallbackInfo<v8::Value> const&) http-client-request
13903  node86755              node node::DTRACE_HTTP_CLIENT_RESPONSE(v8::FunctionCallbackInfo<v8::Value> const&) http-client-response
13960  node86755              node node::DTRACE_HTTP_SERVER_REQUEST(v8::FunctionCallbackInfo<v8::Value> const&) http-server-request
14046  node86755              node node::DTRACE_HTTP_SERVER_RESPONSE(v8::FunctionCallbackInfo<v8::Value> const&) http-server-response
81707  node86755              node node::DTRACE_NET_SERVER_CONNECTION(v8::FunctionCallbackInfo<v8::Value> const&) net-server-connection
81798  node86755              node node::DTRACE_NET_STREAM_END(v8::FunctionCallbackInfo<v8::Value> const&) net-stream-end
```

Static probes, like dynamic probes, have parameters that can be used in a DTrace
script. In order to display the list of parameters available, simply use the
`-v` DTrace command line option when listing probes:

```
[root@0c933410-7c9b-4bea-96b6-2db35946bc95 ~]# dtrace -lv -n 'node*:::*http*' | c++filt
   ID   PROVIDER            MODULE                          FUNCTION NAME
13057  node86755              node node::DTRACE_HTTP_CLIENT_REQUEST(v8::FunctionCallbackInfo<v8::Value> const&) http-client-request

        Probe Description Attributes
                Identifier Names: Private
                Data Semantics:   Private
                Dependency Class: Unknown

        Argument Attributes
                Identifier Names: Evolving
                Data Semantics:   Evolving
                Dependency Class: ISA

        Argument Types
                args[0]: node_http_request_t *
                args[1]: node_connection_t *
                args[2]: string
                args[3]: int
                args[4]: string
                args[5]: string
                args[6]: int

13903  node86755              node node::DTRACE_HTTP_CLIENT_RESPONSE(v8::FunctionCallbackInfo<v8::Value> const&) http-client-response

        Probe Description Attributes
                Identifier Names: Private
                Data Semantics:   Private
                Dependency Class: Unknown

        Argument Attributes
                Identifier Names: Evolving
                Data Semantics:   Evolving
                Dependency Class: ISA

        Argument Types
                args[0]: node_connection_t *
                args[1]: string
                args[2]: int
                args[3]: int

13960  node86755              node node::DTRACE_HTTP_SERVER_REQUEST(v8::FunctionCallbackInfo<v8::Value> const&) http-server-request

        Probe Description Attributes
                Identifier Names: Private
                Data Semantics:   Private
                Dependency Class: Unknown

        Argument Attributes
                Identifier Names: Evolving
                Data Semantics:   Evolving
                Dependency Class: ISA

        Argument Types
                args[0]: node_http_request_t *
                args[1]: node_connection_t *
                args[2]: string
                args[3]: int
                args[4]: string
                args[5]: string
                args[6]: int

14046  node86755              node node::DTRACE_HTTP_SERVER_RESPONSE(v8::FunctionCallbackInfo<v8::Value> const&) http-server-response

        Probe Description Attributes
                Identifier Names: Private
                Data Semantics:   Private
                Dependency Class: Unknown

        Argument Attributes
                Identifier Names: Evolving
                Data Semantics:   Evolving
                Dependency Class: ISA

        Argument Types
                args[0]: node_connection_t *
                args[1]: string
                args[2]: int
                args[3]: int
```

From the output above, we can determine that the `http-client-request` DTrace
static has seven arguments:

```
 Argument Types
                args[0]: node_http_request_t *
                args[1]: node_connection_t *
                args[2]: string
                args[3]: int
                args[4]: string
                args[5]: string
                args[6]: int
```

When displaying probes' arguments, DTrace only includes their type. From that
information alone, it can sometimes be difficult to infer their semantics. The
best way to know the semantics of argument types is to take a look at the source
code where the associated probe is fired.

###### Bunyan probes

Bunyan uses static probes to provide the ability to dynamically change the level
of the information that is logged. See [more details in Bunyan's documentation](
https://github.com/trentm/node-bunyan#runtime-log-snooping-via-dtrace).

###### Restify probes

Restify also leverages static probes to provide additional insights into the
requests/responses cycle and the way Restify handles them. See [Restify's
tracing documentation for more details](http://restify.com/#dtrace).

##### DTrace scripts

DTrace scripts that leverage these static probes already exists, and they have
the advantage of not requiring to know all the details about probes' arguments'
types. As such, they require a lot less work to start tracing events.

###### nhttpsnoop

nhttpsnoop leverages Node.js' core HTTP static probes to trace
requests/responses cycles. It includes information about request/reponses
parameters/results, latency, etc.

See https://github.com/joyent/nhttpsnoop for more details.

### Post-mortem debugging

#### mdb_v8

See [mdb_v8's documentation on
GitHub](https://github.com/joyent/mdb_v8/blob/master/docs/usage.md).

### Live debugging

## Linux debugging tools

### Process state examination tools

### Tracing

### Post-mortem debugging

### Live debugging

## Multi-platform debugging tools
