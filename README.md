# Introduction to profiling

List of applications:

- gprof
- perf
- valgrind --tool=callgrind
- lttng & babeltrace
- blkin
- zipkin

# "Throw against the wall and see what sticks"

Both gprof, perf and callgrind (part of Valgrind suite) are relatively easy to use and there won't need any code changes. Thanks to that they're fast to enable them in your application get the overview what's going on, what are the most frequently called functions and in which functions application spends most of the time. The penalty for ease of use is performance hit related to the tracing of the whole application, not getting 

Pros:

- fast to enable and use,
- all of the tools gave good image of where could be possible hotspots, which functions are called the most and which functions are the most expensive computionally.

Cons:

gprof & perf: 

- the computed time and number of function calls are statistical, both of the programs checks callstack at "ticks", the more of those "ticks" pre second the worse the performance,
- low granuality (compared to lttng).

callgrind: 

-  tracks every call, so performance could be very low, in applications relying on network connections could led to timeouts and abnormal program behaviour,

## gprof

Simple tool, very easy way to enable: add '-pg' to to CFLAGS and LDFLAGS (at compiling and linking stages). After compilation run program, after finished execution there will be created file "gmon.out".

Ex:

```bash
$ ls
hello.c
$ gcc -pg -o hello hello.c
$ ./hello
$ ls
gmon.out hello hello.c
$ gprof hello gmon.out > gprof_out.txt
```

After this you could inspect flat profile and callgraph.

## perf

## valgrind --tool=callgrind  

Ex:

```bash
$ valgrind --tool=callgrind ./hello
$ callgrind_annotate
# or
$ kcachegrind # (GUI in Qt)
```

# "Sieve"

## lttng & babeltrace

## blkin

## zipkin

# Bonus round

## go pprof