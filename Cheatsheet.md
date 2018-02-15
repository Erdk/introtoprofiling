# Cheatsheet

## gprof

- compile application with: `-O2 -g -pg --no-pie -fPIC`, link with `-pg`
- to gather profile: just run your program
- to inspect profile: `gprof <path to executable> gmon.out`
- flat profile: `gprof -p <path to executable> gmon.out`
- flat profile of choosen function: `gprof -p<function name> <path to executable> gmon.out`
- flat profile without choosen function: `gprof -P<function name> <path to executable> gmon.out`
- call graph: `gprof -q <path to executable> gmon.out`
- call graph of specific funtion: `gprof -q<function name> <path to executable> gmon.out`
- call graph without choosen function: `gprof -Q<function name> <path to executable> gmon.out`
- annotated sources:  `gprof -A<source file> <path to executable> gmon.out`

## perf
- compile application with: `-O2`, if you need annotated C source compile with `-O2 -g`,
- to get basic runtime statistics of application: `perf stat <path to executable>`
- to gather flat profile: `perf record <path to executable>`
- to gather call graph: `perf record -g <path to executable>`
  
  executing with `-g` produces profile data which have both flat profile and call graph, you could inspect both of them with below commands.
- to inspect flat profile: `perf report`
- to inspect call graph: `perf report -g`
- to compare to profiles: `perf diff <first profile> <second profile>`
  
  When recording profile you could set custom name for filename, to do so run `perf record -o <filename> ...`
- get systemwide realtime look at performance counters: `sudo perf top`
- get realtime look at performance counters: `perf top -p <pid>`

## callgrind
- compile with: `-O2 -g`
- to gather profile: `valgrind --tool callgrind <path to executable>`
- flat profile: `callgrind_annotate callgrind.out.<pid>`
  
  after each execution there will be new file starting with `callgrind.out.` with appended `pid` of process

- call graph, callers: `callgrind_annotate --tree=caller callgrind.out.<pid>`
- call gprah, callees: `callgrind_annotate --tree=calling callgrind.out.<pid>`
- call graph full tree: `callgrind_annotate --tree=both callgrind.out.<pid>`
  
  To use `callgrind_control` application must be executed with `valgrind --tool=callgrind ...`
- get realtime statistics: `callgrind_control -s <pid>`
- get stack/back traces of running application: `callgrind_control -b <pid>`
