---
name: diagnose-heap-leak
description: Narrow down a suspected heap/application-object leak in a running Java process using jcmd/jstat/ps — confirm it's a real leak (not GC lag), find what other leaking resource it correlates with, and locate the retaining GC root. Use when a class's instance count on the heap keeps climbing, or after diagnose-thread-leak has flagged a suspect endpoint/class and you need to check whether it's also leaking heap objects.
---

# Diagnose a heap / application-object leak

Requires a PID. Pairs naturally with `diagnose-thread-leak` — a request-driven thread leak often has a sibling heap leak on the same code path (both symptoms of one unbounded-collection bug), so once you've found one, check for the other.

## 1. If you don't have a suspect class yet, find one from a two-snapshot diff

Take a full `GC.class_histogram` snapshot, wait, take a second, and diff instance counts by class name to see what's actually growing — don't eyeball a single snapshot's top-N, since that's dominated by whatever's largest at rest, not whatever's growing.

```bash
jcmd <pid> GC.class_histogram > /tmp/hist1.txt
# wait 30-60s (traffic must be flowing for growth to show)
jcmd <pid> GC.class_histogram > /tmp/hist2.txt
# diff by class name, sort by biggest positive delta
```

**First pass: raw diff, no filtering.** Read the ranked deltas as-is. Sometimes the top movers are already unambiguous (a specific app class or app-owned lambda growing steadily while everything else is flat).

**Second pass, only if the raw diff is dominated by noise:** a bare instance-count diff is usually swamped by JDK/GC housekeeping churn (`[B`, `String`, `ConcurrentHashMap$Node`, compiler/reflection classes, etc.) that grows and shrinks constantly and tells you nothing. If that's what you're looking at, rerun the same diff filtered to exclude `java.*`/`jdk.*`/`sun.*` packages:

```bash
grep -vE '\(java\.[a-z]+@|\(jdk\.|\(sun\.'
```

This surfaces the app- and library-level classes (your own code, Spring/H2/Jackson/etc.) whose growth pattern is otherwise buried under system-class noise. Treat this as a fallback, not the default first look: filtering by default risks hiding a leak that manifests as growth in a *library* collection type used directly by a JDK class (e.g. `WeakHashMap$Entry` climbing because something app-level keeps inserting into a `WeakHashMap` and nothing evicts it) — you'd want to see that in the raw pass and then decide whether it's noise or a lead, not have it filtered away unseen. Whatever class(es) show sustained, one-directional growth (not oscillating) across both passes are your suspect(s) for step 2 below.

## 2. Rule out "just hasn't been collected yet"

A high or growing instance count in `GC.class_histogram` is not proof of a leak — it could be reclaimable garbage sitting between collections. Force the question:

```bash
jcmd <pid> GC.class_histogram | grep -i <SuspectClass>   # note instance count
jcmd <pid> GC.run                                         # force a full GC
jcmd <pid> GC.class_histogram | grep -i <SuspectClass>   # recheck
```

If the count is unchanged after a forced full GC, the instances are strongly reachable from a live GC root — confirmed leak, not GC lag. If the count drops, it was reclaimable garbage and not a leak (at least not at this rate).

## 3. Check whether it's the only leaking resource

Don't stop at the first suspect class. Cross-reference against other resource counters taken the same way you would for `diagnose-thread-leak`:

```bash
jcmd <pid> Thread.print | grep -c '^"'          # live thread count
jcmd <pid> GC.class_histogram | grep -i <SuspectClass>
```

If a heap-object count and the thread count grow in lockstep (compare ratios across two or more samples, e.g. 3,111 objects vs 3,157 threads, then 3,118 vs 3,186 later), that's strong evidence both are symptoms of the *same* per-request code path, not two unrelated bugs. Look for other objects that moved together in the same histogram diff too (e.g. a `Foo$$Lambda` class tracking 1:1 with the suspect class) — that pairing usually marks the exact call site.

## 4. Narrow down what's keeping the objects reachable, without a heap dump

A leaked object doesn't retain itself — something with a long lifetime is holding a reference to it (directly, or transitively through a chain of other objects). Without a full heap-dump analyzer you can't see the exact reference chain, but you can narrow the *category* of root:

- **Check candidate owning classes' instance counts:**
  ```bash
  jcmd <pid> GC.class_histogram | grep -i <CandidateOwningClass>
  ```
  A count of exactly **1** for a class that's a framework-managed singleton (a Spring `@Controller`/`@Service`/`@Component` bean, a cache instance, a static holder) means that single instance is a de-facto permanent GC root for the life of the process — anything it references transitively is retained regardless of request volume. This is the most common shape, but not the only one:
- **A growing count of the leaked class itself with no single obvious owner** can instead point at: a `ThreadLocal` that's set but never removed (each thread pins its own copy — cross-check against the thread count/dump from `diagnose-thread-leak`); a listener/callback list that things register into but never deregister from; an entry accumulating in a cache that *does* have an eviction policy, but one that's broken or never triggered (e.g. wrong equals/hashCode preventing eviction lookups from matching); or a classloader leak (check `jcmd <pid> VM.classloader_stats` if the suspect class's own class, not just instances, keeps reappearing after redeploys).
- **Static fields are a root without needing any instance at all** — a `static` collection field on any class is itself always reachable, independent of whether the declaring class is a "singleton" in the framework sense.

State whichever shape the evidence actually supports (singleton-held collection, ThreadLocal, broken eviction, static field, listener accumulation) rather than defaulting to "unbounded cache on a singleton" — that's one possible shape among several with the same class-count symptom.

## 5. Optional deeper root-cause tracing (often blocked)

`jhsdb clhsdb --pid <pid>` can attach to a live process and inspect live objects/roots directly, without producing a heap dump file. On macOS this is usually blocked by SIP (`task_for_pid failed`) unless SIP is disabled or you run as root — don't ask the user to weaken system security just to get this; treat the failure as expected and fall back to the singleton-plus-lockstep-counts reasoning in step 4, which is normally sufficient to hand off to a source read.

## 6. Read the source

With a likely retaining root/mechanism identified (from step 4) and, ideally, an endpoint/method identified as the per-request trigger (from a paired `diagnose-thread-leak` finding, or from natural traffic observed in step 3), read just that class for the specific field or registration call responsible — an unbounded collection field, a `ThreadLocal` without a matching `remove()`, a listener list without deregistration, or an eviction policy that never fires.

## Why this order matters

Finding a suspect via a filtered/unfiltered histogram diff (step 1) beats guessing a class name up front. Confirming non-collectability (step 2) before hunting for a cause avoids chasing heap noise. Cross-referencing against other resources (step 3) catches cases where a single bug manifests as multiple "separate-looking" leaks — missing this leads to fixing one symptom and declaring victory while the other keeps growing. Step 4 narrows the root cause to one of a few specific shapes before source is even opened, so the eventual code read is a confirmation step, not a fishing expedition — but which shape it is should come from the evidence (instance counts, thread-local/classloader checks), not be assumed up front.

Deliberately not included: firing a bulk burst of synthetic requests at the process to force an unambiguous before/after delta. That technique proves causality cleanly, but generating load against a system is a decision with its own blast radius (it's not always a private demo instance) and shouldn't be a default step in a diagnostic skill — treat it as a separate, deliberate call the user makes on its own, not something baked into routine leak-hunting.
