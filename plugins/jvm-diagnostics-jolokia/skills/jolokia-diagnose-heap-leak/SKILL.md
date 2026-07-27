---
name: jolokia-diagnose-heap-leak
description: Narrow down a suspected heap/application-object leak in a JVM reachable only through a Jolokia MCP server (no local jcmd/SSH access) — confirm it's a real leak (not GC lag), find what other leaking resource it correlates with, and locate the retaining GC root — before opening source code. Use when a class's instance count keeps climbing on a Jolokia-only-reachable process, or after jolokia-diagnose-thread-leak has flagged a suspect and you need to check for a sibling heap leak.
---

# Diagnose a heap / application-object leak via Jolokia

Requires a connected Jolokia MCP server pointed at the target JVM. Pairs naturally with `jolokia-diagnose-thread-leak` — a request-driven thread leak often has a sibling heap leak on the same code path. See `jolokia-run-diagnostic-command` for the DiagnosticCommand calling convention used below.

## 1. If you don't have a suspect class yet, find one from a two-snapshot diff

```
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / gcClassHistogram, args: null
# wait 30-60s (traffic must be flowing for growth to show)
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / gcClassHistogram, args: null
```

Both calls will very likely exceed the tool result size limit and get redirected to local files (see `jolokia-run-diagnostic-command` step 4) — diff those two saved files by class name, sort by biggest positive delta, same as you would with two local `jcmd` redirects.

**First pass: raw diff, no filtering.** Read the ranked deltas as-is; sometimes the top movers are already unambiguous.

**Second pass, only if the raw diff is dominated by noise:** a bare instance-count diff is usually swamped by JDK/GC housekeeping churn (`[B`, `String`, `ConcurrentHashMap$Node`, compiler/reflection classes) that grows and shrinks constantly. Rerun the diff filtered to exclude system packages:

```bash
grep -vE '\(java\.[a-z]+@|\(jdk\.|\(sun\.'
```

Treat this as a fallback, not the default — filtering by default risks hiding a leak that shows up as growth in a library collection type used directly by a JDK class. Whatever class(es) show sustained, one-directional growth across both passes are your suspect(s).

## 2. Rule out "just hasn't been collected yet"

A high or growing instance count is not proof of a leak — it could be reclaimable garbage sitting between collections. Force the question:

```
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / gcClassHistogram, args: null   # note instance count for <SuspectClass>
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / gcRun, args: null                # force a full GC — see the caution in jolokia-run-diagnostic-command step 5
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / gcClassHistogram, args: null   # recheck
```

If the count is unchanged after a forced full GC, the instances are strongly reachable from a live GC root — confirmed leak. If the count drops, it was reclaimable garbage, not a leak (at least not at this rate).

`gcRun` is a real, if cheap, state-changing operation (it forces `System.gc()` JVM-wide) — see the caution in `jolokia-run-diagnostic-command` step 5 before calling it on a live traffic-serving process.

## 3. Check whether it's the only leaking resource

Cross-reference against the thread count, cheaply, without needing another full dump:

```
readMBeanAttribute: java.lang:type=Threading / ThreadCount
executeMBeanOperation: com.sun.management:type=DiagnosticCommand / gcClassHistogram, args: null   # check <SuspectClass> count
```

If a heap-object count and the thread count grow in lockstep across two or more samples (compare ratios, e.g. 3,111 objects vs 3,157 threads, then 3,118 vs 3,186 later), that's strong evidence both are symptoms of the *same* per-request code path — see `jolokia-diagnose-thread-leak`. Look for other classes that moved together in the same histogram diff too (e.g. a `Foo$$Lambda` class tracking 1:1 with the suspect class) — that pairing usually marks the exact call site.

## 4. Narrow down what's keeping the objects reachable, without a heap dump

A leaked object doesn't retain itself — something with a long lifetime holds a reference to it, directly or transitively. Without a heap-dump analyzer you can't see the exact chain, but you can narrow the *category* of root:

- **Check candidate owning classes' instance counts:**
  ```
  executeMBeanOperation: com.sun.management:type=DiagnosticCommand / gcClassHistogram, args: null   # check <CandidateOwningClass> count
  ```
  A count of exactly **1** for a framework-managed singleton (a Spring `@Controller`/`@Service`/`@Component` bean, a cache instance, a static holder) means that single instance is a permanent GC root for the life of the process — anything it references transitively is retained regardless of request volume. This is the most common shape, but not the only one:
- **A growing count of the leaked class with no single obvious owner** can instead point at: a `ThreadLocal` set but never removed (cross-check against thread count/dump from `jolokia-diagnose-thread-leak`); a listener/callback list that's registered into but never deregistered from; a cache entry that *does* have an eviction policy that's broken or never triggered; or a classloader leak.
- **Static fields are a root without needing any instance at all** — reachable independent of whether the declaring class is a singleton in the framework sense.

State whichever shape the evidence actually supports rather than defaulting to "unbounded cache on a singleton" — that's one possible shape among several with the same class-count symptom.

## 5. Optional deeper root-cause tracing (generally unavailable here)

`jhsdb clhsdb --pid <pid>` (used in the local-`jcmd` version of this skill for live GC-root inspection) requires local process attach — it has no Jolokia equivalent, since Jolokia only exposes what the JVM publishes via MBeans. If you need this level of detail on a Jolokia-only-reachable process, the reasoning in step 4 (singleton-plus-lockstep-counts) is normally sufficient to hand off to a source read; a full heap dump (`HotSpotDiagnostic`'s `dumpHeap` operation, if whitelisted) plus offline analysis is the next escalation if it isn't.

## 6. Only now, open the source

With a likely retaining root/mechanism identified (step 4) and, ideally, an endpoint/method identified as the per-request trigger (from a paired `jolokia-diagnose-thread-leak` finding), read just that class for the specific field or registration call responsible — an unbounded collection field, a `ThreadLocal` without a matching remove, a listener list without deregistration, or a broken eviction policy.

## Why Jolokia instead of local jcmd here

Use this skill instead of the local-`jcmd` `diagnose-heap-leak` skill specifically when the target JVM isn't reachable via local process attach or SSH. The diagnostic logic is identical; only the transport differs, and large histogram output needs an extra file-read step per snapshot (see `jolokia-run-diagnostic-command` step 4) since it can't be redirected directly to a file the way a local `jcmd` invocation's stdout can.

Deliberately not included: firing a bulk burst of synthetic requests to force an unambiguous before/after delta. Generating load against a system is a decision with its own blast radius and shouldn't be a default step in a diagnostic skill.