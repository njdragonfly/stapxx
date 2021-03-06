NAME
====

stap++ - Simple macro language extensions to systemtap

Synopsis
========

    $ stap++ -I ./tapset -x 12345 --arg limit=10 samples/ngx-upstream-post-conn.sxx
    $ stap++ -e 'probe begin { println("hello") exit() }'

Description
===========

This interpreter adds some simple macro language extensions to the systemtap scripting language.

Efforts has been made to ensure that this macro language expansion does
not affect the source line numbers so that the line numbers reported by `stap` are exactly the same in the original `.sxx` source files.

Features
========

Standard Macro Variables
------------------------

### $^exec_path

The variable `$^exec_path` is always evaluated to the path to the executable file
for the pid specified by the `-x` option.

Here is an example:

    probe process("$^exec_path").function("blah") { ... }

### $^libNAME_path

This variable expands to the absolute path of the DSO library file specified by a pattern.

`stap++` automatically scans all the loaded DSO files in the running process (if the `-x PID` option is specified) to find a match. If it fails to find a match, this variable will take the value of `$^exec_path`, that is, assuming the library is statically linked.

Below is an example for tracing a user-land function in the libpcre library:

    probe process("$^libpcre_path").statement("pcre_exec")
    {
        println("pcre_exec called")
        print_ubacktrace()
    }

### $^arg_NAME

This variable can evaluate to the value of a specified command-line argument. For example, `$^arg_limit` is evaluated to the value of the command line argument `limit` specified like this:

    stap++ --arg limit=1000

You can dump out all the available arguments in the stap++ script by specifying the --args option, for example:

    $ stap++ --args foo.sxx
    --arg method=VALUE (default: )
    --arg time=VALUE (default: 60)

### Default values

It's possible to specify a default value for a macro variable by means of the `default` trait, as in

    foreach (key in stats- limit $^arg_limit :default(1000)) {
        ...
    }

where `$^arg_limit` takes the default value 1000 when the user does not specify the `--arg limit=N` command-line option while invoking `stap++`.

User-defined Macro Variables
----------------------------

It's possible to bind a `@cast()` or `@var()` expression to a user-defined macro variable of the form `$*NAME`. Here is an example,

    sock = sockfd_lookup(fd)
    $*sock := @cast(sock, "socket", "kernel")

    printf(", sock->state:%d", $*sock->state)
    state = $*sock->sk->__sk_common->skc_state
    printf(", sock->sk->sk_state:%d (%s)\n", state, tcp_sockstate_str(state))

Note that we used the `:=` operator to bind a `@cast()` or `@var()` expression to user variable `$*sock`, and later we reference it whenever we need that `@cast()` or `@var()` expression.

The scope of user variables is always limited to the current `.sxx` source file.

Tapset Modules
--------------

One can use the stap++ language to define new tapset module files and later use the `@use` directive to load the module in a main stap++ program file.

For example, we can have a module file located at `./tapset/kernel/socket.sxx`:

    // module kernel.socket
    function socketfd_lookup(fd)
    {
        ...
    }

And then in a stap++ script file, `foo.sxx`, we can import this library like this

    @use kernel.socket

and in `foo.sxx`, we are now free to call the `socketfd_lookup` function defined in the `kernel.socket` module.

Finally, we should invoke the `stap++` interpreter like this:

    stap++ -I ./tapset foo.sxx ...

Note the `-I ./tapset` option that specifies the search path for the stap++ tapset modules. The default module search paths are `.`, and `<bin-dir>/tapset`, where `<bin-dir>` is the directory where `stap++` sits in.

Unlike `stap`, only the used stapset modules are processed so as to reduce startup time.

One can `@use` multiple modules like this

    @use kernel.socket
    @use nginx.upstream

or equivalently,

    @use kernel.socket, nginx.upstream

All those macro variables are free to use in the tapset module files.

Shorthands
----------

### @pfunc(FUNCTION)

