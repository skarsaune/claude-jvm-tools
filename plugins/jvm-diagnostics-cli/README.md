# jvm-diagnostics-cli

Claude Code skills for diagnosing running JVMs using only local serviceability
tools (`jps`, `jcmd`, `jstat`, JFR) — before ever opening source code.

Use this plugin when you have local process access (same host, or SSH) to the
target JVM. If the only access you have is a Jolokia HTTP endpoint (e.g. a
container/pod with no shell access), use
[jvm-diagnostics-jolokia](../jvm-diagnostics-jolokia/README.md) instead.

## Install

```
/plugin marketplace add skarsaune/claude-jvm-tools
/plugin install jvm-diagnostics-cli@claude-jvm-tools
```

## Skills

- **find-java-processes** — list running JVMs (PID, main class, JVM args).
- **inspect-java-process** — uptime, flags, CPU/memory, thread count, heap/GC
  summary, thread dumps for a given PID.
- **capture-jfr-recording** — start/stop/dump a JDK Flight Recorder recording
  via `jcmd`, no agent or restart required.
- **analyze-jfr-recording** — command-line triage of a `.jfr` file: CPU hot
  paths, allocation pressure, lock/IO contention, and separating real signal
  from environment noise.
- **diagnose-thread-leak** — narrow down a suspected thread leak via
  `jcmd`/`jstat` trends and thread-dump clustering.
- **diagnose-heap-leak** — narrow down a suspected heap/object leak via
  class-histogram diffs, forced-GC confirmation, and retaining-root checks.

Each skill is observation-first: it favors live JVM tooling over reading
source, and only points at specific classes/methods once the tooling has
narrowed things down.
