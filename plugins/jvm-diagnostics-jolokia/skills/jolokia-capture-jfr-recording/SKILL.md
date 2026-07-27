---
name: jolokia-capture-jfr-recording
description: Start, check, and dump a JDK Flight Recorder (JFR) recording on a remote JVM through a Jolokia MCP server's DiagnosticCommand MBean (JFR.start/JFR.check/JFR.dump equivalents) — file-based, requires separate access to fetch the resulting .jfr file off the target's filesystem. Use when you have Jolokia access to a remote JVM AND some other way to retrieve a file from that host (it's actually local, or you have docker cp/kubectl cp/SSH). If you have Jolokia access ONLY (no filesystem access to the target at all), use jolokia-stream-jfr-recording instead.
---

# Capture a JFR recording via Jolokia (file-based)

Requires a connected Jolokia MCP server pointed at the target JVM, with the `com.sun.management:type=DiagnosticCommand` MBean whitelisted. See `jolokia-run-diagnostic-command` for the calling convention used below.

**Read this first:** `JFR.start`'s `filename=` option, and `JFR.dump`'s output, are written to a file **on the target JVM's own filesystem** — not on the machine running the Jolokia MCP client, and not returned as data over JMX. Confirmed by testing: starting a recording with `filename=/tmp/foo.jfr` produces no such file on the client host, even after the recording duration elapses. Only use this file-based skill when you have an independent way to retrieve that file afterwards:

- The target JVM is actually running on your local machine (the file lands in your own `/tmp`).
- The target is in a container/pod and you have `docker cp`/`kubectl cp` access to it.
- You have SSH/other file access to the target host.

**If none of those apply — Jolokia is your only access to the target — use `jolokia-stream-jfr-recording` instead**, which reads the recording data back over the JMX stream itself and needs no separate filesystem access.

## 1. Check for an existing recording first

Don't stack a second recording on top of one already running:

```
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / jfrCheck, args: null
```

Note this operation still requires the 1-arg array shape — `args: null` (or `args: [""]`, confirmed equivalent-ish but `null` is cleaner), not `args: []`. If it reports an active recording, either use that one (`JFR.dump name=<existing-name> filename=...`) or stop it before starting a new one.

## 2. Start a timed recording

```
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / jfrStart,
  args: ["name=<label>", "settings=profile", "duration=<N>s", "filename=/tmp/<app>-<label>.jfr"]
```

- `settings=profile` — the higher-detail built-in profile (vs. `default`, lighter-weight, meant for always-on monitoring). Use `profile` for a deliberate, short diagnostic session.
- `duration=<N>s` — e.g. `120s`. The JVM auto-stops and auto-dumps to `filename` when it elapses.
- `filename=` — remember: this path is resolved **on the target's filesystem**. Use a path you'll actually be able to retrieve afterwards.
- Pick a `name` you'll recognize if you need to reference the recording again (`jfrCheck`, or `JFR.stop name=<label>`).

## 3. Wait for it to finish, then confirm it landed — on the target

```
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / jfrCheck, args: ["name=<label>"]
```

Once the recording no longer shows as running, retrieve the file using whichever access path applies:

```bash
# local target
ls -la /tmp/<app>-<label>.jfr

# containerized target
docker cp <container>:/tmp/<app>-<label>.jfr ./
kubectl cp <namespace>/<pod>:/tmp/<app>-<label>.jfr ./
```

## 4. Analyze the recording

Once the `.jfr` file is on a machine you can read files from, this is identical to the local-`jcmd` workflow — see the `analyze-jfr-recording` skill (from the `jvm-diagnostics-cli` plugin) for command-line triage with `jfr summary`/`jfr print`, or open it in JMC directly.

## Why this needs separate filesystem access

`JFR.start`'s `filename` option and `JFR.dump` both write through `java.io.File` on the JVM process's own host — that's a property of the diagnostic command itself, not something Jolokia can route around. Jolokia/JMX only carries the *command* to the target; it doesn't tunnel the resulting file back. `jolokia-stream-jfr-recording` exists specifically to avoid this constraint by reading the recording's bytes back through a different MBean (`jdk.management.jfr:type=FlightRecorder`'s stream operations) instead of a filesystem path.