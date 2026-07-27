# jvm-diagnostics-jolokia

Claude Code skills for diagnosing a remote JVM over JMX-via-HTTP using a
[Jolokia](https://jolokia.org/) MCP server, when there's no local
`jcmd`/SSH access to the target process.

Use this plugin when the target JVM only exposes a Jolokia HTTP endpoint —
e.g. a container/pod where you can't get a shell. If you have local process
access or SSH, [jvm-diagnostics-cli](../jvm-diagnostics-cli/README.md) is
simpler and doesn't need Jolokia set up.

## Install

```
/plugin marketplace add skarsaune/claude-jvm-tools
/plugin install jvm-diagnostics-jolokia@claude-jvm-tools
```

Requires an MCP server that bridges to Jolokia and exposes `listMBeans`,
`listMBeanOperations`, `listMBeanAttributes`, `readMBeanAttribute`,
`writeMBeanAttribute`, and `executeMBeanOperation` tools, connected to a
target JVM's Jolokia HTTP agent.

## Skills

- **jolokia-run-diagnostic-command** — the calling convention for
  `com.sun.management:type=DiagnosticCommand` operations over Jolokia
  (the `jcmd`-equivalent MBean). Referenced by the other skills below rather
  than repeated in each.
- **jolokia-inspect-jvm** — uptime, flags, heap/GC summary, thread count,
  class histogram, OS-level CPU for a Jolokia-reachable JVM.
- **jolokia-diagnose-thread-leak** — narrow down a suspected thread leak via
  MBean attribute trends and `threadPrint` clustering.
- **jolokia-diagnose-heap-leak** — narrow down a suspected heap/object leak
  via class-histogram diffs, forced-GC confirmation, and retaining-root
  checks.
- **jolokia-capture-jfr-recording** — start/check/dump a JFR recording via
  `DiagnosticCommand`'s `JFR.*` operations. File-based: the resulting `.jfr`
  file lands on the *target's* filesystem, so this needs some other way to
  retrieve it (local access, `docker cp`/`kubectl cp`, SSH).
- **jolokia-stream-jfr-recording** — capture a JFR recording using only the
  `jdk.management.jfr:type=FlightRecorder` MBean's stream operations, with
  zero filesystem dependency on the target host. Use this when Jolokia is
  your only access to the target at all.

Each skill is observation-first: it favors live JVM tooling over reading
source, and only points at specific classes/methods once the tooling has
narrowed things down.
