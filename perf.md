perf reads jit symbol info from /tmp/perf-... If you profile large builds like the one that Mono has it's a problem because PIDs get wrapped and the performance data gets lost.

To fix this you need to increase `/proc/sys/kernel/pid_max` (works only on 64-bit, on 32 bits you're stuck with 32768)

Record with call graph information. (without it it looks like a flat eeeew)
```
perf record -g mono --jitmap ...
```

Show the actual thing.
```
perf report
```