This is equivalent to `process("$^exec_path").function("FUNCTION").

For example,

    probe @pfunc(ngx_http_upstream_finalize_request),
          @pfunc(ngx_http_upstream_send_request)
    {
        ...
    }

is equivalent to

    probe process("$^exec_path").function("ngx_http_upstream_finalize_request"),
          process("$^exec_path").function("ngx_http_upstream_send_request")
    {
        ...
    }

Samples
=======

ngx-rps
-------

Calculate the current number of requests per second handled by the Nginx
worker process specified by its pid:

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming one nginx worker process has the pid 19647.
    $ ./samples/ngx-rps.sxx -x 19647
    WARNING: Tracing process 19647.
    Hit Ctrl-C to end.
    [1376939543] 300 req/sec
    [1376939544] 235 req/sec
    [1376939545] 235 req/sec
    [1376939546] 166 req/sec
    [1376939547] 238 req/sec
    [1376939548] 234 req/sec
    ^C

The numbers in the leading square brackets are the current timestamp (seconds since the Epoch).

Behind the scene, the Nginx main requests' completion events are traced.

ngx-req-latency
---------------

Calculates the distribution of the Nginx request latencies (excluding the request header reading time) in any specified
Nginx worker process at real time:

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    $ ./samples/ngx-req-latency.sxx -x 28078
    WARNING: Start tracing process 28078 (/path/to/some/program)...
    ^C
    Distribution of the main request latencies (in microseconds)
    (min/avg/max: 92/242181/42808832)
        value |-------------------------------------------------- count
           16 |                                                      0
           32 |                                                      0
           64 |                                                      8
          128 |                                                      1
          256 |                                                      3
          512 |@@@@@                                               274
         1024 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  2474
         2048 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                     1547
         4096 |@@@@@@@@@@@@@@@@@@@                                 952
         8192 |@@@@@@@@@@                                          500
        16384 |@@@@@@@                                             359
        32768 |@@@@@@@@                                            414
        65536 |@@@@@@@@@@@@                                        644
       131072 |@@@@@@@@@@@@@@@@@                                   851
       262144 |@@@@@@@@@@@@                                        614
       524288 |@@@@@@                                              334
      1048576 |@@                                                  147
      2097152 |                                                     46
      4194304 |                                                     24
      8388608 |@                                                    64
     16777216 |                                                      1
     33554432 |                                                      1
     67108864 |                                                      0
    134217728 |                                                      0

One can also filter out requests by a specified request method name via the `--arg method=METHOD` option. For instance,

    $ ./samples/ngx-req-latency.sxx -x 5447 --arg method=POST --arg time=60
    Start tracing process 5447 (/usr/local/nginx-waf/sbin/nginx-waf)...
    Please wait for 60 seconds...
    (Tracing only POST request methods)

    Distribution of the main request latencies (in microseconds) for 52 samples:
    (min/avg/max: 1167/8373/28281)
    value |-------------------------------------------------- count
      256 |                                                    0
      512 |                                                    0
     1024 |@@                                                  2
     2048 |@@@@@@@@                                            8
     4096 |@@@@@@@@@@@@@@@@@@@@@@@                            23
     8192 |@@@@@@@@@@@@@@                                     14
    16384 |@@@@@                                               5
    32768 |                                                    0
    65536 |                                                    0

We can also see from the example above that we can limit the sampling period by specifying the `--arg time=SECONDS` option.

ctx-switches
------------

Calculates the CPU context switching rate (number/second) in any specified user process at real time:

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming the target process pid is 6254:
    $ ./samples/ctx-switches.sxx -x 6254
    WARNING: Tracing process 6254 (/path/to/some/process).
    Hit Ctrl-C to end.
    [1379631372] 13741 cs/sec
    [1379631373] 13330 cs/sec
    [1379631374] 14263 cs/sec
    [1379631375] 14424 cs/sec
    [1379631376] 14591 cs/sec
    [1379631377] 11108 cs/sec
    [1379631378] 12620 cs/sec
    [1379631379] 12519 cs/sec
    [1379631380] 13479 cs/sec
    [1379631381] 14614 cs/sec
    [1379631382] 14721 cs/sec
    [1379631383] 13408 cs/sec
    [1379631384] 14682 cs/sec

Both switch-in and switch-out are counted in this tool.

High context switching rate usually means higher overhead in the system. Ideally
we could keep the context switching rate low.

ngx-lj-gc
---------

This tool analyses the LuaJIT 2.0 GC in the specified Nginx worker process via the [ngx_lua](http://wiki.nginx.org/HttpLuaModule) mdoule.

For now, it just prints out the total memory currently allocated in the LuaJIT GC. For example,

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming the nginx worker process's pid is 4771:
    $ ./samples/ngx-lj-gc.sxx -x 4771
    Start tracing 4771 (/opt/nginx/sbin/nginx)
    Total GC count: 258618 bytes

ngx-lj-gc-objs
--------------

This tool dumps the GC objects' memory usage stats in any specified running Nginx worker process
according to the GC object's types.

This tool reveals exactly how the memory is distributed among all Lua value types, which is useful for optimizing Lua code's memory usage and debugging memory leak issues in the Lua programs.

For now, only LuaJIT 2.0 is supported.

Here is an example.

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming the nginx worker pid is 5686:
    $ ngx-lj-gc-objs.sxx -x 5686
    Start tracing 5686 (/opt/nginx/sbin/nginx)

    main machine code area size: 65536 bytes
    C callback machine code size: 4096 bytes
    GC total size: 922648 bytes
    GC state: sweep

    4713 table objects: max=6176, avg=90, min=32, sum=428600 (in bytes)
    3341 string objects: max=2965, avg=47, min=18, sum=159305 (in bytes)
    677 function objects: max=144, avg=29, min=20, sum=20224 (in bytes)
    563 userdata objects: max=8895, avg=82, min=24, sum=46698 (in bytes)
    306 proto objects: max=34571, avg=541, min=78, sum=165557 (in bytes)
    287 upvalue objects: max=24, avg=24, min=24, sum=6888 (in bytes)
    102 trace objects: max=928, avg=337, min=160, sum=34468 (in bytes)
    8 cdata objects: max=24, avg=17, min=16, sum=136 (in bytes)
    7 thread objects: max=1648, avg=1493, min=568, sum=10456 (in bytes)
    JIT state size: 6920 bytes
    global state tmpbuf size: 2948 bytes
    C type state size: 2520 bytes

    My GC walker detected for total 922648 bytes.
    5782 microseconds elapsed in the probe handler.

For LuaJIT instances with big memory usage, you need to increase the `MAXACTION` threshold, as in

    $ ngx-lj-gc-objs.sxx -x 14378 -D MAXACTION=200000
    Start tracing 14378 (/usr/local/nginx-fl/sbin/nginx-fl)

    main machine code area size: 65536 bytes
    C callback machine code size: 4096 bytes
    GC total size: 9683407 bytes
    GC state: pause

    27948 table objects: max=131112, avg=106, min=32, sum=2983944 (in bytes)
    22343 string objects: max=1421562, avg=198, min=18, sum=4432482 (in bytes)
    12168 userdata objects: max=8916, avg=50, min=27, sum=619223 (in bytes)
    2837 function objects: max=148, avg=27, min=20, sum=78264 (in bytes)
    1200 upvalue objects: max=24, avg=24, min=24, sum=28800 (in bytes)
    650 proto objects: max=3860, avg=313, min=74, sum=203902 (in bytes)
    349 thread objects: max=1648, avg=774, min=424, sum=270464 (in bytes)
    202 trace objects: max=1560, avg=375, min=160, sum=75832 (in bytes)
    9 cdata objects: max=36, avg=17, min=12, sum=156 (in bytes)
    JIT state size: 7696 bytes
    global state tmpbuf size: 710772 bytes
    C type state size: 4568 bytes

    My GC walker detected for total 9683407 bytes.
    45008 microseconds elapsed in the probe handler.

The "objects" are Lua values that actually participate in garbage
collection (GC).

Primitive Lua values like numbers, booleans, nils, and light user data
do not participate in GC, so we shall never see them listed in the
output of the ngx-lj-gc-objs tool.

Another interesting exception is empty Lua strings, they are specially
handled by LuaJIT and they never appear in the output either.

The following types of objects participate in GC and thus considered
"GC objects" in LuaJIT 2.0:

* string: Lua strings
* upvalue: Lua Upvalues
* thread: Lua threads (i.e., Lua coroutines)
* proto: Lua function prototypes
* function: Lua functions (Lua closures) and C functions
* cdata: cdata created by the FFI API in Lua.
* table: Lua tables
* userdata: Lua user data
* trace: JIT compiled Lua code paths

Note that for the space calculated for aggregate objects like "table" and
"function", only the size of their backbones is calculated. For
example, for a Lua table, we do not follow the references to its value
objects and key objects (but we do include the size of the "key" and
"value" reference pointers themselves). Similarly, for a Lua function
object, we do not follow its upvalues either. Therefore, we can safely
add up the sizes of all the GC objects to obtain the total size
allocated by the LuaJIT GC without repeating anyone.

It's worth mentioning that the LuaJIT GC may also allocate some space
for some components in the VM itself (on demand):

* the JIT compiler state (i.e., the "JIT state size" item in the tool's output).
* the state for FFI C types (i.e., the "C type state size" item in the output).
* global state temporary buffer (i.e., the "global state tmpbuf size" item).

These allocations may not happen for trivial examples. For example, if
the JIT compiler is disabled, we won't see nonzero "JIT state size".
Similarly, if we don't use FFI in our Lua code at all, we won't see
nonzero "C type state size" either.

Like any language with a proper GC, any unused Lua objects will be
automatically freed by the LuaJIT GC. You make a Lua object become
unused by avoid referencing the object from any "GC root objects"
either directly or indirectly.

It is usually fine for Lua objects to stay a little longer then needed.Basically it's fine and also
normal to see the GC count (or the memory allocated by the GC) going
up and down during the lifetime of the process.

That is how a typical incremental GC works. The LuaJIT GC cycle
consists of various different phases. And all these phases are divided
into many small pieces that interleave with normal Lua code execution.
For example, when the GC cycle is at the "sweep-string" phase,
non-string GC objects will not freed at all until the GC cycle later
enters the "sweep" phase.

This tool is good at finding out the largest GC objects
in your already bloated Nginx worker processes. It is not really designed for debugging
individual requests due to the non-determinism of the GC on micro
levels. You should load your nginx workers by tools like ab and
weighttp or just trace workers in production, so as to make your nginx worker eat up a _lot_ of memory. The
more, the merrier. After that, run ngx-lj-gc-objs on your largest
nginx worker process.

To debug individual requests, you *can* force the LuaJIT GC to free all the unsed objects at once by calling

    collectgarbage()

This forces the LuaJIT (or standard Lua interpreter) GC to run a
complete collection cycle immediately. See
http://www.lua.org/manual/5.1/manual.html#pdf-collectgarbage for more
details. But this is usually very expensive to call and it is strongly discouraged for production use.

epoll-et-lt
-----------

This tool can checks how many epoll_ctl syscalls use the epoll edge-trigger (ET) mode and how many use the epoll level-trigger (LT) model. The statistics is gathered and printed out every 1 second. This tool is not specific to Nginx.

Here is an example:

    # making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    $ ./samples/epoll-et-lt.sxx -x 5728
    Tracing epoll_ctl in user process 5728 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    51 ET, 0 LT.
    384 ET, 0 LT.
    388 ET, 0 LT.
    390 ET, 0 LT.
    389 ET, 0 LT.
    394 ET, 0 LT.
    394 ET, 0 LT.
    396 ET, 0 LT.
    395 ET, 0 LT.
    384 ET, 0 LT.
    394 ET, 0 LT.
    396 ET, 0 LT.
    397 ET, 0 LT.
    ^C

We can see that Nginx is using epoll ET exclusively :)

Below is another example for KyotoTycoon servers:

    $ epoll-et-lt.sxx -x 4011
    Tracing epoll_ctl in user process 4011 (/usr/local/bin/ktserver)...
    Hit Ctrl-C to end.
    0 ET, 0 LT.
    0 ET, 3 LT.
    0 ET, 0 LT.
    0 ET, 2 LT.
    0 ET, 0 LT.
    0 ET, 0 LT.
    0 ET, 4 LT.
    0 ET, 3 LT.
    0 ET, 0 LT.
    0 ET, 5 LT.
    0 ET, 1 LT.
    0 ET, 1 LT.
    0 ET, 1 LT.
    0 ET, 0 LT.
    ^C

We can see that the `ktserver` process is using epoll LT all the time :)

epoll-loop-blocking-distr
-------------------------

This tool can sample any specified user process for the specified time (by default, 5 seconds) and print out the distribution
of the latency between successive epoll_wait syscalls.

Essentially, it can give you a picture about the blocking latency involved in
the process's epoll-based event loop (if there is one).

Here is an example for analyzing a massively blocked Nginx worker processes in production (by really bad disk IO):

    $ ./samples/epoll-loop-blocking-distr.sxx -x 19647 --arg time=60
    Start tracing 19647...
    Please wait for 60 seconds.
    Distribution of epoll loop blocking latencies (in milliseconds)
    max/avg/min: 1097/0/0
    value |-------------------------------------------------- count
        0 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  18471
        1 |@@@@@@@@                                            3273
        2 |@                                                    473
        4 |                                                     119
        8 |                                                      67
       16 |                                                      51
       32 |                                                      35
       64 |                                                      20
      128 |                                                      23
      256 |                                                       9
      512 |                                                       2
     1024 |                                                       2
     2048 |                                                       0
     4096 |                                                       0

Note the long tail from 4ms ~ 1.1sec.

To further analyse exactly what is blocking the epoll loop, you
can use the off-CPU and on-CPU flame graph tools:

https://github.com/agentzh/nginx-systemtap-toolkit#sample-bt

https://github.com/agentzh/nginx-systemtap-toolkit#sample-bt-off-cpu

Author
======

Yichun Zhang (agentzh), <agentzh@gmail.com>, CloudFlare Inc.

Copyright and License
=====================

This module is licensed under the BSD license.

Copyright (C) 2013, by Yichun "agentzh" Zhang (章亦春) <agentzh@gmail.com>, CloudFlare Inc.

All rights reserved.

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

See Also
========
* SystemTap Wiki Home: http://sourceware.org/systemtap/wiki
* Nginx Systemtap Toolkit: https://github.com/agentzh/nginx-systemtap-toolkit
