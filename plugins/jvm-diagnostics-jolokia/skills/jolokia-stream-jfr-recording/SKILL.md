---
name: jolokia-stream-jfr-recording
description: Capture a JDK Flight Recorder (JFR) recording from a remote JVM using ONLY the jdk.management.jfr:type=FlightRecorder MBean over Jolokia, with zero dependency on filesystem access to the target host — open a byte stream over JMX, read it in chunks, and reassemble a .jfr file locally. This is the default choice for a Jolokia-reachable target: containers/pods rarely give you a working way to pull a file off afterwards (docker cp needs a local Docker daemon; kubectl cp needs tar inside the container, absent from hardened images). Only skip this in favor of jolokia-capture-jfr-recording (file-based, via DiagnosticCommand JFR.start) when the target is genuinely local to you or you have real SSH access to the host.
---

# Stream a JFR recording via Jolokia (no filesystem access needed)

Requires a connected Jolokia MCP server pointed at the target JVM, with the `jdk.management.jfr:type=FlightRecorder` MBean whitelisted (check `listMBeans` for it). This MBean is the JMX-native `jdk.management.jfr.FlightRecorderMXBean` — it manages recordings and lets you read their bytes back as an in-band stream, entirely separate from `com.sun.management:type=DiagnosticCommand`'s `JFR.*` operations (which write to a file on the target's own disk — see `jolokia-capture-jfr-recording`).

Every operation call below takes a **flat**, non-nested `args` array — e.g. `args: [2, "default"]`, not `args: [[2, "default"]]`. This mirrors the `DiagnosticCommand` calling convention in `jolokia-run-diagnostic-command`, but note the argument *types* here are the MBean's declared Java types (`long`, `String`, `boolean`, `TabularData`), not all strings.

## 1. Create and configure a recording

```
executeMBeanOperation: jdk.management.jfr:type=FlightRecorder / newRecording, args: []
  -> returns a recording id (long), e.g. 2

executeMBeanOperation: jdk.management.jfr:type=FlightRecorder / setPredefinedConfiguration, args: [2, "default"]
  # or "profile" for higher-detail/higher-overhead capture, same tradeoff as JFR.start settings=

executeMBeanOperation: jdk.management.jfr:type=FlightRecorder / startRecording, args: [2]
```

Unlike `DiagnosticCommand`'s `jfrStart`, `newRecording` genuinely takes zero arguments — `args: []` is correct here (not `null`; this MBean's operations use ordinary JMX arity, not the DiagnosticCommand string-array convention).

## 2. Let it run, then get a stoppable copy of the data

**Confirmed by testing: you cannot open a stream on a recording that is still running** — `openStream` on a live recording throws `IOException: Recording must be stopped before it can be read.` You have two options:

- **Stop the recording outright** if you're done capturing:
  ```
  executeMBeanOperation: jdk.management.jfr:type=FlightRecorder / stopRecording, args: [2]
  ```
- **Take a snapshot** if you want to read data so far *without* interrupting an ongoing recording (e.g. checking progress mid-capture):
  ```
  executeMBeanOperation: jdk.management.jfr:type=FlightRecorder / takeSnapshot, args: []
    -> returns a NEW recording id representing a stopped, point-in-time copy, e.g. 3
  ```
  Confirmed working: `takeSnapshot` on recording 2 (still running) returned recording 3, and `openStream` succeeded against 3 while 2 kept recording. Remember to `closeRecording` the snapshot separately when done (step 5) — it's a distinct recording object, not a view into the original.

## 3. Open a stream and read it to EOF

```
executeMBeanOperation: jdk.management.jfr:type=FlightRecorder / openStream, args: [<recordingId>, {}]
  -> returns a stream id (long), e.g. 1
```

The second argument is a `TabularData` of stream options (block size, start/end time filters) — an empty object `{}` is accepted and gives sane defaults; confirmed working live. Then loop:

```
executeMBeanOperation: jdk.management.jfr:type=FlightRecorder / readStream, args: [<streamId>]
```

Each call returns one chunk (observed ~170KB per call against a live agent, but treat the size as implementation-defined, not a contract) as a **JSON array of Java `byte` values — i.e. SIGNED integers in the range -128..127**, e.g. `[70, 76, 82, 0, 0, 2, 0, 1, ...]`. This is the single most important gotcha in this skill:

