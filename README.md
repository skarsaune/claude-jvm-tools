# claude-jvm-tools

Claude Code skills for diagnosing running JVMs.

## Install

```
/plugin marketplace add skarsaune/claude-jvm-tools
```

Then install whichever plugin matches how you can reach the target JVM:

| Plugin | Use when | |
|---|---|---|
| [jvm-diagnostics-cli](plugins/jvm-diagnostics-cli/README.md) | You have local process access or SSH (`jps`/`jcmd`/`jstat`/JFR) | `/plugin install jvm-diagnostics-cli@claude-jvm-tools` |
| [jvm-diagnostics-jolokia](plugins/jvm-diagnostics-jolokia/README.md) | Only a Jolokia HTTP/JMX endpoint is reachable (e.g. a container with no shell) | `/plugin install jvm-diagnostics-jolokia@claude-jvm-tools` |

See each plugin's README for its full skill list.

## License

Public domain — see [LICENSE](LICENSE) (Unlicense). Use it however you like.
