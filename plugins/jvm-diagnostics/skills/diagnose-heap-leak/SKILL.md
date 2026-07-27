---
name: diagnose-heap-leak
description: Narrow down a suspected heap/application-object leak in a running Java process using only jcmd/jstat/ps — confirm it's a real leak (not GC lag), find what other leaking resource it correlates with, and locate the retaining GC root — before opening source code. Use when a class's instance count on the heap keeps climbing, or after diagnose-thread-leak has flagged a suspect endpoint/class and you need to check whether it's also leaking heap objects.
---

# Diagnose a heap / application-object leak

Requires a PID. Pairs naturally with `diagnose-thread-leak` — a request-driven thread leak often has a sibling heap leak on the same code path (both symptoms of one unbounded-collection bug), so once you've found one, check for the other. As with that skill: stay in the observation domain, only open source once the tooling has pinpointed a specific retaining field/class.

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

## 4. Find the retaining root, without a heap dump

You often don't need a full heap-dump analyzer to find *what kind* of root is holding the leaked objects. Check whether the owning class is a singleton:

```bash
jcmd <pid> GC.class_histogram | grep -i <OwningClass>
```

If the count is exactly **1**, and that class is a framework-managed singleton (a Spring `@Controller`/`@Service`/`@Component` bean, a JCache/Caffeine cache instance, a static holder, etc.), that single instance is itself a de-facto permanent GC root for the life of the process. Anything it holds via an instance field (commonly an unbounded `List`/`Map`/`Set` used as an ad-hoc cache) is retained forever, regardless of how many requests come and go. This is enough to state the *shape* of the bug (an unbounded collection field on a long-lived singleton) without needing to see the field itself yet.

## 5. Optional deeper root-cause tracing (often blocked)

`jhsdb clhsdb --pid <pid>` can attach to a live process and inspect live objects/roots directly, without producing a heap dump file. On macOS this is usually blocked by SIP (`task_for_pid failed`) unless SIP is disabled or you run as root — don't ask the user to weaken system security just to get this; treat the failure as expected and fall back to the singleton-plus-lockstep-counts reasoning in step 4, which is normally sufficient to hand off to a source read.

## 6. Only now, open the source

With a specific singleton class identified as the likely retaining root (from step 4) and, ideally, an endpoint/method identified as the per-request trigger (from a paired `diagnose-thread-leak` finding, or from natural traffic observed in step 3), read just that class for an instance-level collection field that's appended to but never trimmed/evicted.

## Why this order matters

Finding a suspect via a filtered/unfiltered histogram diff (step 1) beats guessing a class name up front. Confirming non-collectability (step 2) before hunting for a cause avoids chasing heap noise. Cross-referencing against other resources (step 3) catches cases where a single bug manifests as multiple "separate-looking" leaks — missing this leads to fixing one symptom and declaring victory while the other keeps growing. The singleton check (step 4) narrows the root cause to a specific shape before source is even opened, so the eventual code read is a confirmation step, not a fishing expedition.

Deliberately not included: firing a bulk burst of synthetic requests at the process to force an unambiguous before/after delta. That technique proves causality cleanly, but generating load against a system is a decision with its own blast radius (it's not always a private demo instance) and shouldn't be a default step in a diagnostic skill — treat it as a separate, deliberate call the user makes on its own, not something baked into routine leak-hunting.
