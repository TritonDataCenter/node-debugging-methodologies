# Investigating a potential memory leak

## Gathering basic information about the problem

When diagnosing a memory leak, make sure that it is possible that it can be
connected to such a problem:

  * Is the value of the RSS or the virtual memory size too high for the typical
  application workload?

  * Are these values growing constantly over a large period of time without ever
  plateauing or decreasing?

  * Does the application crash with an "out of memory error" message?

For every observable metric or event mentioned above, we need
to collect information that is as detailed as possible. For instance, we need
the full error message (including a stack trace, if there is any) if the
application crashes due to an out of memory error. If the application seems to
be using too much memory, we need the full output from tools like `prstat` or
`ps`.

## Ruling out the possibility of the application not leaking memory

Due to the dynamic nature of the Node.js runtime and the way it manages memory,
what can look as a memory leak might just be a deliberate choice from V8's
garbage collector to not collect garbage as aggressively as possible. There are
several indicators that can help to be more confident about either of these
possibilities.

An application is likely leaking memory if:

  * The amount of memory that the process is using, the RSS (resident set size)
  and/or the virtual size have been growing higher than the expected memory
  footprint without ever plateauing or shrinking for an extended period of time.

  * The application is crashing with an "out of memory" error when performing
  garbage collection.

An application might not be leaking memory if:

  * The amount of memory that the process is using, the RSS (resident set size)
  and/or the virtual size have been growing higher than the expected memory
  footprint without ever plateauing or shrinking for an extended period of time,
  but the command line options used to run the node program use V8-specific
  command line options to increase the size of the JavaScript heap, such as the
  following:

   * `--min_semi_space_size`
   * `--initial_old_space_size`
   * `--max_old_space_size`
   * `--max_semi_space_size`
   * `--target_semi_space_size`

  * The application is crashing with an "out of memory" error when not
  performing garbage collection. For instance, a common issue is for an API
  server to crash with an out of memory error when sending a JSON formatted
  response for a "list all objects" query that is not bounded, so that it
  generates a string object that is too large to fit in the space reserved for
  JavaScript objects.

There are more tools to gather a bit more information, the caveat being that
they may require restarting the application:

  * Using the `--max_old_space_size` command line option to put pressure on V8's
  garbage collector. If the expected memory footprint of an application is 512
  MBs of memory, then artificially limiting the space available for storing JS
  objects to 512 MBs will put pressure on V8's garbage collector and will make
  the application crash if it cannot collect enough garbage in order to make
  room for future allocations. This is a great way to confirm that memory
  footprint growth is not due to V8's garbage collector not being aggressive
  enough, but to an actual memory leak.

  * Using the `--trace-gc` command line option to collect information on what
  V8's garbage collector is doing. When used with the `--max_old_space_size`
  command line option mentioned above, this can provide even more evidence for
  determining whether we're in presence of an actual memory leak.

  * Logging the output of a call to `process.memoryUsage()`. This outputs three
  different values: RSS, amount of space used in the JavaScript heap, and total
  amount of space in the JavaScript heap. The second value (amount of space
  used in the JavaScript heap) cannot be obtained from a system tool and can
  help determine the memory growth pattern of the application.

## Gathering information from one core file

Keep in mind that at this point that it's possible that the application is
not leaking memory. What we'll do during this step is gather information from
one single core file. We'll use that information during the next step to
identify potentially suspicious patterns.

The information we'll gather is:

  * The process' information: environment variables, call stack(s), arguments,
  open file descriptors, memory mappings, etc.

  * libumem information for processes running on SmartOS.

  * JavaScript/V8 specific information: list of objects and functions.

### Getting general information about the process from the core file

Run the following commands:

```
$ pargs path-to-core-file
$ penv path-to-core-file
$ pstack path-to-core-file
$ pmap path-to-core-file
$ pfiles path-to-core-file
```

### Getting libumem information from the core file

If the program is running on SmartOS and was running with libumem's debugging
environment variables set, then we can use libumem's debugging features to get
some insights on the state of the memory usage of the process.

#### Check if libumem was used by the process

Simply run `penv path-to-core-file | grep PRELOAD` and make sure the
`LD_PRELOAD` environment variable includes `libumem.so`.

#### Collect libumem debugging info

If the process was running with libumem enabled, then start mdb:

