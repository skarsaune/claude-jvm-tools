# claude-jvm-tools

Claude Code skills for diagnosing running JVMs using only serviceability tools
(`jps`, `jcmd`, `jstat`, JFR) — before ever opening source code.

## Install

```
/plugin marketplace add <your-github-user-or-org>/claude-jvm-tools
/plugin install jvm-diagnostics@claude-jvm-tools
```

## What's included

The `jvm-diagnostics` plugin bundles:

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

## License

Public domain — see [LICENSE](LICENSE) (Unlicense). Use it however you like.
