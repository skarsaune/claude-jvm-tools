---
name: find-java-processes
description: List running Java processes on the local machine (JVM PID, main class, and full JVM args). Use when the user asks what Java/JVM processes are running, wants to find the PID of the PetClinic app, JMC, or another JVM, or needs JVM args (e.g. debug/JMX/Jolokia ports) for an already-running process.
---

# Find Java processes

To list every running JVM with its PID, main class, and full command-line arguments:

```bash
jps -lv
```

- `-l` prints the fully-qualified main class (or jar path) instead of just the short name.
- `-v` prints the JVM arguments passed to each process (heap flags, javaagents, debug/JMX ports, etc.) — this is what lets you spot things like the JDWP debug socket, the Jolokia agent, or the JMX remote port on the PetClinic process.

Fallback if `jps` isn't on `PATH` (e.g. non-JDK environments):

```bash
ps aux | grep -i java | grep -v grep
```

`ps` output is less structured (no clean main-class column) but works without a JDK install.

`jps` only sees JVMs owned by the same user unless run with elevated privileges.
