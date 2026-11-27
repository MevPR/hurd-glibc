# hurd-glibc

## Purpose

Fixes to the GNU C Library's Hurd port for running Debian GNU/Hurd x86_64
on several processors.

This is the glibc part of [mach-SMP](https://github.com/MevPR/mach-SMP),
which lets GNU Mach run ordinary tasks on every CPU (boot option
`smp_pset=default`).  Two bugs in glibc's Hurd code only show up with
real parallelism; one of them corrupts the heap of every multithreaded
program.

Base: glibc from sourceware.org/git/glibc.git (branch `upstream`).  The
original glibc README is unchanged in [`README`](README).

## Related repositories

The project is split into three repositories that belong together:
running Debian GNU/Hurd on several CPUs needs the fixes from all three.

| Repository | Component |
|---|---|
| [mach-SMP](https://github.com/MevPR/mach-SMP) | GNU Mach kernel (SMP fixes, `smp_pset=default`, tests); investigation notes, tools and test scripts for the whole project; the netdde and rumpkernel patches |
| [hurd-SMP](https://github.com/MevPR/hurd-SMP) | Hurd libraries and servers (libports, libfshelp, libpager, pflocal, console-client) |
| **hurd-glibc** (this one) | glibc's Hurd port (fork deadlock; `multiple_threads` for glibc ≤ 2.42) |
| [chromium-hurd](https://github.com/MevPR/chromium-hurd) | Chromium for the Hurd, used as the acceptance test: it opens and renders https://www.duckduckgo.com on 2 CPUs |

## Status

| | |
|---|---|
| **Fixes** | 2 patches on current upstream master (branch `main` = `hurd-smp`); the `multiple_threads` one only matters for glibc ≤ 2.42 |
| **Tested** | on Debian GNU/Hurd amd64 (glibc 2.41, snapshot 20250807) through equivalent binary patches of `libpthread.so.0.3` and `libc.so.0.3`, with static servers relinked with the same change: a multithreaded malloc test that failed every run on 2 CPUs passes every run, and the full Hurd stress test passes 6 times out of 6.  Not yet tested as a glibc rebuild from these sources. |
| **Tested on glibc 2.43** | the fork fix, as a binary patch of `libc.so.0.3` of Debian's glibc 2.43-7 (`binpatch.py libc243-fork` in mach-SMP): the first 4 GB boot on 2 CPUs hung in runlevel S without it; with it, about 15 boots on 2 CPUs had no fork hang, and Chromium opens and renders https://www.duckduckgo.com |
| **Upstream** | prepared for libc-alpha@sourceware.org (Cc bug-hurd@gnu.org), **not yet sent** |

## What was fixed

- **Heap corruption in every multithreaded program on SMP (glibc ≤ 2.42).**
  Since 4eef002328 ("x86: Remove catomic_* locking primitives", September
  2025, in 2.43) nothing reads `multiple_threads` any more and the
  corruption is gone; this patch is kept for older branches only.  On x86,
  glibc's `catomic_*` operations, used by malloc for its fastbins, only
  take the `lock` prefix when `tcbhead_t.multiple_threads` is set in the
  current thread's control block.  On the Hurd only `_hurd_tls_init` set
  it, for the main thread; threads created by `pthread_create` got a
  freshly cleared TCB from `_dl_allocate_tls`, and `_hurd_tls_new` left
  the field at 0.  All threads but the main one therefore ran these
  compare-and-exchange operations without `lock`: atomic on a single
  processor, not on several.  A multithreaded malloc/free test aborts
  within seconds on 2 CPUs ("malloc_consolidate(): unaligned fastbin
  chunk detected", "corrupted size vs. prev_size in fastbins"), and Hurd
  servers such as ext2fs corrupted their heaps.  `_hurd_tls_new` now sets
  the field (x86_64 and i386).
  (`hurd: Set multiple_threads in the TCB of new threads`)
- **fork deadlock between the forking thread and the signal thread.**
  `_Fork` resumed the other threads, the signal thread among them, before
  reading the signal thread's state for the child.  If the signal thread
  was meanwhile delivering a signal (e.g. SIGCHLD) to the forking thread,
  each thread ended up waiting for the other forever; seen as a sporadic
  hang of `/bin/sh` at boot.  The threads are now resumed only after that
  state was read.
  (`hurd: fork: fetch the signal thread state before resuming it`)

The investigation is in
[mach-SMP: smp-dev/notes/hang-fork-suspend.md](https://github.com/MevPR/mach-SMP/blob/main/smp-dev/notes/hang-fork-suspend.md);
the test program is
[mtstress.c](https://github.com/MevPR/mach-SMP/blob/main/smp-dev/stress/mtstress.c).

## Branches

| Branch | Content |
|---|---|
| `upstream` | mirror of sourceware.org/git/glibc.git master |
| `main` | the fix series + this README |
| `hurd-smp` | the fix series alone, as prepared for libc-alpha |
| `hurd-multiple-threads`, `hurd-fork-sigthread` | one fix each, on `upstream` |

Work by Magnus Reimann, done together with Claude (an AI model by
Anthropic).  glibc is free software of the GNU project.
