Gathering perf
==
perf reads jit symbol info from /tmp/perf-... If you profile large builds like the one that Mono has it's a problem because PIDs get wrapped and the performance data gets lost.

To fix this you need to increase `/proc/sys/kernel/pid_max` (works only on 64-bit, on 32 bits you're stuck with 32768)

Record with call graph information. (without it it looks like a flat eeeew)
```
MONO_DEBUG=disable_omit_fp perf record -g mono --jitmap ...
```

Show the actual thing.
```
perf report
```

Some basic perf 🐻ings
==

record
---

* `perf_evlist__prepare_workload` forks out the command we'll be profiling and starts waiting on a "cork" fd

* `perf_session__write_header` writes header

* `record__synthesize` writes system info like mmap and modules (?)

* `perf_evlist__start_workload` signals the waiting child to start the process we wanted to profile.

* `perf_event_output` kernel side event recording

* `_unwind__get_entries` does unwinding

* `get_entries` @ `unwind-libunwind-local.c` main unwinding loop

Misc
---
The way V8 handles it: https://github.com/v8/v8/wiki/V8-Linux-perf-Integration
`perf inject --jit`?
Patch: https://lkml.org/lkml/2015/2/10/568
