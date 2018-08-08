# Investigating a Node.js process that crashed

## Introduction

A "crash" is a term commonly used to describe a process that exited for any
other reason than a voluntary successful exit. This description covers a lot of
different use cases, including when:

* The operating system sends a signal to the process because it performed an
invalid operation. Examples include dereferencing a `NULL` pointer, or any other programming error that is not recoverable.

* The process voluntarily crashes due to a failed assertion. The Node.js runtime
uses a lot of assertions to make sure it crashes early when it identifies an
inconsistent state.

* Another process sends a signal that makes the process exit, such as sending
`SIGTERM`, `SIGKILL`, `SIGABRT`, etc.

When investigating a crash, be mindful of all the circumstances that can lead to
a process exiting with an exit code representing an error condition.

## Gathering information from the process' output

Depending on what component of the program is making the process exit, some
content may be output on the console before the process exits. Even if that
content often does not describe the root cause of the problem, it usually
provides a starting point and can be used to identify previous similar problems.
Thus, it is very important to get access to the output of the process right
_before_ the process exited.

## Gathering information from one core file

While the process' output before it exited usually gives us only a small bit of
context, a core file allows us to inspect the entire state of the process at the
time it exited.

As a result, unless we can identify a problem that was previously root-caused
only using the process' output, we won't be able to root cause a crash without a
core file.

### Getting general information about the process from the core file

Run the following commands:

```
$ pargs path-to-core-file
$ penv path-to-core-file
$ pstack path-to-core-file
$ pmap path-to-core-file
$ pfiles path-to-core-file
```

## Getting JavaScript runtime's information from the core file

In the last step, we were able to extract the call stack at the time of the
crash but the names of the JavaScript functions were missing. Using mdb's
mdb_v8 dynamic module (or dmod), we can load the application's core file and
run the following commands to get that information:

```
$ mdb path-to-node-binary path-to-core-file
Loading modules: [ libc.so.1 ld.so.1 ]
> ::load v8
> ::jsstack
```

The call stack includes both JavaScript and native function names, as well as
the value of the parameters passed to every function.

## Identifying impacted subsystem

It's time to process all the information we collected and identify in which
subsystem of Node.js the crash is located.

### Common patterns

#### `SIGILL`

Most of V8's diagnostic code used to check the consistency of data at runtime
executes an illegal instruction when it detects a problem. Thus, when a node
process exited with `SIGILL`, it is likely due to V8's diagnostic code. This
should be confirmed by looking at the call stack and making sure that the
functions at the top of the stack belong to the V8 runtime.

#### Out of memory errors

When V8 runs out of memory to store JavaScript objects, it displays an error
message mentioning an "out of memory" error, and it aborts (usually using the
illegal instruction method described previously).

## Searching through existing bug reports

Given the large number of users of Node.js, it is likely that crashes that
were already reported by other users. Now that we have identified which
subsystem/component of node is responsible for the crash, we can search through
that component's bug tracker to check if a similar problem was reported before.

#### Node.js bug reports

[Node.js bug tracker](https://github.com/nodejs/node) can be tricky to search
through, as GitHub provides an interface that is easy to use for basic searches,
but can be difficult to perform more complex ones. When the search result is
displayed, make sure to click on the `issues` tab even if the search reports
that no issues were found: the number of relevant issues found will be displayed
_only after you click_ on the issues tab.

#### V8 bug reports

When searching through issues in [V8's bug
tracker](https://bugs.chromium.org/p/v8/issues/list), it is recommended to use
[their advanced search
interface](https://bugs.chromium.org/p/v8/issues/advsearch), otherwise it's easy
to miss relevant issues.

#### libuv bug reports

Libuv's bug tracker is also [available on
GitHub](https://github.com/libuv/libuv) and the same recommendations as for
Node.js' bug tracker apply.

### Looking at Node.js' internals

If no existing similar problem can be found in any of Node.js' component's
bugtracker, then it's time to take a look into Node's internals to figure out
what the problem is.