**You must convert each signed byte to its unsigned 8-bit value before writing it out, or the resulting file is corrupt.** In pseudocode: `unsignedByte = signedByte < 0 ? signedByte + 256 : signedByte` (equivalently `signedByte & 0xFF`). A valid `.jfr` file starts with the literal bytes `FLR\0` — `70, 76, 82, 0` in decimal — which is exactly what a correctly-decoded first chunk looks like; if your reassembled file doesn't start with those four bytes, the byte conversion is wrong.

Keep calling `readStream` and appending decoded bytes to a local buffer/file until it returns `null` — that's end-of-stream (per the `FlightRecorderMXBean.readStream` contract; not independently re-confirmed to null in testing here, but this is the documented JDK API behavior and the only sane termination condition — don't loop on a fixed count).

**For anything beyond a trivial/short recording, don't do this read loop through the MCP tool calls themselves.** Each `readStream` result is a JSON array of tens of thousands of byte values that lands in the calling agent's context — confirmed in practice: a 2-minute `profile`-settings recording alone took ~2.3MB / 46 chunks, which is already impractical to shuttle through tool-call results, and real recordings are routinely far larger. Instead, write a short local script (Python, Node, whatever's available) that talks to the underlying Jolokia HTTP endpoint directly — same `exec`-type POST body as the MCP tool sends, but looped and decoded locally, writing straight to disk. The agent should still drive the *orchestration* (open the recording, start/stop it, confirm the result) through the MCP tools; only the bulk `readStream` loop needs to bypass them.

**The MCP tool surface itself gives you no way to discover that HTTP endpoint's address** — `listMBeans`/`readMBeanAttribute`/`executeMBeanOperation` are all scoped to MBean names and JMX operations, none of them expose a self-link, base URL, or "what am I connected to" fact about the Jolokia agent itself. Do not assume a default like `localhost:8778` — that only happened to be correct for a local demo target; a real target is just as likely to be a remote host or a Kubernetes pod. If the server was connected via `jolokia-configure-mcp-server` (`claude mcp add --scope local`), the URL is visible in `claude mcp list`'s launch args since it's baked into the add-time command — check there first. Otherwise, ask the user for the Jolokia base URL before writing the fallback script.

## 4. Reassemble the file locally

Whatever language/tool is doing the byte-array decoding, write the concatenated unsigned bytes to a local `.jfr` file in **binary** mode (not text/UTF-8 — this is raw binary data that will contain arbitrary byte values, including ones that aren't valid UTF-8 sequences). Once assembled, it's a normal `.jfr` file — analyze it with the `analyze-jfr-recording` skill (from `jvm-diagnostics-cli`) or open it in JMC, exactly as if it had come from a local `jcmd JFR.dump`.

## 5. Clean up

Always close what you opened, in this order, or resources leak on the target JVM for the life of the process:

```
executeMBeanOperation: jdk.management.jfr:type=FlightRecorder / closeStream, args: [<streamId>]
executeMBeanOperation: jdk.management.jfr:type=FlightRecorder / closeRecording, args: [<recordingId>]   # the snapshot id, if you took one
executeMBeanOperation: jdk.management.jfr:type=FlightRecorder / stopRecording, args: [<originalId>]     # if the original is still running and you're done with it
executeMBeanOperation: jdk.management.jfr:type=FlightRecorder / closeRecording, args: [<originalId>]
```

Confirmed all four calls succeed cleanly in this order after a snapshot-based read (streamId, snapshot recordingId, original recordingId stop, original recordingId close).

## Why this over the file-based approach

`jolokia-capture-jfr-recording` (via `DiagnosticCommand`'s `JFR.start`/`JFR.dump`) is simpler when you have it, but it writes to a file on the *target's* filesystem — and for the containerized/hardened targets this plugin is mainly written for, that file is usually unreachable in practice. `docker cp` needs a local Docker daemon socket, which a remote Jolokia HTTP endpoint gives you no path to. `kubectl cp` shells out to `tar` inside the target container, and hardened/distroless images typically don't ship one — it fails even with valid cluster access. Real SSH onto the host is the only one of the "alternatives" that reliably works, and it's often not available either. This skill's `openStream`/`readStream`/`closeStream` sequence sidesteps all of that by tunneling the recording's bytes back through the same JMX/HTTP channel Jolokia already uses for everything else, so a full `.jfr` file can be reconstructed with nothing more than the Jolokia access you already have — at the cost of the extra streaming/reassembly work and the signed-byte decoding step above. Treat it as the default for any target you can't literally `ls` a file on, not as a fallback of last resort.
