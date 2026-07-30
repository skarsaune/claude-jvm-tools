---
name: jolokia-capture-jfr-recording
description: Start, check, and dump a JDK Flight Recorder (JFR) recording on a remote JVM through a Jolokia MCP server's DiagnosticCommand MBean (JFR.start/JFR.check/JFR.dump equivalents) — file-based, requires separate access to fetch the resulting .jfr file off the target's filesystem. Only use this when the target is genuinely local to you or you have SSH onto the host — NOT for containers/pods you can only reach via Jolokia, where docker cp (needs a local Docker daemon) and kubectl cp (needs tar inside the container, absent from most hardened images) usually won't work. Default to jolokia-stream-jfr-recording instead, which needs nothing beyond the Jolokia connection you already have.
---

# Capture a JFR recording via Jolokia (file-based)

Requires a connected Jolokia MCP server pointed at the target JVM, with the `com.sun.management:type=DiagnosticCommand` MBean whitelisted. See `jolokia-run-diagnostic-command` for the calling convention used below.

**Read this first:** `JFR.start`'s `filename=` option, and `JFR.dump`'s output, are written to a file **on the target JVM's own filesystem** — not on the machine running the Jolokia MCP client, and not returned as data over JMX. Confirmed by testing: starting a recording with `filename=/tmp/foo.jfr` produces no such file on the client host, even after the recording duration elapses. Separately, an absolute `/tmp/...` path can also fail to write *on the target itself* in hardened/containerized images (read-only root filesystem, or a `/tmp` with narrowed permissions) — see the note on `filename=` in step 2 for why a bare relative filename (resolving to the JVM's CWD) is the safer default.

**Default to `jolokia-stream-jfr-recording` instead** — it needs nothing beyond the Jolokia connection itself. Only reach for this file-based skill in the narrow case where you have an independent, *actually working* way to pull that file off the target host afterwards:

- The target JVM is genuinely running on your local machine (the file lands in your own `/tmp`) — this is the realistic case for this skill.
- You have SSH/other real file access to the target host.

Two options that sound like they'd work but usually don't for the containerized/Jolokia-only targets this plugin is meant for:

- `docker cp` requires a local Docker daemon/socket you can talk to — if you're reaching the target only via a Jolokia HTTP endpoint (a remote pod, a locked-down container), you almost certainly don't have that.
- `kubectl cp` shells out to `tar` *inside the target container* to build the archive — hardened/distroless images (the kind this plugin targets, e.g. BellSoft Liberica base image) typically don't ship a shell or `tar`, so it fails even with valid `kubectl` access and RBAC.

If neither local access nor real SSH applies, **use `jolokia-stream-jfr-recording`**, which reads the recording data back over the JMX stream itself and needs no separate filesystem access.

## 1. Check for an existing recording first

Don't stack a second recording on top of one already running:

```
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / jfrCheck, args: null
```

Note this operation still requires the 1-arg array shape — `args: null` (or `args: [""]`, confirmed equivalent-ish but `null` is cleaner), not `args: []`. If it reports an active recording, either use that one (`JFR.dump name=<existing-name> filename=...`) or stop it before starting a new one.

## 2. Start a timed recording

```
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / jfrStart,
  args: ["name=<label>", "settings=profile", "duration=<N>s", "filename=<app>-<label>.jfr"]
```

`args` must be a **flat** array — one string per option, not `[["name=<label>", "settings=profile"]]`. Nesting here doesn't throw an obvious argument-shape error; it fails as `Could not find file 'profile]'`, which looks like a filesystem problem. See `jolokia-run-diagnostic-command` step 1 if you hit that.

- `settings=profile` — the higher-detail built-in profile (vs. `default`, lighter-weight, meant for always-on monitoring). Use `profile` for a deliberate, short diagnostic session.
- `duration=<N>s` — e.g. `120s`. The JVM auto-stops and auto-dumps to `filename` when it elapses.
- `filename=` — remember: this path is resolved **on the target's filesystem**. Use a **relative filename with no `/tmp` prefix** (resolves to the JVM's CWD) as the default, not an absolute `/tmp/...` path. Hardened/containerized images commonly run with a read-only root filesystem or a `/tmp` whose permissions were deliberately narrowed during hardening, while the CWD (e.g. `WORKDIR` in the Dockerfile) is far more likely to be writable — an absolute `/tmp/...` path is a common way for this whole command to silently produce no file. Only use an absolute path (`/tmp/...` or otherwise) if you've independently confirmed that directory is writable on this specific target.
- Pick a `name` you'll recognize if you need to reference the recording again (`jfrCheck`, or `JFR.stop name=<label>`).

## 3. Wait for it to finish, then confirm it landed — on the target

```
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / jfrCheck, args: ["name=<label>"]
```

Once the recording no longer shows as running, retrieve the file using whichever access path actually applies to you (paths below assume the CWD-relative `filename=` from step 2 — adjust if you used an absolute path):

```bash
# local target — the realistic case for this skill
ls -la <app>-<label>.jfr    # relative to the target JVM's working directory

# real SSH/file access to the host
scp <host>:<app>-<label>.jfr ./
```

`docker cp`/`kubectl cp` are sometimes suggested here, but don't count on them for the containerized targets this plugin is meant for: `docker cp` needs a local Docker daemon, and `kubectl cp` needs `tar` inside the container, which hardened images typically lack. If that's your situation, you shouldn't have used this skill — go back and use `jolokia-stream-jfr-recording`.

## 4. Analyze the recording

Once the `.jfr` file is on a machine you can read files from, this is identical to the local-`jcmd` workflow — see the `analyze-jfr-recording` skill (from the `jvm-diagnostics-cli` plugin) for command-line triage with `jfr summary`/`jfr print`, or open it in JMC directly.

## Why this needs separate filesystem access

`JFR.start`'s `filename` option and `JFR.dump` both write through `java.io.File` on the JVM process's own host — that's a property of the diagnostic command itself, not something Jolokia can route around. Jolokia/JMX only carries the *command* to the target; it doesn't tunnel the resulting file back. `jolokia-stream-jfr-recording` exists specifically to avoid this constraint by reading the recording's bytes back through a different MBean (`jdk.management.jfr:type=FlightRecorder`'s stream operations) instead of a filesystem path.