```
$ mdb path-to-node-binary path-to-core-file
Loading modules: [ libumem.so.1 libc.so.1 ld.so.1 ]
> ::vmem
ADDR             NAME                        INUSE        TOTAL   SUCCEED  FAIL
fffffd7ffec9a680 sbrk_top               1234845696   1570119680    462069 18530
fffffd7ffec9b288   sbrk_heap            1234845696   1234845696    462069     0
fffffd7ffec9be90     vmem_internal        19795968     19795968      4643     0
fffffd7ffec9ca98       vmem_seg           18911232     18911232      4617     0
fffffd7ffec9d6a0       vmem_hash            844288       847872        22     0
fffffd7ffec9e2a8       vmem_vmem             46200        55344        15     0
0000000001a1f000     umem_internal         8647168      8650752      4531     0
0000000001a20000       umem_cache           268656       475136        58     0
0000000001a21000       umem_hash            410624       413696       289     0
0000000001a22000     umem_log                    0            0         0     0
0000000001a23000     umem_firewall_va            0            0         0     0
0000000001a24000       umem_firewall             0            0         0     0
0000000001a25000     umem_oversize               0            0      8400     0
0000000001a27000     umem_memalign               0            0         0     0
0000000001a42000     umem_default       1206398976   1206398976    444495     0
> ::umausers
::umausers
76382208 bytes for 1036 allocations with data size 73728:
         libumem.so.1`umem_cache_alloc_debug+0xfd
         libumem.so.1`umem_cache_alloc+0xb3
         libumem.so.1`umem_alloc+0x64
         libumem.so.1`umem_malloc+0x3f
         deflateInit2_+0x1e7
         node::ZCtx::Init+0x512
76382208 bytes for 1036 allocations with data size 73728:
         libumem.so.1`umem_cache_alloc_debug+0xfd
         libumem.so.1`umem_cache_alloc+0xb3
         libumem.so.1`umem_alloc+0x64
         libumem.so.1`umem_malloc+0x3f
         deflateInit2_+0x1b6
         node::ZCtx::Init+0x512
[...]
> ::findleaks
```

### Getting JavaScript runtime's information from the core file

Finally, we can get a lot of information from the Node.js runtime by using mdb's
`mdb_v8` dynamic module (or dmod). Load the application's core file with mdb if
it's not already loaded, and run the following commands:

```
$ mdb path-to-node-binary path-to-core-file
Loading modules: [ libc.so.1 ld.so.1 ]
> ::load v8
> ::jsstack
> ::findjsobjects ! sort -k 2 -n
> ::findjsobjects ! sort -k 3 -n
> ::jsfunctions ! sort -k 2 -n
```

## Identifying suspicious patterns from one core file

Now that all of this raw information was gathered, we can start looking at known
suspicious patterns to find a starting point for our investigation. The
following sub-sections describe the most common suspicious patterns that
indicate an actual memory leak.

### Unexpected large number of objects instances

The `::findjsobjects ! sort -k 2 -n` command lists object types and sorts them
by number of instances ascending. The objects types with the highest number of
instances are outputted last.

Any object type with a number of instances that is more than twice that of the
group of objects with the second largest number of instances is suspicious.
Here's an example of a suspicious number of instances:

```
> ::findjsobjects ! sort -k 2 -n | tail -20
      2072dc1d01    15250        4 WriteReq: chunk, encoding, callback, ...
      2072d1f6e9    15410        6 Object: x-request-id, content-type, ...
      2072d1ea11    17249        4 Window: id, isServer, recv, send
      2072d0b731    18217        3 BufferList: head, tail, length
      2072d05e19    18757        6 TransformState: afterTransform, ...
      2072d1f5c9    19195        4 Array
      2072d0f351    23740        1 Object: overflow
    1d597b360f89    29168        3 TickObject: callback, domain, args
      2072d137b9    33744       12 Side: domain, _events, _eventsCount, ...
      2072d128a9    53372       23 WritableState: objectMode, ...
      2072d05e81    53655        0 Uint8Array
      2072d0fc81    62065        3 CorkedRequest: next, entry, finish
      2072d06c19    63213        8 PriorityNode: tree, id, parent, weight, ...
      2072d61ec1    64986        4 Object: id, priority, fin, data
      2072d066e1    67035        2 Object: list, weight
      2072d05c31    78865       21 ReadableState: objectMode, ...
      2072d04a29   106305        1 Array
      2072d0e069   125344        2 Array
      2072d08f19   126603        0 Object
      2072d04341   408624        0 Array
```

In the output above, the last line indicates that there are `408624` instances
of `Array`, which is about four times larger than the second largest number of
instances, so looking at these objects should definitely be one of the first
things to do.

It's also important to consider the absolute numbers of instances compared to
what we'd expect the application's workload to be. In the output above, even the
first line, which represents the 20th largest number of instances (see the `tail
-20` in the command that produced the output), shows `15250` instances. This is
a very large number of instances if we're inspecting a core file from e.g an
HTTP server that we expect to handle around a few hundreds of concurrent
requests.

### Unexpected large number of properties for single objects

While large number of instances of the same object type is suspicious, large
number of properties can also be the sign of an actual memory leak.

The `::findjsobjects ! sort -k 3 -n` command lists object types and sorts them
by number of properties ascending. The objects types with the highest number of
properties are outputted last.

For instance, Node.js event emitters store their event listeners in an array
referenced by a property named `_events`. It is not uncommon for Node.js
programs to add a very large amount of the same event listener to the same event
emitter, making this array grow to a very large number of items. If these event
emitters objects are never cleaned up and/or these event listeners are never
removed, we are in the presence of a memory leak.

### Unexpected large number of functions instances

In JavaScript, functions are objects, and the same function can be instantiated
any number of time (it's only limited by how much memory can be used to store
these functions instances).

The `::jsfunctions ! sort -k 2 -n` command lists functions and sorts them by
number of instances ascending. The functions with the highest number of
instances are outputted last.

A large number of functions can be the sign of function instances not being
cleaned up. Since functions instances store a reference to their enclosing
enviroment (a closure), which can contain other objects, a function instance
that is not garbage collected can hold one or several references to objects, and
thus have a compounding effect.
