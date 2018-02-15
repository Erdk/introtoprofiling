# Introduction to profiling

## Table of contents
<!--ts-->
* [Get sources](#get-sources)
* [Introduction](#introduction)
* [Throw against the wall and see what sticks](#throw-against-the-wall-and-see-what-sticks)
    * [x] [Pros & cons](#pros--cons)
    * [x] [gprof](#gprof)
    * [perf](#perf)
        * [x][perf report](#perf-report)
        * [perf diff \<filename1> \<filename2>](#perf-diff-filename1-filename2)
        * [perf data](#perf-data-convert---to-ctf-)
    * [x][callgrind](#callgrind)
        * [x][kcachegrind](#kcachegrind)
    * [x][summary](#summary)
* ["Scalpel"](#scalpel)
    * [lttng & babeltrace](#lttng--babeltrace)
    * [zipkin & blkin](#zipkin--blkin)
* [Other tools](#other-tools)
    * [go prof](#go-prof)
* [x] [Sources](#sources)
<!--ts-->

# Get sources

Clone repository and download submodules:

```bash
$ git clone https://github.com/Erdk/introtoprofiling
$ cd introtoprofiling
$ git submodule update --init
```

After (or before) clonnig repository install applications and libraries from this list:

- gprof (should be in distro repo)
- perf (should be in distro repo)
- valgrind (should be in distro repo)
- lttng & babeltrace (if not available from repo check here: http://lttng.org/download/ for download instructions)
- blkin (`git clone https://github.com/ceph/blkin`)
- zipkin (`docker run -d -p 9411:9411 openzipkin/zipkin` or `wget -O zipkin.jar 'https://search.maven.org/remote_content?g=io.zipkin.java&a=zipkin-server&v=LATEST&c=exec' && java -jar zipkin.jar`)
- (optional) go pprof

# Introduction

# "Throw against the wall and see what sticks"

`gprof`, `perf` and `callgrind` (part of `valgrind` suite) are relatively easy to use without any changes to the code. Thanks to this it's easy to use them with your application to get an overview on what's going on, what are the most frequently called functions and in which functions application spends most of the time. The penalty for ease of use is performance hit related to the tracing of the whole application, and relatively low granularity. `gprof` is the oldest of those applications, I've included it because it widely available (and maybe because of historical reasons). Most likely you'll be more satisfied with `perf` (in terms of functionality and performance) or with `valgrind`, when 100% correctness of the result is more important than execution time.

Each of those tools produces three types of results:


- flat profile: shows how much time your program spent in each function, and how many times that function was called. As the name suggests on this view you won't see any relation between functions,
- call graph: for each function shows which functions called it, which other functions was called by it, and how many times. There is also an estimate of how much time was spent in the subroutines of each function.
- annotate source code: returns source code with performance metrics in lines, where it was applicable.

## Pros & cons

Pros:

- easy to enable and use,
- all the tools gave good image of where could be possible hotspots, which functions are called the most and which functions are the most expensive computationally.

Cons:

`gprof` & `perf`: 

- the computed time and number of function calls are statistical, both of the programs check callstack at fixed time spans,
- lower granuality than lttng,
- adds overhead.

`callgrind`: 

-  executes code on simulatr @ single CPU core, tracks every call, so performance could be very low, in applications relying on network connections could led to timeouts and abnormal program behavior.

## gprof

Simple tool, very easy way to enable: add '-pg' to to CFLAGS and LDFLAGS (at compiling and linking stages). Also, with recent GCC versions you've also must add '--no-pie -fPIC' to the compiler options for it to work. Key advantage here is no need to add new code (apart from Makefile). In directory `c-ray` you could test `gprof`:

```bash
$ cd introtoprofiling/c-ray
$ make profile
```

This would create binary 'bin/c-ray-prof'. After executing it:

```bash
$ bin/c-ray-prof
```

There will be a new file in the directory: `gmon.out`. With `gprof` executable you could inspect the results.

```bash
$ gprof bin/c-ray-prof gmon.out | less
```

You should see something like this:

```
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total
 time   seconds   seconds    calls   s/call   s/call  name
 44.89    208.66   208.66 495703455     0.00     0.00  rayIntersectWithAABB
 17.60    290.46    81.80 62948596     0.00     0.00  rayIntersectsWithNode
  6.91    322.60    32.14 247988102     0.00     0.00  vectorCross
  6.57    353.15    30.55 168061837     0.00     0.00  rayIntersectsWithPolygon
  5.41    378.32    25.17 620561735     0.00     0.00  subtractVectors
  5.15    402.26    23.94  8180655     0.00     0.00  getClosestIsect
  4.18    421.70    19.44 408107946     0.00     0.00  scalarProduct
  1.48    428.58     6.88  1799410     0.00     0.00  getHighlights
...
```

The output is relatively long, after each section there's lengthy description of columns. To suppress it run grof with '-b' (brief), but I encourage you to read it at least once ;) 

Now we get to options which could help us with analysis. When you only want to inspect flat profile use '-p' switch:

```bash
$ gprof -b -p bin/c-ray-prof gmon.out |  head
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total
 time   seconds   seconds    calls   s/call   s/call  name
 44.89    208.66   208.66 495703455     0.00     0.00  rayIntersectWithAABB
 17.60    290.46    81.80 62948596     0.00     0.00  rayIntersectsWithNode
  6.91    322.60    32.14 247988102     0.00     0.00  vectorCross
  6.57    353.15    30.55 168061837     0.00     0.00  rayIntersectsWithPolygon
  5.41    378.32    25.17 620561735     0.00     0.00  subtractVectors

```

You could also inspect flat profile of functions matching "symspec" (function name, source file) '-p\<symspec>' (note the lack of whitespace between symspec and -p):

```bash
$ gprof -b -prayIntersectWithAABB bin/c-ray-prof gmon.out  | head
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total
 time   seconds   seconds    calls  us/call  us/call  name
100.02    208.66   208.66 495703455     0.42     0.42  rayIntersectWithAABB

```

This could be useful, when you have multiple static functions with the same name. To exclude functions from flat profile use '-P' switch:

```bash
$ gprof -b -PrayIntersectWithAABB bin/c-ray-prof gmon.out  | head
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total
 time   seconds   seconds    calls   s/call   s/call  name
 31.93     81.80    81.80 62948596     0.00     0.00  rayIntersectsWithNode
 12.55    113.94    32.14 247988102     0.00     0.00  vectorCross
 11.92    144.49    30.55 168061837     0.00     0.00  rayIntersectsWithPolygon
  9.82    169.65    25.17 620561735     0.00     0.00  subtractVectors
  9.35    193.60    23.94  8180655     0.00     0.00  getClosestIsect
```

When you compare with previous example you could see, that all values were adjusted. This could be useful when you want to exclude function you couldn't optimize and want to have a closer look on other potential candidates for optimization.

In similar way you could inspect call graphs. To do this use '-q' and '-Q' switches. First display ony call graph:

```bash
$ gprof -b -q bin/c-ray-prof gmon.out  | head -n 27
                        Call graph


granularity: each sample hit covers 2 byte(s) for 0.00% of 464.88 seconds

index % time    self  children    called     name
                                                 <spontaneous>
[1]     98.3    3.86  453.32                 renderThread [1]
                1.71  448.86 2703244/2703244     newTrace <cycle 1> [9]
                0.21    1.11 1366674/1366676     transformCameraView [28]
                0.35    0.25 1632466/10151652     normalizeVector [17]
                0.54    0.00 1537256/1537256     getPixel [45]
                0.18    0.00 4013744/19821218     getRandomDouble [36]
                0.11    0.00 1441645/13375163     addColors [34]
                0.00    0.00       6/6           computeTimeAverage [171]
                0.00    0.00       5/5           getTile [172]
-----------------------------------------------
[2]     96.9    1.71  448.86 2703244+4995853 <cycle 1 as a whole> [2]
                0.53  274.74 1721452             getLighting <cycle 1> [5]
                0.44  173.35 3080079             newTrace <cycle 1> [9]
                0.74    0.77 2897566             getReflectsAndRefracts <cycle 1> [27]
-----------------------------------------------
                9.68  163.67 3308211/8180655     newTrace <cycle 1> [9]
               14.26  241.06 4872444/8180655     isInShadow [7]
[3]     92.2   23.94  404.72 8180655         getClosestIsect [3]
               81.80  315.75 62948596/62948596     rayIntersectsWithNode [4]
                1.93    5.25 15315957/15315957     rayIntersectsWithSphereTemp [14]

```

Line with index denotes 'main' function of call graph. Functions above are 'parents', callers. Functions below are 'children', callees. In this example, call graph with index 3:

Function | relation
---------|---------
newTrace | top most function
isInShadow |  function called by newTrace
getClosestInsect | main function of this call graph
rayIntersectsWithNode | function called by getClosestInsect
rayIntersectsWithSphereTemp | function called by rayIntersectsWithNode

Numbers in 'called' column denotes respectively number of calls in scope of the parent and number of total calls. Analogous to '-p' switch with '-q\<symspec>' you could inspect call graph of function you choose:

```bash
$ gprof -b -qrayIntersectsWithNode bin/c-ray-prof gmon.out
                        Call graph


granularity: each sample hit covers 2 byte(s) for 0.00% of 464.88 seconds

index % time    self  children    called     name
                             213618252             rayIntersectsWithNode [4]
               81.80  315.75 62948596/62948596     getClosestIsect (3)
[4]     85.5   81.80  315.75 62948596+213618252 rayIntersectsWithNode [4]
              208.66    0.00 495703455/495703455     rayIntersectWithAABB [8]
               30.55   74.51 168061837/168061837     rayIntersectsWithPolygon [10]
                1.58    0.00 18565813/25178963     vectorScale [22]
                0.45    0.00 7636053/15517841     addVectors [35]
                             213618252             rayIntersectsWithNode [4]
-----------------------------------------------
              208.66    0.00 495703455/495703455     rayIntersectsWithNode [4]
[8]     44.9  208.66    0.00 495703455         rayIntersectWithAABB [8]
-----------------------------------------------
               30.55   74.51 168061837/168061837     rayIntersectsWithNode [4]
[10]    22.6   30.55   74.51 168061837         rayIntersectsWithPolygon [10]
               31.88    0.00 245938645/247988102     vectorCross [11]
               23.91    0.00 589609039/620561735     subtractVectors [12]
               17.05    0.00 357801568/408107946     scalarProduct [13]
                1.68    0.00 6623695/6623695     uvFromValues [23]
...
```

Last, but not least: `gprof` could annotate source file with profile information. As symspec the best is to choose file (e.g. render.c) to have annotated source file and files with callees:

```C
$ gprof -b -Arender.c bin/c-ray-prof gmon.out | less
...
*** File .../c-ray/src/sphere.c:
                //
                //  sphere.c
                //  C-Ray
                //
                //  Created by Valtteri Koskivuori on 28/02/15.
                //  Copyright (c) 2015 Valtteri Koskivuori. All rights reserved.
                //

                #include "includes.h"
                #include "sphere.h"

           3 -> struct sphere newSphere(struct vector pos, double radius, int materialIndex) {
                        return (struct sphere){pos, radius, materialIndex};
                }

                //Just check for intersection, nothing else.
       ##### -> bool rayIntersectsWithSphereFast(struct lightRay *ray, struct sphere *sphere) {
                        double A = scalarProduct(&ray->direction, &ray->direction);
                        struct vector distance = subtractVectors(&ray->start, &sphere->pos);
                        double B = 2 * scalarProduct(&ray->direction, &distance);
                        double C = scalarProduct(&distance, &distance) - (sphere->radius * sphere->radius);
                        double trigDiscriminant = B * B - 4 * A * C;
                        if (trigDiscriminant < 0) {
                                return false;
                        } else {
                                return true;
                        }
                }

                //Calculates intersection with a sphere and a light ray
    39921178 -> bool rayIntersectsWithSphere(struct lightRay *ray, struct sphere *sphere, double *t) {
                        bool intersects = false;

                        //Vector dot product of the direction
                        double A = scalarProduct(&ray->direction, &ray->direction);

                        //Distance between start of a lightRay and the sphere position
                        struct vector distance = subtractVectors(&ray->start, &sphere->pos);

                        double B = 2 * scalarProduct(&ray->direction, &distance);

                        double C = scalarProduct(&distance, &distance) - (sphere->radius * sphere->radius);

                        double trigDiscriminant = B * B - 4 * A * C;

                        //If discriminant is negative, no real roots and the ray has missed the sphere
                        if (trigDiscriminant < 0) {
                                intersects = false;
                        } else {
                                double sqrtOfDiscriminant = sqrt(trigDiscriminant);
                                double t0 = (-B + sqrtOfDiscriminant)/(2);
                                double t1 = (-B - sqrtOfDiscriminant)/(2);

                                //Pick closest intersection
                                if (t0 > t1) {
                                        t0 = t1;
                                }

                                //Verify intersection is larger than 0 and less than the original distance
                                if ((t0 > 0.001f) && (t0 < *t)) {
                                        *t = t0;
                                        intersects = true;
                                } else {
                                        intersects = false;
                                }
                        }
                        return intersects;
                }


Top 10 Lines:

     Line      Count

       31   39921178
       12          3

Execution Summary:

        3   Executable lines in this file
        3   Lines executed
   100.00   Percent of the file executed

 39921181   Total number of line executions
13307060.33   Average executions per line


*** File .../c-ray/src/main.c:

```

Used functions are annotated with 'num ->' prior to function definition, denoting number of executions. Also after each source file there's footer with statistics, the hot lines, execution summary and total number of executions.

## perf

Perf 
TODO: much better than gprof, faster execution, more detailed profile, easy to profile userspace app or kernel. Better tui. Con: annotation interleaves asm with C. Annotating with C only available when app was compiled with debug information.

To get trace you have to run `perf record <path to executable>`:
- Just flat profile (default): `perf record bin/c-ray`
- Flat profile + call graph: `perf record --call-graph bin/c-ray`

### perf report

`$ perf report`

Main window:
![perf main window](/img/perf1.png)

To get help press 'h':
![perf help window](/img/perf2.png)

Select function with arrow keys and press 'a' to go to the annotated view. Without debug information there will be only asm:
![perf annotated asm](/img/perf3.png)

When app was compiled with debug information asm code will be interleaved with C code:
![perf annotated asm and C](/img/perf4.png)

GTK2 interface:
`perf report --gtk`

### perf diff \<filename1> \<filename2>

`perf diff \<filename1> \<filename2>` compares to perf files.
TODO: screenshot, fedora by default doesn't include support for this

### perf data convert --to-ctf <path>
  
  convert native perf data format to CTF, understandable by babeltrace, kcachegrind.
  TODO: example, fedora by default doesn't include support for this

## callgrind

Similarily to previous tools `callgrind` doesn't require changes to the code, it's sufficient to compile with optimizations and debug information turned on. Contrary to `gprof` and `perf` it doesn't run code directly on host CPU, but via it's own simulator. This allow  

To gather profile:
```bash
$ valgrind --tool=callgrind bin/c-ray
```

To inspect flat profile:
```
$ callgrind_annotate callgrind.out.13671                                 
--------------------------------------------------------------------------------                  
Profile data file 'callgrind.out.13671' (creator: callgrind-3.13.0)                               
--------------------------------------------------------------------------------                  
I1 cache:                                                                                         
D1 cache:                                                                                         
LL cache:                                                                                         
Timerange: Basic block 0 - 119733969377                                                           
Trigger: Program termination                                                                      
Profiled target:  bin/c-ray (PID 13671, part 1)                                                   
Events recorded:  Ir                                                                              
Events shown:     Ir                                                                              
Event sort order: Ir                                                                              
Thresholds:       99                                                                              
Include dirs:                                                                                     
User annotated:                                                                                   
Auto-annotation:  off                                                                             
                                                                                                  
--------------------------------------------------------------------------------                  
             Ir                                                                                   
--------------------------------------------------------------------------------                  
936,450,558,371  PROGRAM TOTALS                                                                   
                                                                                                  
--------------------------------------------------------------------------------                  
             Ir  file:function                                                                    
--------------------------------------------------------------------------------

238,763,680,382  .../c-ray/src/bbox.c:rayIntersectWithAABB [.../c-ray/
bin/c-ray]                  
168,001,295,859  .../c-ray/src/raytrace.c:rayIntersectsWithNode'2 [...
/c-ray/bin/c-ray]                  
158,259,872,671  .../c-ray/src/poly.c:rayIntersectsWithPolygon [.../c-
ray/bin/c-ray]
...
```

`I1 cache`, `D1 cache`, `LL cache` refers to simulated CPU caches, in this example we didn't turn option to gather them so there's no data here.

With switch `--tree=<mode>` you could turn on information about call graph: `caller`, `calling`, and `both`:

```
$ callgrind_annotate --tree=caller callgrind.out.13671
...
187,684,131,006  < .../c-ray/src/raytrace.c:rayIntersectsWithNode'2 (3842892764x)
 [.../c-ray/bin/c-ray]
 51,079,549,376  < .../c-ray/src/raytrace.c:rayIntersectsWithNode (1060050002x) [
.../c-ray/bin/c-ray]
238,763,680,382  *  .../c-ray/src/bbox.c:rayIntersectWithAABB [.../c-r
ay/bin/c-ray]
...
```

```
$ callgrind_annotate --tree=caller callgrind.out.13671
...
238,763,680,382  *  .../c-ray/src/bbox.c:rayIntersectWithAABB [.../c-r
ay/bin/c-ray]

168,001,295,859  *  .../c-ray/src/raytrace.c:rayIntersectsWithNode'2 [.../c-ray/bin/c-ray]
346,471,804,141  >   .../c-ray/src/poly.c:rayIntersectsWithPolygon (2027649930x) [.../c-ray/bin/c-ray]                   
  1,206,571,355  >   .../c-ray/src/vector.c:vectorScale (109688305x) [.../c-ray/bin/c-ray]
187,684,131,006  >   .../c-ray/src/bbox.c:rayIntersectWithAABB (3842892764x) [.../c-ray/bin/c-ray]
Negative repeat count does nothing at /usr/bin/callgrind_annotate line 828.
3,118,318,860,534  >   .../c-ray/src/raytrace.c:rayIntersectsWithNode'2 (3488504364x) [.../c-ray/bin/c-ray]
  1,316,259,660  >   .../c-ray/src/vector.c:addVectors (109688305x) [.../c-ray/bin/c-ray]
...
```

```
$ callgrind_annotate --tree=both callgrind.out.13671
...
3,118,318,860,534  < .../c-ray/src/raytrace.c:rayIntersectsWithNode'2 (3488504364x) [.../c-ray/bin/c-ray]                                                      
704,680,062,021  < .../c-ray/src/raytrace.c:rayIntersectsWithNode (354388400x) [.../c-ray/bin/c-ray]                                                           
168,001,295,859  *  .../c-ray/src/raytrace.c:rayIntersectsWithNode'2 [.../c-ray/bin/c-ray]                                                                     
Negative repeat count does nothing at /usr/bin/callgrind_annotate line 828.                                                                                                                
3,118,318,860,534  >   .../c-ray/src/raytrace.c:rayIntersectsWithNode'2 (3488504364x) [.../c-ray/bin/c-ray]                                                    
187,684,131,006  >   .../c-ray/src/bbox.c:rayIntersectWithAABB (3842892764x) [.../c-ray/bin/c-ray]                                                             
  1,316,259,660  >   .../c-ray/src/vector.c:addVectors (109688305x) [.../c-ray/bin/c-ray]                                                                      
  1,206,571,355  >   .../c-ray/src/vector.c:vectorScale (109688305x) [.../c-ray/bin/c-ray]                                                                     
346,471,804,141  >   .../c-ray/src/poly.c:rayIntersectsWithPolygon (2027649930x) [.../c-ray/bin/c-ray]
...
```

With `callgrind` there also comes `callgrind_control`, tool which allows to inspect application during run. 

To get quick statistics about running application:
```bash
$  callgrind_control -s 22710
PID 22710: bin/c-ray
sending command status internal to pid 22710
  Number of running threads: 9, thread IDs: 1 2 3 4 5 6 7 8 9
  Events collected: Ir
  Functions: 575 (executed 3,805,993,131, contexts 575)
  Basic blocks: 5,681 (executed 31,519,541,333, call sites 1,252)
```

Inspect backtrace:
```bash
$ callgrind_control -b 22710 |head -n 30
PID 22710: bin/c-ray
sending command status internal to pid 22710

  Frame: Backtrace for Thread 1
   [ 0]  nanosleep (843 x)
   [ 1]  usleep (842 x)
   [ 2]  sleepMSec (843 x)
   [ 3]  main (1 x)
   [ 4]  (below main) (1 x)
   [ 5]  _start (1 x)
   [ 6]  0x0000000000000ed0


  Frame: Backtrace for Thread 2
   [ 0]  subtractVectors (289348042 x)
   [ 1]  rayIntersectsWithPolygon (289348055 x)
   [ 2]  rayIntersectsWithNode (25762561 x)
   [ 3]  rayIntersectsWithNode (128149695 x)
   [ 4]  getClosestIsect (3052621 x)
   [ 5]  newTrace (3052621 x)
   [ 6]  renderThread (8 x)
   [ 7]  start_thread (8 x)
   [ 8]  clone
...
```

With `callgrind_control -i on|off` you could turn on or off instrumentation during runtime. You could combine it with `--instr-atstart=no|yes` option to `valgrind` when you start application, to start without instrumentation, then turn it on for a few minutes to gather profile and then turn it off again.

### kcachegrind

It's possible to base analysis on `callgrind_annotate` output, but better suited to this is KCacheGrind. It provides graphical interface and it provides visualization of data, which helps with analysis. It's available for Linux, and probobly for Windows as QCacheGrind (I saw old builds on sourceforge but I didn't test them...).

List of callers and callees of choosen function:
![kcachegrind callers and callees](/img/kcachegrind1.png)

All callers and call graph:
![kcachegrind all callers and call graph](/img/kcachegrind2.png)

Visual maps of execution time:
![kcachegrind maps](/img/kcachegrind3.png)

## Summary

Below are total times of execution for `gprof`, `perf` and `callgrind`. Only one run, results measured with `time`, just to show how roughly those execution times differ. For obvious reasons there were differences in compilation options of our application:

- gprof: -O2 -g -pg --no-pie -fPIC
- perf: -O2 -g
- callgrind: -O2 -g

`gprof` required the most changes, `perf` only requires `-g` when you want to see annotated code in C, not only in asm. `callgrind` requiers `-g` and `-O2`: first to gather required information, second to have semi-decent execution time. I've tested all with `-O2` because you **want** to run optimized executable to gather results as close to reality as possible.

app                | time
-------------------|--------
gprof              | 5119.32s user 5.38s system 762% cpu 11:12.00 total
perf (flat)        | 309.79s user 86.39s system 725% cpu 54.629 total
perf (flat + call) | 299.25s user 90.70s system 719% cpu 54.199 total
callgrind (-02 -g) | 4699.96s user 17.50s system 100% cpu 1:18:30.59 total

Keep in mind that despite that `callgrind` took less seconds than `gprof` total execution time was much longer. Code instrumented with `gprof` runs directly on CPU, when with `callgrind` inside `valgrind`'s simulator, which runs on single CPU and serializes all ops. The fastest of all three was `perf`, and in my opinion is the best option to use. If for some reason you couldn't run `perf` (e.g. you couldn't prepend command) the next option would be `gprof`. `callgrind` could be used if you really, **really** need 100% accurate profile. Given the methodology I propose here (get rough idea what's going on and then use other tools to inspect specific codepaths) I don't think it would be a good match for this stage.

# "Scalpel"

## lttng & babeltrace

## zipkin && blkin

# Other tools

## go pprof

# Sources

- gprof: https://www.thegeekstuff.com/2012/08/gprof-tutorial/
- gprof: `man gprof`
- perf: https://dev.to/etcwilde/perf---perfect-profiling-of-cc-on-linux-of
- perf: https://perf.wiki.kernel.org/index.php/Tutorial
- valgrind: http://valgrind.org/docs/manual/cl-manual.html
- (Q|K)cachegrind: https://github.com/KDE/kcachegrind
- LTTNG: https://lttng.org/docs/v2.10/#doc-what-is-tracing