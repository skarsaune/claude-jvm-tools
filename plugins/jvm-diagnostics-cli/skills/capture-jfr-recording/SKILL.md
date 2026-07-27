---
name: capture-jfr-recording
description: Capture a JDK Flight Recorder (JFR) recording from a running Java process using only jcmd, for profiling suspected performance issues (CPU hot paths, allocation pressure, lock contention, slow I/O). Use when jstack/jcmd snapshots (diagnose-thread-leak, diagnose-heap-leak) have narrowed things down but you need timeline/sampling data instead of point-in-time counts, or when the user directly asks to "profile" the app or "take a flight recording."
---

# Capture a JFR recording via jcmd

Requires a PID (use the `find-java-processes` skill / `jps -lv` first if you don't have one). No extra agent or JVM restart needed — JFR is built into the JDK and `jcmd` can start/stop/dump it on a JVM that's already running.

## 1. Check for an existing recording first

Don't stack a second recording on top of one already running (e.g. left over from a previous session, or JMC's own default recording if JMC is attached).

```bash
jcmd <pid> JFR.check
```

If it reports an active recording, either use that one (`JFR.dump name=<existing-name> filename=...`) or stop it before starting a new one, rather than layering recordings.

## 2. Start a timed recording

```bash
jcmd <pid> JFR.start name=<label> settings=profile duration=<Ns> filename=/tmp/<app>-<pid>-<label>.jfr
```

- `settings=profile` — the higher-detail built-in profile (vs. `default`, which is lighter-weight and meant for always-on production monitoring). Use `profile` for a deliberate, short diagnostic session like this.
- `duration=<Ns>` — e.g. `120s`. With a fixed duration, the JVM auto-stops and auto-dumps to `filename` when it elapses — no need to babysit it with a manual `JFR.stop`.
- Store the file under `/tmp/` (or another scratch location) — it's a diagnostic artifact, not something that belongs in the repo.
- Pick a `name` you'll recognize if you need to reference the recording again before it finishes (`JFR.check`, `JFR.stop name=<label>`).

Recording for 60-120s under real traffic is usually enough to get a meaningful method-sampling profile; longer if the suspected issue is intermittent or only shows up under sustained load.

## 3. Wait for it to finish, then confirm the file landed

A fixed-duration recording finishes on its own — don't manually stop it early unless you need the data sooner. To wait without polling manually, use a loop that checks for the dump:

```bash
until [ -f /tmp/<app>-<pid>-<label>.jfr ]; do sleep 5; done
```

Then confirm:

```bash
jcmd <pid> JFR.check        # should report "no recordings" once auto-dumped, or show it as stopped
ls -la /tmp/<app>-<pid>-<label>.jfr
```

## 4. Analyze the recording

Two options, depending on what's available:

- **JMC (JDK Mission Control)** — if it's installed/running (check via `find-java-processes`), open the `.jfr` file directly (File → Open File). This gives the full UI: hot methods, allocation profiles, lock contention, GC pauses, an event timeline — the natural tool for this data.
- **`jfr print`** (bundled with the JDK, no GUI needed) — for a quick command-line look without opening JMC:

```bash
jfr summary /tmp/<app>-<pid>-<label>.jfr                          # event counts/types overview
jfr print --events jdk.ExecutionSample /tmp/<app>-<pid>-<label>.jfr | head -100   # sampled stacks
```

`jdk.ExecutionSample` events are the CPU-profiling signal (periodic stack samples) — tallying the top frame across many samples approximates a hot-path profile without a GUI.

## Why jcmd over other capture methods

`jcmd JFR.start` works against any already-running JVM the current user can attach to, with no javaagent, no restart, and no extra dependency — the same low-friction, observe-first posture as the other skills in this set. It's also exactly what this repo's `jolokia-access.xml` whitelists the `FlightRecorder` MBean for, so this is the intended diagnostic path here, just driven from the command line instead of over JMX/HTTP.
