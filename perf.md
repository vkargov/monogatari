Gathering perf
==
perf reads jit symbol info from /tmp/perf-... If you profile large builds like the one that Mono has it's a problem because PIDs get wrapped and the performance data gets lost.

To fix this you need to increase `/proc/sys/kernel/pid_max` (works only on 64-bit, on 32 bits you're stuck with 32768)

Record with call graph information. (without it it looks like a flat eeeew)
```
MONO_DEBUG=disable_omit_fp MONO_ENV_OPTIONS=--jitmap perf record -g mono ...
```
for the whole mono build reducing the profiling frequency down to `-F 50` is a good idea for an 8Gb machine, otherwise `prof report` may run out of memory.

High level view:
```
perf report --hierarchy
```

Normal view:
```
perf report
```

There is an unsolved issue that `--call-graph dwarf` does not play well with the `/tmp/perf\*` files generated JITted code which forces one to use the default `fp` unwinder instead. That makes the report less pretty than desired (and breaks libc traces in particular), but it should still have enough information to perform analysis with a trace obtained like that.

Current build
==

Tool breakdown
--
* 67.8% mono
* 7.6% cc
* 6.9% libtool
* 5.5% make
* 2.6% bash
* 2.3% as
* 0.5% cmake
* 0.5% grep
* 0.5% install
* 5.8% kernel+other

Mono breakdown
--



Miscellanea
==
Some basic perf source code bearings
--
* `perf_evlist__prepare_workload` forks out the command we'll be profiling and starts waiting on a "cork" fd
* `perf_session__write_header` writes header
* `record__synthesize` writes system info like mmap and modules (?)
* `perf_evlist__start_workload` signals the waiting child to start the process we wanted to profile.
* `perf_event_output` kernel side event recording
* `_unwind__get_entries` does unwinding
* `get_entries` @ `unwind-libunwind-local.c` main unwinding loop

Misc
--
debugging perf data: `perf report -D`

The way V8 handles it: https://github.com/v8/v8/wiki/V8-Linux-perf-Integration
`perf inject --jit`?
Patch: https://lkml.org/lkml/2015/2/10/568
