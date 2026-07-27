---
name: analyze-jfr-recording
description: Do a quick command-line triage of a JDK Flight Recorder (.jfr) file to find CPU hot paths, allocation pressure, and lock/IO contention — and to tell real application signal apart from environment noise (idle NIO acceptor threads, an attached IDE debugger's polling, Tomcat housekeeping). Use right after capture-jfr-recording produces a .jfr file, or whenever the user hands you an existing recording and asks for a performance read.
---

# Analyze a JFR recording (command-line triage)

Requires a `.jfr` file (e.g. from the `capture-jfr-recording` skill). This is a first-pass triage to decide *where to look next* — it is not a replacement for opening the file in JMC when you need the full timeline/flame-graph view, but it's often enough to answer "is there actually a performance problem here, and if so, what kind."

## 0. Locate the `jfr` binary

`jfr` isn't always on `PATH` even when a JDK is installed — resolve it explicitly against the same JDK the target process runs on (or any modern JDK; the tool reads the file format, not the running process):

```bash
/usr/libexec/java_home -V                    # macOS: list installed JDKs
JFR=<jdk-home>/bin/jfr
```

## 1. Start with the summary — event counts by type

```bash
$JFR summary <file>.jfr
```

This alone tells you a lot before reading a single stack trace:

- **`jdk.ExecutionSample` count** — the CPU-sampling signal. If this is a large number, there's real CPU-bound work to profile (go to step 2). If it's **zero or near-zero**, the app was mostly idle/blocked during the recording window, not CPU-bound — don't go hunting for a hot method that isn't there.
- **`jdk.ObjectAllocationSample` count** — allocation pressure signal (step 3).
- **`jdk.JavaMonitorWait` / `jdk.JavaMonitorEnter` / `jdk.SocketRead` / `jdk.SocketWrite` / `jdk.FileRead` / `jdk.FileWrite`** — lock contention and I/O wait signals (step 4). Zero across the board means the recording window simply didn't capture contention or I/O — not that the app has none, just that this window didn't show it.
- A recording dominated by very high counts of `jdk.NativeMethodSample` and `jdk.ThreadSleep` relative to everything else is a flag to check for environment noise before drawing any conclusion (step 5) — these are cheap, frequent events that can dwarf the signal you actually want.

## 2. If there are execution samples: find the hot stacks

```bash
$JFR print --events jdk.ExecutionSample --stack-depth 8 <file>.jfr \
  | grep -E '^\s+(your\.app\.package|other\.lib\.package)' \
  | sed -E 's/^\s+//; s/ line:.*//' \
  | sort | uniq -c | sort -rn | head -30
```

Tally which frames appear most often across samples — that's your approximate hot-path profile without opening JMC. Filter to the app's own package(s) first; if nothing from the app shows up, widen to library/framework packages.

## 3. If there are allocation samples: find hot allocation sites and types

```bash
$JFR print --events jdk.ObjectAllocationSample --stack-depth 8 <file>.jfr \
  | grep -E '^\s+(your\.app\.package)' | sed -E 's/^\s+//; s/ line:.*//' | sort | uniq -c | sort -rn
$JFR print --events jdk.ObjectAllocationSample <file>.jfr | grep 'objectClass' | sort | uniq -c | sort -rn
```

Look for a small number of call sites or types accounting for a disproportionate share of samples — that's the allocation-pressure lead.

## 4. If there's contention/IO data: inspect it directly

```bash
$JFR print --events jdk.JavaMonitorWait <file>.jfr | head -60
$JFR print --events jdk.SocketRead,jdk.SocketWrite <file>.jfr | head -60
```

Long `duration` values on monitor waits, or a cluster of them on the same `monitorClass`/thread name, point at a specific lock or queue. Match this up with what you already know about the app's threads/singletons if you've run `diagnose-thread-leak` / `diagnose-heap-leak` first — a monitor wait on a class you already flagged as suspect is corroborating evidence, not a coincidence.

## 5. Separate real signal from environment noise

Before treating any of the above as an app-performance finding, check whether it's actually the *environment* the JVM happens to be running in:

- **`jdk.NativeMethodSample` frames in `sun.nio.ch.Net.accept`, `sun.nio.ch.KQueue.poll`, `DatagramChannelImpl.receive0`, etc.** — these are typically idle server acceptor/poller threads parked waiting for a connection, not CPU work. High counts here usually just mean "the server had capacity to spare," not a problem.
- **`jdk.ThreadSleep` events attributed to an IDE/debugger thread** (e.g. a thread named `*Suspend Helper*`, stack trace running through `com.intellij.rt.debugger.agent.*` or similar) — this is the IDE's attached debugger polling on a fixed interval (commonly every 50ms), not application behavior. If the target JVM was started with a JDWP agent (`-agentlib:jdwp=...`, visible via `find-java-processes`), expect this noise and discount it.
- **Frames in `org.apache.tomcat.util.threads.*` / `NioEndpoint$Poller`** — Tomcat's own connector housekeeping, not request-handling application code.

Grep for how many samples/events actually land in the app's own package vs. everything else — a low ratio (e.g. tens of app frames out of thousands of total events) is itself informative: it usually means the recording window had light real traffic, or an attached debugger/IDE is drowning out the signal, rather than meaning "nothing is wrong." In that case the right next step is a longer recording, a heavier-traffic window, or detaching the debugger before re-recording — not declaring the app problem-free from a quiet window.

## 6. Report the shape, not just the numbers

Summarize as: what fraction of the recording is real app signal vs. environment noise, which event types actually had data (and which were empty), and — if a hot path/allocation site/contention point was found — the specific class/method it points to. If the window was too quiet or too noisy to conclude anything, say that explicitly rather than forcing a conclusion from thin data.
