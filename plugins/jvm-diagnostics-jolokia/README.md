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

- **jolokia-configure-mcp-server** — connect (or switch/disconnect) a Jolokia
  MCP server for the current project via `claude mcp add --scope local`,
  keyed to a specific target URL. Start here if no Jolokia MCP server is
  connected yet.
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
- **jolokia-stream-jfr-recording** — capture a JFR recording using only the
  `jdk.management.jfr:type=FlightRecorder` MBean's stream operations, with
  zero filesystem dependency on the target host. **Default choice** — if
  Jolokia is your access path at all, this works regardless of what else you
  do or don't have.
- **jolokia-capture-jfr-recording** — start/check/dump a JFR recording via
  `DiagnosticCommand`'s `JFR.*` operations. File-based: the resulting `.jfr`
  file lands on the *target's* filesystem. Only worth it when the target
  really is local to you (the file lands in your own `/tmp`) or you have SSH
  onto the host — `docker cp` assumes a local Docker daemon you often won't
  have, and `kubectl cp` needs `tar` inside the container, which hardened/
  distroless images typically don't ship.
