---
name: jolokia-diagnose-thread-leak
description: Diagnose a suspected thread leak in a JVM reachable only through a Jolokia MCP server (no local jcmd/SSH access), using MBean attribute reads and DiagnosticCommand's threadPrint. Use when a remote process's live thread count looks abnormally high or keeps climbing over time, and the only tooling available is Jolokia.
---

# Diagnose a thread leak via Jolokia

Requires a connected Jolokia MCP server pointed at the target JVM. Same observe-first posture as the local-`jcmd` version of this skill (`jvm-diagnostics-cli`'s `diagnose-thread-leak`): stay in the observation domain for as long as possible, only open source once the tooling has pointed at a specific class/method/lock. See `jolokia-run-diagnostic-command` for the DiagnosticCommand calling convention used below.

## 1. Confirm the trend, don't rely on one snapshot

A single thread count means nothing on its own — sample the cheap attribute at least twice, spaced apart:

```
readMBeanAttribute: java.lang:type=Threading / ThreadCount
# ... wait / do other work ...
readMBeanAttribute: java.lang:type=Threading / ThreadCount
```

Also pull `TotalStartedThreadCount` (monotonically increasing since JVM start, never decreases) alongside it — if `ThreadCount` (currently-live) and `TotalStartedThreadCount` (cumulative) are climbing at close to the same rate, threads are being created and never terminating, which is the leak signature; if `TotalStartedThreadCount` climbs but `ThreadCount` stays flat, threads are being created and properly cleaned up (healthy pool churn).

Cross-check against heap to distinguish a thread leak from a heap leak, the same way the CLI skill uses `jstat -gcutil`:

```
readMBeanAttribute: java.lang:type=Memory / HeapMemoryUsage
readMBeanAttribute: java.lang:name=G1 Old Generation,type=GarbageCollector / CollectionCount
```

- **Thread leak signature**: `ThreadCount` rises monotonically, while heap `used` and Old-Gen `CollectionCount` stay flat or dip after a young GC.
- **Heap leak signature**: heap `used` ratchets upward over time and Old-Gen `CollectionCount` increases, often independent of thread count.

If thread count only goes up and heap looks stable, treat it as a thread leak and continue below.

## 2. Take a full thread dump and cluster by name pattern

```
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / threadPrint, args: null
```

This routinely exceeds the tool result size limit for a process with 100+ threads and gets redirected to a local file (see `jolokia-run-diagnostic-command` step 4). Take a second dump the same way after waiting 30-60s, then work from both saved files with `grep`/`sed`/`awk`, same as you would local thread dumps:

```bash
grep '^"' <saved-file-1> | sed -E 's/^"([^"]*)".*/\1/' | sed -E 's/[0-9]+/#/g' | sort | uniq -c | sort -rn
grep '^"' <saved-file-2> | sed -E 's/^"([^"]*)".*/\1/' | sed -E 's/[0-9]+/#/g' | sort | uniq -c | sort -rn
```

Diff the two counts by cluster name. The cluster(s) whose count grew are your suspects, not whichever is biggest in one snapshot. Known-benign pools (Tomcat `http-nio-*-exec-*`, `GC Thread#`, `G1 Conc*`, JFR/JDWP/RMI/Jolokia threads, HikariCP housekeeper, etc.) are normally flat or bounded across two samples even if their absolute count looks large.

**Don't assume the leaked cluster has a "generic" name.** A JVM-default name like `Thread-N` (unnamed `new Thread(...)`) is one signature of ad-hoc, unmanaged thread creation, but a leak can just as easily show up under an ordinary, framework-assigned pool name — e.g. an `ExecutorService` created per-request instead of reused, a scheduled-task pool whose tasks are never cancelled, or a resource reaper duplicated each time something is opened but not closed. Treat "which cluster is growing" as the signal; a well-named cluster that keeps growing is just as much a leak as an anonymous one.

## 3. Inspect the stack of the suspect cluster

```bash
awk -v pat='^"<suspect-cluster-prefix>' '$0 ~ pat{p=1} p{print} /^$/{if(p){print "---"; p=0}}' <saved-file-2> | head -60
```

(substitute the normalized cluster name/prefix you found above, e.g. `Thread-` or whatever pool name is growing)

Check for:
- **Identical or near-identical stack traces** across many threads in the cluster — they all originate from the same code path, not organic variety.
- **Thread state**, and what it implies:
  - `WAITING`/`TIMED_WAITING (on object monitor)` inside `Object.wait()` with no timeout — the thread sits forever unless something else calls `notify()`/`notifyAll()`.
  - `BLOCKED (on object monitor)` waiting to *enter* a `synchronized` block — something else is holding that lock far longer than expected (or forever).
  - `RUNNABLE` parked inside a blocking I/O or library call (socket read/connect, queue `take()`) with no visible timeout — stuck waiting on an external resource that never arrives.
  - `TIMED_WAITING` in a scheduled-executor's park loop, repeated across many threads — often means tasks are scheduled faster than a fixed-size pool can retire them.
- **A shared lock/monitor address**, e.g. `- locked <0x000000060181b0f0> (a java.lang.Object)` or `- waiting to lock <0x...>`. If many leaked threads reference the *same* address, they're piling up on one shared resource.

Don't assume the mechanism in advance — let the state and stack tell you which shape applies. If the plain dump doesn't show the "waiting on"/lock-owner reference, rerun with locks enabled:

```
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / threadPrint, args: ["-l"]
```

## 4. Check elapsed time spread to confirm continuous accumulation

Thread dump text includes `elapsed=<seconds>` per thread. Compare the oldest leaked thread's elapsed time against the process's total uptime:

```
readMBeanAttribute: java.lang:type=Runtime / Uptime      # milliseconds, always available and locale-safe
```

```bash
grep -n "elapsed=" <saved-file-2> | grep "<suspect-cluster-prefix>" | awk -F'elapsed=' '{print $2}' | awk -F's' '{print $1}' | sort -n | sed -n '1p;$p'
```

If the oldest leaked thread's elapsed time is close to the process's total uptime and the newest is recent, threads have been accumulating steadily since startup — not created in one burst (a deploy/restart storm) or a one-off batch job.

## 5. Corroborate with natural traffic, don't force synthetic load

If the leaked stack trace names a request-handling class (e.g. a `Controller`), take several spaced `ThreadCount` samples while whatever traffic is already flowing continues, and check for **sustained, monotonic growth**. Avoid firing a bulk burst of synthetic requests to force a clean before/after delta — generating load against a system is a decision with its own blast radius and shouldn't be a routine diagnostic step, and a leak can be gated behind a non-deterministic condition (probabilistic check, cache-miss branch, time-based trigger) such that an exact +N-requests-to-+N-threads delta never appears even when the leak is real. Sustained upward trend across samples is the more robust signal.

## 6. Only now, open the source

With a specific class, method, and line number in hand from the stack trace, plus the thread-state/lock evidence from step 3, read just that method to see why threads accumulate instead of terminating or returning to a pool. The stack/state combination narrows the *category* (unreleased lock, unbounded pool/queue, stuck external call, per-call resource never reused) — the exact bug still requires reading that specific code.

## Why Jolokia instead of local jcmd here

Use this skill instead of the local-`jcmd` `diagnose-thread-leak` skill specifically when the target JVM isn't reachable via local process attach or SSH — e.g. it's in a container/pod with only its Jolokia HTTP endpoint exposed. The diagnostic logic is identical; only the transport differs (HTTP-via-Jolokia MBean calls instead of local `jcmd`/`jstat`), and large text output (a full thread dump) needs an extra file-read step since it can't be piped directly the way a local `jcmd` invocation's stdout can.
