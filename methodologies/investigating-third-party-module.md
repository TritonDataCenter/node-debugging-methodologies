# Investigating a problem with a third-party module

## Gather a clear problem statement

1. Get detailed information about the third-party module that is not behaving
   as expected:
  * Is the program crashing when using that module?
  * Is the program logging error messages when using that module?
  * Is the program behaving unexpectedly when using that module?

  All raw data that can be gathered about the symptoms (log messages, stack
  traces, monitoring systems data, operating systems messages, etc.) must be
  gathered. The goal here is to have the most objective view of the symptoms of the alleged problem.

2. Get detailed information about the version of the module used by the
   application, as well as the version of _all_ its dependents, by gathering the
   output of the following command run from the application's directory where
   its npm dependencies are stored (usually the parent of its `node_modules`
   directory):

    ```
    $ npm ls
    ```

3. Get detailed information about the conditions under which the problem can be
   reproduced.

4. Get access to a core file. If the symptom of the problem is that the
   program is crashing, or that the program is using too much memory, then a
   core file should be generated at the time of crash or when the memory
   footprint is considered to be too large.

## Identify the type of problem

It's now time to determine to which category of problems this problem belongs:

1. crashes
2. memory leaks
3. performance problems
4. unexpected behaviors

This will guide the rest of the investigation.

## Search for existing reports of the same problem

Locate the module's issues tracker and search for similar existing issues.

## Write minimal code that reproduces the problem

If possible, isolate the problem by writing code that uses Node.js and
__no other dependency__ than the third-party module under investigation.

## Root cause the problem

Root cause the problem using one of the other methodologies described in this
repository.

## Submit ticket/issue/PR on project's repository

Once the problem has been root caused, it's time to either come up with a fix
and submit it to the project upstream, or create an issue in the project's
issues tracker with all the details gathered so far.
