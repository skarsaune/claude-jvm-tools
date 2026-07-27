---
name: jolokia-run-diagnostic-command
description: Call any jcmd-equivalent operation on a remote JVM's com.sun.management:type=DiagnosticCommand MBean through a Jolokia MCP server (executeMBeanOperation) — covers the exact args shape, how to pass jcmd-style options, and how to handle output too large for a single tool result. Use this first whenever another jolokia-* skill tells you to run a DiagnosticCommand operation, or whenever the user asks for a jcmd-like command against a JVM reachable only through Jolokia (no local jcmd/SSH access).
---

# Run a DiagnosticCommand operation via Jolokia

The `com.sun.management:type=DiagnosticCommand` MBean exposes (almost) every `jcmd <pid> <Command>` as an MBean operation, reachable via the Jolokia MCP's `executeMBeanOperation` tool without any local `jcmd`/SSH access to the JVM. Operation names are the `jcmd` command name with the dot removed and camelCased: `Thread.print` → `threadPrint`, `GC.class_histogram` → `gcClassHistogram`, `VM.uptime` → `vmUptime`, `JFR.start` → `jfrStart`, etc.

Confirm the MBean is actually reachable before relying on this (see `jolokia-inspect-jvm` step 0, or just try `listMBeans` and check for `com.sun.management:type=DiagnosticCommand` in the result). Access is controlled by the target's `jolokia-access.xml` — not every deployment whitelists this MBean.

## 1. Every operation takes exactly one argument: a string array

Every `DiagnosticCommand` operation's signature is `(String[] arguments)` — there is no zero-arg overload, even for commands that take no options. Getting the `args` shape wrong is the most common failure mode here, confirmed against a live agent:

| What you want | Correct `args` value | Why |
|---|---|---|
| No options (e.g. plain `Thread.print`) | `null` | The operation still requires 1 parameter; `null` satisfies it. |
| No options, alternate | *(omit the args param if your tool allows it)* | Some MCP tool wrappers treat a missing `args` the same as `null`. |
| One or more options | a **flat** JSON array of strings, e.g. `["-l"]` or `["name=skilltest", "settings=profile"]` | Each element is one whitespace-separated token from the equivalent `jcmd` command line. |

**Do NOT pass `[]`** (empty array) for a no-arg call — this throws `IllegalArgumentException: Unknown argument '[]' in diagnostic command`, because Jolokia hands the JMX layer an empty-string element rather than truly zero elements. Use `null`, not `[]`.

**Do NOT nest the array** — `[["-l"]]` throws `Unknown argument '[-l]'`. `args` is the array itself, e.g. `["-l"]`, not an array containing an array.

## 2. Options use `key` or `key=value`, never bare CLI flags

`jcmd` options that look like Unix flags (`-l`, `-all`) are actually `key`/`key=value` tokens in disguise — pass them exactly as `jcmd` would show them, as individual array elements:

```
threadPrint,        args: ["-l"]                          // Thread.print -l
gcClassHistogram,    args: ["-all"]                        // GC.class_histogram -all
jfrStart,            args: ["name=demo", "settings=profile", "duration=60s", "filename=/tmp/demo.jfr"]
jfrCheck,            args: ["name=demo", "verbose=true"]
```

Each `key=value` pair is **one** array element (comma-separated on the `jcmd` command line becomes one array entry here, not split further) — e.g. `jcmd <pid> JFR.start maxage=1h,maxsize=1000M` on the command line is a single option pair that becomes `args: ["maxage=1h,maxsize=1000M"]` as one element, or split into two elements `["maxage=1h", "maxsize=1000M"]` — both work, confirmed live; use whichever reads more clearly.

## 3. Discover exact option syntax with `help`

Don't guess flag names or defaults — `help` is itself a `DiagnosticCommand` operation and returns the same text as `jcmd <pid> help <Command>`, including every option, its type/default, and example invocations:

```
mbean: com.sun.management:type=DiagnosticCommand
operation: help
args: ["Thread.print"]      // or ["JFR.start"], ["GC.class_histogram"], etc. — use the dotted jcmd name here, not the camelCase operation name
```

Run this before guessing at a command's option syntax, especially for `JFR.start`/`JFR.check`/`GC.class_histogram`, which have the richest option sets.

## 4. Large output gets redirected to a local file — read it, don't assume it's in the tool result

Commands that produce a lot of text — a full `threadPrint`, `gcClassHistogram -all`, or even a plain `gcClassHistogram` on an app with many loaded classes — routinely exceed the MCP tool result's token budget. When that happens the tool call doesn't fail; it succeeds and saves the full output to a local file, e.g.:

```
Error: result (855,876 characters across 8,522 lines) exceeds maximum allowed tokens.
Output has been saved to /Users/.../tool-results/mcp-jolokia-executeMBeanOperation-<ts>.txt
```

Read that file with `Read`/`grep`/`awk` as you would any large local file (chunk with `offset`/`limit`, or shell out to `grep`/`wc`) — the diagnostic data is all there, just not inline in the conversation. This is a key difference from the local-`jcmd` skills, which redirect to `/tmp` themselves by design; here the redirection is an artifact of the tool layer, not something you asked for, so don't mistake the error text for a real failure.

## 5. This is read/observe-first tooling — most operations are safe, a few are not

The vast majority of `DiagnosticCommand` operations (`threadPrint`, `gcClassHistogram`, `vmUptime`, `vmFlags`, `jfrStart`/`jfrDump`/`jfrCheck`, etc.) are non-destructive snapshots or additive recordings. A few are not, and deserve the same caution as any other state-changing action:

- `gcRun` forces a full GC — cheap to call, but can cause a visible latency blip on a live traffic-serving process; fine for a deliberate "confirm this isn't just GC lag" check (see `jolokia-diagnose-heap-leak`), not something to call speculatively.
- `vmSetFlag` mutates a live JVM flag — treat like any other production config change.
- `jvmtiAgentLoad` loads a native agent into the target process — do not call this without explicit user direction.

Everything else in this skill set only calls the safe, observational subset.