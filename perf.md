Gathering perf
==
perf reads jit symbol info from /tmp/perf-... If you profile large builds like the one that Mono has it's a problem because PIDs get wrapped and the performance data gets lost.

To fix this you need to increase `/proc/sys/kernel/pid_max` (works only on 64-bit, on 32 bits you're stuck with 32768)

Record with call graph information.
```
MONO_DEBUG=disable_omit_fp MONO_ENV_OPTIONS=--jitmap perf record -g mono ...
```
We use the default `-g` unwinder (`fp`) because `--call-graph dwarf` does not play well with the `/tmp/perf\*` files generated JITted code. That makes the report less pretty than desired (and breaks libc traces in particular), but it should still have enough information to perform analysis with a trace obtained like that.

for the whole mono build reducing the profiling frequency down to `-F 50` is a good idea for an 8Gb machine, otherwise `prof report` may run out of memory.

High level view:
```
perf report --hierarchy
```

Normal view:
```
perf report
```



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
* 5.8% other

Mono basic breakdown (relative %)
--
* 47.3% runtime
* 9.4% libc
* 8.8% syscalls
* 8.6% Microsoft.CodeAnalysis.CSharp.dll.so
* 4.8% Microsoft.CodeAnalysis.dll.so
* 3.0% mscorlib.dll.so
* 2.9% libpthread
* 0.7% System.Collections.Immutable.dll.so
* 14.7 JIT + other AOT

Mono most taxing functions (w/o children)
--
* 2.9% monoeg_g_hash_table_lookup_extended
* 1.3% mono_method_to_ir
* 1.1% mono_local_deadce
* 1.1% mono_metadata_type_hash
* 0.8% mono_conc_hashtable_lookup
* 0.8% mono_local_cprop
* 0.6% mono_generic_inst_equal_full
* 0.6% jit_info_table_add

TODO: compare runtime data with Instruments output

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
