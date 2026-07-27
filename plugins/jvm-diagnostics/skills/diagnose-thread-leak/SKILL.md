---
name: diagnose-thread-leak
description: Diagnose a suspected thread leak in a running Java process using only serviceability tools (jcmd/jstat), before ever opening source code. Use when a process's live thread count looks abnormally high or keeps climbing over time, or the user reports a JVM process "deteriorating" under sustained load.
---

# Diagnose a thread leak

Requires a PID (use the `find-java-processes` skill / `jps -lv` first if you don't have one). Stay in the observation domain — jcmd, jstat, and controlled requests — for as long as possible. Only open source once the tooling has pointed at a specific class/method/lock; reading code earlier than that skips past the diagnostic value of the tools themselves.

## 1. Confirm the trend, don't rely on one snapshot

A single thread count means nothing on its own — take at least two samples spaced apart and check the direction:

```bash
jcmd <pid> Thread.print | grep -c '^"'
# ... wait / do other work ...
jcmd <pid> Thread.print | grep -c '^"'
```

Also pull `jstat -gcutil <pid>` alongside it. This distinguishes a thread leak from a heap leak:
- **Thread leak signature**: thread count rises monotonically (never drops), while old-gen occupancy and full-GC count stay flat or even dip after a young GC.
- **Heap leak signature**: old-gen occupancy ratchets upward over time and full GCs increase, often independent of thread count.

If thread count only goes up and heap looks stable, treat it as a thread leak and continue below.

## 2. Take a full thread dump and cluster by name pattern

```bash
jcmd <pid> Thread.print > /tmp/threaddump-<pid>.txt
```

Don't eyeball 100+ threads one by one — normalize names (strip numeric suffixes) and count:

```bash
grep '^"' /tmp/threaddump-<pid>.txt | sed -E 's/^"([^"]*)".*/\1/' | sed -E 's/[0-9]+/#/g' | sort | uniq -c | sort -rn
```

Known-benign pools (Tomcat `http-nio-*-exec-*`, `GC Thread#`, `G1 Conc*`, JFR/JDWP/RMI/Jolokia threads, HikariCP housekeeper, etc.) will show up with small, bounded counts. What you're looking for is a **generic pool name** (commonly `Thread-N` — the default name the JVM gives an unnamed `new Thread(...)`) present in unusually large numbers. That's the signature of ad-hoc, unmanaged thread creation rather than a bounded executor/pool.

## 3. Inspect the stack of the suspect cluster

```bash
awk '/^"Thread-/{p=1} p{print} /^$/{if(p){print "---"; p=0}}' /tmp/threaddump-<pid>.txt | head -60
```

Check for:
- **Identical stack traces** across many threads — means they all originate from the same code path, not organic variety.
- **Thread state** — `WAITING (on object monitor)` inside `Object.wait()` with no timeout means the thread will sit there forever unless something else calls `notify()`/`notifyAll()` on that monitor.
- **The locked monitor address**, e.g. `- locked <0x000000060181b0f0> (a java.lang.Object)`. If every leaked thread reports the *same* address, they're all piling up on one shared lock object — a strong signal that whatever's supposed to release them isn't running, or was never wired up.

Use `jcmd <pid> Thread.print -l` if the plain dump doesn't show the "waiting on" object reference.

## 4. Check elapsed time spread to confirm continuous accumulation

```bash
grep -n "elapsed=" /tmp/threaddump-<pid>.txt | grep "Thread-" | awk -F'elapsed=' '{print $2}' | awk -F's' '{print $1}' | sort -n | sed -n '1p;$p'
```

If the oldest leaked thread's elapsed time is close to the process's total uptime (`jcmd <pid> VM.uptime`) and the newest is recent, the threads have been accumulating steadily since startup — not created in one burst (e.g. a deploy/restart storm) or a one-off batch job.

## 5. Corroborate with natural traffic, don't assume an exact delta

If the leaked stack trace names a request-handling class (e.g. a `Controller`), you still want more than the stack trace alone before pointing at an endpoint — but avoid firing a bulk burst of synthetic requests to force a clean before/after delta. Generating load against a system is a decision with its own blast radius and shouldn't be a routine step in a diagnostic skill.

Instead, take several spaced thread-count samples (`jcmd <pid> Thread.print | grep -c '^"'`, a few seconds to tens of seconds apart) while whatever traffic is already flowing continues, and check for **sustained, monotonic growth** in the generic-pool count. That's a solid corroborating signal without needing to generate traffic yourself.

Also don't assume the trigger is a deterministic one-request-in-one-thread-out relationship. A leak can be gated behind a condition (a probabilistic check, a cache-miss branch, a time-based trigger) so the growth is real but not 1:1 with request count — in that case an exact +N delta for N requests will *not* materialize even though the leak is genuine, and chasing that exact match can wrongly rule out the correct endpoint. Sustained upward trend across samples is the more robust signal; treat an exact-delta match as a nice-to-have bonus confirmation when it happens to hold, not a requirement.

## 6. Only now, open the source

With a specific class, method, and line number in hand from the stack trace (e.g. `Controller.lambda$someMethod$0(Controller.java:119)`), read just that method to see why the lock is never released (commonly: a `wait()` with no matching `notify()`/`notifyAll()` anywhere in the class, or a `synchronized` block guarding a `Thread` that's spawned per-call instead of reused from a pool).

## Why this order matters

This project (see root `CLAUDE.md`) is a JVM observability/diagnostics demo — the point is exercising jcmd/JFR/Jolokia-style tooling, not code review. Following steps 1-5 before step 6 also means the diagnosis is falsifiable at every step (trend confirmed, cluster identified, growth corroborated) rather than a source-reading guess that happens to match a symptom.

Deliberately not included: firing a bulk burst of synthetic requests to force an exact +N delta. That technique looks rigorous, but it both (a) generates load against a system as a routine diagnostic step, which is a decision with its own blast radius, and (b) assumes a deterministic one-request-in-one-thread-out relationship that seldom holds — real leaks are often gated behind a condition (probabilistic, cache-miss-dependent, time-based), so the clean delta this technique looks for may simply never appear even when the diagnosis is correct. Sustained, monotonic growth across spaced natural-traffic samples is a more robust and lower-risk signal.
