---
name: inspect-java-process
description: Inspect a specific running Java process in more depth once you have its PID (e.g. from the find-java-processes skill) — uptime, JVM flags, CPU/memory usage, live thread count, heap/GC summary, and full thread dumps. Use when the user asks for more detail about a particular Java/JVM process, wants to check resource usage (threads, heap, CPU), or wants a thread dump.
---

# Inspect a Java process

Requires a PID (use the `find-java-processes` skill / `jps -lv` first if you don't have one).

## Quick health snapshot

```bash
jcmd <pid> VM.uptime
jcmd <pid> VM.flags
ps -o pid,ppid,%cpu,%mem,rss,etime,stat -p <pid>
```

- `VM.uptime` — how long the JVM has been up, in seconds.
- `VM.flags` — the effective JVM flags (heap sizing, GC algorithm, code cache, etc.), useful even when the process wasn't launched with them explicitly (JVM ergonomics fill in defaults).
- `ps` — OS-level view: CPU%, memory%, RSS (actual resident memory, usually more meaningful than `%mem`), elapsed wall-clock time, and process state (`S` sleeping/idle, `R` running, etc.).

**Locale gotcha:** on systems with a comma-decimal locale, `VM.uptime` (and `jstat` output) render the decimal separator as a comma, not a period — e.g. `92,448 s` means **92.448 seconds**, not "92 thousand 448." Misreading this as thousands-grouping makes a freshly-started process look like it's been up for days. Always cross-check against `ps ELAPSED`/`lstart` before drawing conclusions from an uptime number that looks implausible; if in doubt, take two `VM.uptime` samples a few seconds apart and confirm the delta matches real elapsed time.

## Heap / GC summary

```bash
jcmd <pid> GC.heap_info
```

Shows committed/used heap per generation (young/old for G1) and Metaspace usage — a fast way to spot heap growth or Metaspace bloat without a full histogram.

## Thread count and thread dump

```bash
# Just the count of live threads
jcmd <pid> Thread.print | grep -c '^"'

# Full thread dump (names, states, stack traces) — long output, redirect to a file for
# non-trivial thread counts:
jcmd <pid> Thread.print > /tmp/threaddump-<pid>.txt
```

A live thread count much higher than expected for the app's baseline load (a small Spring Boot MVC app at rest is typically 30-50 threads) is worth a closer look via the full dump — check thread names/states for unexpectedly large pools, threads stuck in `WAITING`/`BLOCKED`, or duplicated one-off worker threads that suggest a thread leak.

## Class histogram (heap object breakdown)

```bash
jcmd <pid> GC.class_histogram | head -30
```

Top of the list by instance count/bytes — useful to spot which type is accumulating if heap usage is growing.

`jcmd` only works on JVMs owned by the same user, and only for JVMs it can attach to (some hardened/container JVMs disable local attach).
