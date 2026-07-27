---
name: jolokia-inspect-jvm
description: Inspect a running JVM's health (uptime, heap/GC, thread count, class histogram) through a Jolokia MCP server, when there's no local jcmd/SSH access to the process — only HTTP-via-Jolokia. Use when the user asks for JVM detail (resource usage, thread count, heap/GC state) about a process reachable only through Jolokia, or wants a jolokia-based equivalent of jcmd VM.uptime/GC.heap_info/Thread.print.
---

# Inspect a JVM via Jolokia

Requires an already-connected Jolokia MCP server pointed at the target JVM's Jolokia agent (HTTP/JMX bridge). See `jolokia-run-diagnostic-command` for the general calling convention this skill relies on.

## 0. Confirm the connection and see what's exposed

```
listMBeans
```

A working connection returns standard platform MBeans (`java.lang:type=Memory`, `java.lang:type=Threading`, `java.lang:type=OperatingSystem`, GC pools, `java.lang:type=ClassLoading`) plus whatever the target's `jolokia-access.xml` additionally whitelists (commonly `com.sun.management:type=DiagnosticCommand`, `com.sun.management:type=HotSpotDiagnostic`, `jdk.management.jfr:type=FlightRecorder`). If `DiagnosticCommand` isn't listed, the `jcmd`-equivalent operations below (`vmUptime`, `gcClassHistogram`, `threadPrint`) aren't available — fall back to the plain MXBean attribute reads, which only need the standard `java.lang:*` MBeans.

## 1. Uptime and JVM identity

Two ways to get this, prefer whichever MBeans are exposed:

```
# via DiagnosticCommand (jcmd VM.uptime equivalent)
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / vmUptime, args: null

# via plain MXBean attributes (always available)
readMBeanAttribute: java.lang:type=Runtime / Uptime      # milliseconds
readMBeanAttribute: java.lang:type=Runtime / Name        # "<pid>@<hostname>"
readMBeanAttribute: java.lang:type=Runtime / StartTime   # epoch millis
```

**Locale gotcha (inherited from the CLI equivalent):** `vmUptime`'s text output can render the decimal separator as a comma on a comma-decimal locale JVM — `92,448 s` means 92.448 seconds, not "92 thousand 448." The plain `Runtime.Uptime` attribute avoids this ambiguity entirely (it's a raw long in milliseconds), so prefer it when precision matters.

## 2. JVM flags

```
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / vmFlags, args: null
```

No plain-MXBean equivalent for the full flag set; `java.lang:type=Runtime`'s `InputArguments` attribute gives you only what was explicitly passed on the command line (not ergonomics-derived defaults), which is a reasonable fallback if `DiagnosticCommand` isn't exposed.

## 3. Heap / GC summary

```
# via DiagnosticCommand — human-readable per-generation breakdown (jcmd GC.heap_info equivalent)
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / gcHeapInfo, args: null

# via plain MXBean attributes — structured, good for programmatic before/after comparison
readMBeanAttribute: java.lang:type=Memory / HeapMemoryUsage
  -> {"init":..., "committed":..., "max":..., "used":...}   # bytes

readMBeanAttribute: java.lang:name=G1 Old Generation,type=GarbageCollector / CollectionCount
readMBeanAttribute: java.lang:name=G1 Young Generation,type=GarbageCollector / CollectionCount
```

Rising `used` in `HeapMemoryUsage` across two samples, together with a climbing Old-Gen `CollectionCount`, is the same heap-leak signature the CLI skill looks for via `jstat -gcutil`. `gcHeapInfo` gives the same picture as readable text (region breakdown, Metaspace) in one call — use it when you want to eyeball the shape, use the MXBean attributes when you want to diff two numeric samples precisely.

## 4. Thread count and thread dump

```
# just the live thread count — cheap, safe to poll repeatedly
readMBeanAttribute: java.lang:type=Threading / ThreadCount

# full stack-trace dump (jcmd Thread.print equivalent) — can be large, see jolokia-run-diagnostic-command step 4
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / threadPrint, args: null
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / threadPrint, args: ["-l"]   # include java.util.concurrent locks
```

A live thread count much higher than expected for the app's baseline (a small Spring Boot MVC app at rest is typically 30-50 threads) is worth a closer look via the full dump. `Threading` MBean also exposes `PeakThreadCount` and `TotalStartedThreadCount` (monotonically increasing since JVM start) — a `TotalStartedThreadCount` growing much faster than requests served is itself a leak signal without needing a full dump.

## 5. Class histogram (heap object breakdown)

```
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / gcClassHistogram, args: null
```

Almost always exceeds the tool result size limit and gets redirected to a local file (see `jolokia-run-diagnostic-command` step 4) — read that file with `grep`/`head` rather than expecting it inline. Top of the list by instance count/bytes is useful to spot which type is accumulating if heap usage is growing; for a proper leak diagnosis (two-snapshot diff, forced-GC confirmation) see `jolokia-diagnose-heap-leak`.

## 6. OS-level resource usage

```
readMBeanAttribute: java.lang:type=OperatingSystem / ProcessCpuLoad     # 0.0-1.0, -1 if unavailable
readMBeanAttribute: java.lang:type=OperatingSystem / SystemCpuLoad
readMBeanAttribute: com.sun.management:type=OperatingSystem / <same names, if the com.sun.management variant is exposed instead>
```

There's no Jolokia equivalent of local `ps` (RSS, process state) since Jolokia only sees what the JVM itself exposes via MBeans, not OS-level process accounting — `ProcessCpuLoad` is the closest substitute for CPU usage.