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

## Getting stack traces from within glibc6.
Since in our approach we do the simple fp unwinding, we cannot use dwarf unwinding for libc, because it's built with `-fomit-frame-pointer` by default in ubuntu/debian. 

(I guess one could also patch `perf` to be selective about the unwinding mechanism based on the IP or smth, but gosh, at that point one would want to pick one of other more meaningful options than doing that)

Anyway, to get meaningful stack traces from glibc we need to build it with `-fno-omit-frame-pointer`.

To do that:

### Checkout:
`apt-get source libc6 && sudo apt-get build-dep libc6`

### Apply patch:
```
diff -ur glibc-2.24/io/Makefile ../glibc-2.24/io/Makefile
--- glibc-2.24/io/Makefile	2016-08-02 02:01:36.000000000 +0000
+++ ../glibc-2.24/io/Makefile	2017-07-27 22:11:06.826983086 +0000
@@ -94,11 +94,11 @@
 CFLAGS-ftw.c = $(uses-callbacks) -fexceptions
 CFLAGS-ftw64.c = $(uses-callbacks) -fexceptions
 CFLAGS-lockf.c = -fexceptions
-CFLAGS-posix_fallocate.c = -fexceptions
-CFLAGS-posix_fallocate64.c = -fexceptions
-CFLAGS-fallocate.c = -fexceptions
-CFLAGS-fallocate64.c = -fexceptions
-CFLAGS-sync_file_range.c = -fexceptions
+CFLAGS-posix_fallocate.c = -fexceptions -fomit-frame-pointer
+CFLAGS-posix_fallocate64.c = -fexceptions -fomit-frame-pointer
+CFLAGS-fallocate.c = -fexceptions -fomit-frame-pointer
+CFLAGS-fallocate64.c = -fexceptions -fomit-frame-pointer
+CFLAGS-sync_file_range.c = -fexceptions -fomit-frame-pointer
 
 CFLAGS-test-stat.c = -D_FILE_OFFSET_BITS=64 -D_LARGEFILE64_SOURCE
 CFLAGS-test-lfs.c = -D_LARGEFILE64_SOURCE
diff -ur glibc-2.24/Makeconfig ../glibc-2.24/Makeconfig
--- glibc-2.24/Makeconfig	2017-07-27 22:58:02.000000000 +0000
+++ ../glibc-2.24/Makeconfig	2017-07-27 21:12:24.780676747 +0000
@@ -830,7 +830,7 @@
 +cflags	:= $(default_cflags)
 endif	# $(+cflags) == ""
 
-+cflags += $(cflags-cpu) $(+gccwarn) $(+merge-constants) $(+math-flags)
++cflags += $(cflags-cpu) $(+gccwarn) $(+merge-constants) $(+math-flags) -fno-omit-frame-pointer
 +gcc-nowarn := -w
 
 # Don't duplicate options if we inherited variables from the parent.
diff -ur glibc-2.24/misc/Makefile ../glibc-2.24/misc/Makefile
--- glibc-2.24/misc/Makefile	2017-07-27 22:58:02.000000000 +0000
+++ ../glibc-2.24/misc/Makefile	2017-07-27 21:48:28.672504407 +0000
@@ -86,7 +86,7 @@
 CFLAGS-select.c = -fexceptions -fasynchronous-unwind-tables
 CFLAGS-tsearch.c = $(uses-callbacks)
 CFLAGS-lsearch.c = $(uses-callbacks)
-CFLAGS-pselect.c = -fexceptions
+CFLAGS-pselect.c = -fexceptions -fomit-frame-pointer
 CFLAGS-readv.c = -fexceptions -fasynchronous-unwind-tables
 CFLAGS-writev.c = -fexceptions -fasynchronous-unwind-tables
 CFLAGS-preadv.c = -fexceptions -fasynchronous-unwind-tables
diff -ur glibc-2.24/nptl/Makefile ../glibc-2.24/nptl/Makefile
--- glibc-2.24/nptl/Makefile	2017-07-27 22:58:02.000000000 +0000
+++ ../glibc-2.24/nptl/Makefile	2017-07-27 22:13:37.991249944 +0000
@@ -182,8 +182,8 @@
 			-fasynchronous-unwind-tables
 CFLAGS-pthread_cond_wait.c = -fexceptions -fasynchronous-unwind-tables
 CFLAGS-pthread_cond_timedwait.c = -fexceptions -fasynchronous-unwind-tables
-CFLAGS-sem_wait.c = -fexceptions -fasynchronous-unwind-tables
-CFLAGS-sem_timedwait.c = -fexceptions -fasynchronous-unwind-tables
+CFLAGS-sem_wait.c = -fexceptions -fasynchronous-unwind-tables -fomit-frame-pointer
+CFLAGS-sem_timedwait.c = -fexceptions -fasynchronous-unwind-tables -fomit-frame-pointer
 
 # These are the function wrappers we have to duplicate here.
 CFLAGS-fcntl.c = -fexceptions -fasynchronous-unwind-tables
@@ -216,6 +216,9 @@
 CFLAGS-tst-thread_local1.o = -std=gnu++11
 LDLIBS-tst-thread_local1 = -lstdc++
 
+CFLAGS-pthread_rwlock_timedrdlock.c = -fomit-frame-pointer
+CFLAGS-pthread_rwlock_timedwrlock.c = -fomit-frame-pointer
+
 tests = tst-typesizes \
 	tst-attr1 tst-attr2 tst-attr3 tst-default-attr \
 	tst-mutex1 tst-mutex2 tst-mutex3 tst-mutex4 tst-mutex5 tst-mutex6 \
diff -ur glibc-2.24/sysdeps/posix/Makefile ../glibc-2.24/sysdeps/posix/Makefile
--- glibc-2.24/sysdeps/posix/Makefile	2016-08-02 02:01:36.000000000 +0000
+++ ../glibc-2.24/sysdeps/posix/Makefile	2017-07-27 21:37:52.363417329 +0000
@@ -9,3 +9,6 @@
 # Without NPTL, it's just private in librt.
 librt-routines += shm-directory
 endif
+
+CFLAGS-posix_fallocate.c := -fomit-frame-pointer
+CFLAGS-posix_fallocate64.c := -fomit-frame-pointer
diff -ur glibc-2.24/sysdeps/unix/sysv/linux/i386/Makefile ../glibc-2.24/sysdeps/unix/sysv/linux/i386/Makefile
--- glibc-2.24/sysdeps/unix/sysv/linux/i386/Makefile	2017-07-27 22:58:02.000000000 +0000
+++ ../glibc-2.24/sysdeps/unix/sysv/linux/i386/Makefile	2017-07-27 22:18:49.555797045 +0000
@@ -10,6 +10,7 @@
 CFLAGS-mmap.os += -fomit-frame-pointer
 CFLAGS-mmap64.o += -fomit-frame-pointer
 CFLAGS-mmap64.os += -fomit-frame-pointer
+CFLAGS-rtld-mmap.os += -fomit-frame-pointer
 endif
 
 ifeq ($(subdir),sysvipc)
diff -ur glibc-2.24/sysdeps/unix/sysv/linux/Makefile ../glibc-2.24/sysdeps/unix/sysv/linux/Makefile
--- glibc-2.24/sysdeps/unix/sysv/linux/Makefile	2016-08-02 02:01:36.000000000 +0000
+++ ../glibc-2.24/sysdeps/unix/sysv/linux/Makefile	2017-07-27 22:14:22.195327775 +0000
@@ -147,6 +147,7 @@
 CFLAGS-fork.c = $(libio-mtsafe)
 CFLAGS-getpid.o = -fomit-frame-pointer
 CFLAGS-getpid.os = -fomit-frame-pointer
+CFLAGS-mmap.c = -fomit-frame-pointer
 endif
 
 ifeq ($(subdir),inet)
```
Without it the build won't work. Probably need to fix/update the patch for the future versions.

### Build
`dpkg-buildpackage ...`

Now figure out how to make it as a local install for use only with the runtime...

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

Mono most taxing functions (w/o children, relative %)
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
