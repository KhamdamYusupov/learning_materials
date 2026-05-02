# JVM Internals, Garbage Collection, and Production Tuning

> A senior-level guide for Java backend engineers (Java 17/21, Spring Boot) who want to understand the JVM deeply, diagnose real production issues, and tune confidently.

---

## Table of Contents

1. [JVM Architecture](#1-jvm-architecture)
2. [Memory Model Deep Dive](#2-memory-model-deep-dive)
3. [Garbage Collection Fundamentals](#3-garbage-collection-fundamentals)
4. [Garbage Collectors Deep Dive](#4-garbage-collectors-deep-dive)
5. [GC Algorithms and Concepts](#5-gc-algorithms-and-concepts)
6. [JVM Tuning Basics](#6-jvm-tuning-basics)
7. [Advanced JVM Tuning](#7-advanced-jvm-tuning)
8. [GC Logging and Analysis](#8-gc-logging-and-analysis)
9. [Debugging Memory Issues](#9-debugging-memory-issues)
10. [JVM Monitoring Tools](#10-jvm-monitoring-tools)
11. [Performance Optimization](#11-performance-optimization)
12. [Real-World Debugging Scenarios](#12-real-world-debugging-scenarios)
13. [JVM in Containers](#13-jvm-in-containers)
14. [Common Pitfalls](#14-common-pitfalls)
15. [When NOT to Tune the JVM](#15-when-not-to-tune-the-jvm)
16. [Production Best Practices](#16-production-best-practices)
17. [Final Checklist](#17-final-checklist)

---

## 1. JVM Architecture

The JVM is a virtual machine that executes Java bytecode. It is **not** an interpreter alone — it's a tiered execution engine with sophisticated memory management, concurrency primitives, and dynamic compilation.

### 1.1 The end-to-end path: `.java` → execution

```
Hello.java
   │  (javac)
   ▼
Hello.class            ← bytecode (platform-independent)
   │  (java launcher)
   ▼
ClassLoader subsystem  ← finds, loads, links, initializes classes
   │
   ▼
Runtime Data Areas     ← heap, stacks, metaspace, code cache, etc.
   │
   ▼
Execution Engine       ← interpreter + JIT (C1, C2, Graal)
   │
   ▼
Native code on CPU
```

A simplified narrative:

1. **Compile time.** `javac` produces `.class` files containing JVM bytecode and a constant pool.
2. **Launch.** `java -cp ... Main` starts a JVM process. The launcher initializes the runtime, finds `Main`, and asks the class loader to load it.
3. **Loading.** Class loaders read bytes (from the file system, JARs, modules, the network, custom sources) and turn them into in-memory `Class` objects.
4. **Linking.** Verification (bytecode safety), preparation (default values for static fields), resolution (symbolic references → direct references).
5. **Initialization.** Static initializers and static fields run for the class.
6. **Execution.** The JVM starts interpreting bytecode for `main`. Hot methods get compiled to native code by the JIT.
7. **Running.** Threads execute. Heap allocation triggers occasional GC. Native code is patched and re-optimized as profiling data accumulates.
8. **Shutdown.** Shutdown hooks run; JVM tears down threads and the runtime.

### 1.2 ClassLoader subsystem

A class loader's job is **find bytes → produce a `Class` object**. The JVM has multiple loaders forming a parent-delegation chain.

#### The standard chain (Java 9+)

```
Bootstrap ClassLoader      ← loads java.base and other platform modules (native; not a Java object)
        ▲
Platform ClassLoader       ← loads platform modules (java.sql, java.xml, etc.)
        ▲
System (Application)       ← loads classes on the application classpath/module-path
        ▲
Custom ClassLoaders        ← framework loaders (Spring Boot fat-jar loader, OSGi, web containers)
```

#### Parent delegation

When a loader is asked for a class, it first asks its parent. Only if the parent says "not found" does it try itself. **Why:** prevents core classes (`java.lang.String`) from being shadowed by malicious or buggy app classes.

#### Three steps inside class loading

| Step | What happens | Common failures |
|---|---|---|
| **Loading** | Read bytes, parse format, create `Class` object. | `ClassNotFoundException`, `NoClassDefFoundError` |
| **Linking — Verification** | Bytecode safety checks. | `VerifyError` |
| **Linking — Preparation** | Allocate static fields with default values (0/null). | — |
| **Linking — Resolution** | Replace symbolic references with direct ones (often lazy). | `NoSuchMethodError`, `IncompatibleClassChangeError` |
| **Initialization** | Run `<clinit>` (static initializers). | `ExceptionInInitializerError` |

#### Mental model

A `Class<?>` is identified by **`(name, defining ClassLoader)`** — the same FQCN loaded by two different loaders is two **different** classes, not assignable to each other. This is why "cannot cast `com.foo.X` to `com.foo.X`" happens in app servers and hot-reload frameworks.

### 1.3 Runtime Data Areas

The JVM divides memory into per-thread and shared regions.

```
                    ┌─────────────────────────────────────────┐
                    │              JVM Process                │
                    ├─────────────────────────────────────────┤
                    │ Shared (across threads):                │
                    │   ┌───────────────────────────────────┐ │
                    │   │ Heap (objects, arrays)            │ │
                    │   │  ┌────────────┬────────────────┐  │ │
                    │   │  │   Young    │      Old       │  │ │
                    │   │  └────────────┴────────────────┘  │ │
                    │   ├───────────────────────────────────┤ │
                    │   │ Metaspace (class metadata)        │ │
                    │   ├───────────────────────────────────┤ │
                    │   │ Code Cache (JIT-compiled code)    │ │
                    │   ├───────────────────────────────────┤ │
                    │   │ String table, symbol table,       │ │
                    │   │ direct buffers, GC structures     │ │
                    │   └───────────────────────────────────┘ │
                    ├─────────────────────────────────────────┤
                    │ Per-thread:                             │
                    │   JVM stack (frames, locals, operands)  │
                    │   PC register                           │
                    │   Native method stack                   │
                    └─────────────────────────────────────────┘
```

#### Heap

- **Where every Java object lives.** Arrays, instances, Strings (the actual chars), boxed primitives.
- Shared across all threads.
- Managed by the GC.
- Sized with `-Xms` (initial) and `-Xmx` (max).

#### JVM Stack (per thread)

- One **frame** per method invocation: local variables, operand stack, frame data.
- LIFO. Created on call, destroyed on return.
- `StackOverflowError` when stack is exhausted (deep/infinite recursion).
- Sized with `-Xss` (per-thread stack size, default ~1 MB on most platforms). Many threads × default stack = significant native memory.

#### Metaspace (replaced PermGen in Java 8+)

- Holds **class metadata**: `Klass` structures, method bytecode, constant pools, annotations.
- Lives in **native memory**, not the Java heap.
- Default: unbounded, grows until `-XX:MaxMetaspaceSize` (which is unset by default).
- `OutOfMemoryError: Metaspace` happens when classloaders leak (frameworks that reload, dynamic proxy generators, app servers).

#### Code Cache

- Stores **JIT-compiled native code**.
- Default ~240 MB. Tunable with `-XX:ReservedCodeCacheSize`.
- "CodeCache is full" warnings disable further compilation; the JVM falls back to the interpreter and slows down.

#### Other native areas

- **Direct byte buffers** (`ByteBuffer.allocateDirect`) — off-heap, used by Netty, NIO.
- **Memory-mapped files**.
- **Thread stacks** (already mentioned).
- **GC bookkeeping** (card tables, remembered sets, mark bitmaps).

> **Important:** the JVM's resident set size (RSS) = heap + metaspace + code cache + thread stacks + direct buffers + native libs + GC structures. RSS is always larger than `-Xmx`. In containers, this matters (see §13).

### 1.4 Execution Engine

The execution engine takes bytecode and runs it. Its components:

#### Interpreter

- Walks bytecode instruction by instruction.
- Fast to start, slow to run.
- Used for cold code and during the early life of every method.

#### Tiered JIT compilation

HotSpot uses **tiered compilation** by default:

| Tier | Compiler | Purpose |
|---|---|---|
| 0 | Interpreter | Bootstrap, profiling. |
| 1 | C1 (no profiling) | Fast compile; minimal optimizations. |
| 2 | C1 (with limited counters) | Moderate optimization. |
| 3 | C1 (full profiling) | Profiling for C2; modest optimization. |
| 4 | C2 | Aggressive optimization; high-quality native code. |

A method climbs tiers as it heats up. Tier 4 produces code competitive with hand-written C++. **Graal** can replace C2 (`-XX:+UseJVMCICompiler`) and produces even better code on some workloads.

#### Speculative optimization and deoptimization

C2 makes assumptions ("this call site only ever sees `LinkedHashMap`", "this branch is never taken"). If reality contradicts the assumption, the JVM **deoptimizes**: it discards the compiled code and falls back to the interpreter, recompiling later. This is normal — but excessive deopts hurt performance and show up in JFR.

#### What the JIT does

- Inlining (the most important optimization).
- Escape analysis → scalar replacement / stack allocation of objects that don't escape.
- Loop unrolling, vectorization, dead code elimination, branch prediction.
- Lock elision, lock coarsening.
- Devirtualization (turning virtual calls into direct calls when only one implementation exists).

#### Why this matters

The first few seconds of a JVM process are slow (interpreted + warming up). After warmup, performance jumps. If your benchmarks don't include warmup, you're measuring the wrong thing.

---

## 2. Memory Model Deep Dive

This section is about **heap layout** and **object allocation flow** — the substrate every GC operates on.

### 2.1 Generational hypothesis

The empirical observation behind every generational GC:

> **Most objects die young.** A small percentage live for a long time.

So: separate "young" objects (collected often, cheaply) from "old" objects (collected rarely, expensively). This is the basis for **young/old generation** designs in Serial, Parallel, and G1.

ZGC and Shenandoah are *not* generational by default in older Java versions — they treat the heap uniformly. **ZGC became generational in Java 21** (`-XX:+ZGenerational`), bringing a massive improvement on most workloads.

### 2.2 Heap structure (generational GCs)

```
┌──────────────────────────────────────────────────────────┐
│                          Heap                            │
├───────────────────────────┬──────────────────────────────┤
│       Young Generation    │      Old Generation          │
│  ┌─────┬───────┬───────┐  │  ┌────────────────────────┐  │
│  │Eden │ S0    │ S1    │  │  │ Tenured                │  │
│  └─────┴───────┴───────┘  │  └────────────────────────┘  │
└───────────────────────────┴──────────────────────────────┘
```

- **Eden** — almost all new objects start here.
- **Survivor 0 / Survivor 1** — small "from"/"to" spaces; objects that survive a Young GC bounce between them, getting a +1 age each time.
- **Tenured (Old)** — objects that have survived enough young GCs (or were too big for Eden) end up here.

Default sizing (Parallel/G1, before tuning) puts ~⅓ of the heap in young, ~⅔ in old, but this is dynamic.

### 2.3 Object allocation flow (step by step)

```
1. new MyObject()           ← bytecode "new"
2. Try TLAB allocation      ← thread-local Eden slice
   ├── fits → bump pointer, done. ← fast path, no synchronization
   └── doesn't fit:
        a) refill TLAB from Eden (with a lock)
        b) allocate directly in Eden (rare)
3. If Eden is full → trigger Young GC (Minor GC)
4. After GC, retry allocation
5. If still no space → may trigger Old GC / Full GC
6. If allocation is "humongous" (> ½ region for G1) → directly into Old/Humongous regions
```

#### TLAB (Thread-Local Allocation Buffer)

Each thread has a small chunk of Eden it can allocate into without locks. Allocation is just a pointer bump:

```
tlab.top += object_size; return tlab.top - object_size;
```

This is why allocation in Java is essentially free for short-lived objects — until the TLAB fills up and you need a new one (which involves a lock).

### 2.4 Promotion to old generation

A Young GC works like this:

1. Find live objects in **Eden + the current "from" Survivor**.
2. Copy them to the empty "to" Survivor.
3. Increment each survivor's age by 1.
4. If age ≥ `MaxTenuringThreshold` (default 15) **or** the to-Survivor overflows, copy directly to **Old**.
5. Eden + from-Survivor are now empty (they were freed wholesale).
6. Swap roles: from ↔ to.

Promotion happens for three reasons:

- **Aging promotion** — survived enough Young GCs.
- **Premature promotion** — Survivor too small to hold all live young objects.
- **Direct allocation** — object is too large for Eden ("humongous" in G1 terms).

### 2.5 Worked example

```
Heap: -Xms2g -Xmx2g, Young = 800m, Old = 1200m, Survivors 80m each, Eden 640m
MaxTenuringThreshold = 15
```

Step 1 — App allocates 500 MB of small short-lived objects. They land in Eden via TLABs.

Step 2 — Eden fills. Young GC runs. 99% of those objects are unreachable (request scope ended). 5 MB of survivors are copied to S0. Eden is wiped.

Step 3 — More allocation. Another Young GC. The 5 MB in S0 + new survivors go to S1 with age=2. S0 wiped.

Step 4 — After ~15 Young GCs, the long-lived ones reach age 15 → promoted to Old.

Step 5 — Eventually Old fills enough to trigger an Old/Full GC. This is far more expensive.

### 2.6 Important sizes and flags

| Region | Default behavior | Key flags |
|---|---|---|
| Heap | Computed from RAM | `-Xms`, `-Xmx` |
| Young/Old ratio (Parallel) | 2:1 (Old/Young) | `-XX:NewRatio`, `-XX:NewSize`, `-XX:MaxNewSize` |
| Eden/Survivor (Parallel) | 8:1:1 | `-XX:SurvivorRatio` |
| TLAB | Auto-sized | `-XX:+UseTLAB`, `-XX:TLABSize` |
| Tenuring threshold | 15 | `-XX:MaxTenuringThreshold` |
| Metaspace | Unlimited (native) | `-XX:MaxMetaspaceSize` |
| Code cache | ~240 MB | `-XX:ReservedCodeCacheSize` |

> **G1's heap layout is different**: it splits the heap into ~2048 fixed-size **regions**, each dynamically labeled Eden, Survivor, Old, or Humongous. Same generational logic, different bookkeeping.

---

## 3. Garbage Collection Fundamentals

### 3.1 Why GC exists

Manual memory management (C/C++) is fast but error-prone: use-after-free, double-free, leaks, security vulnerabilities. GC trades a bit of CPU for **memory safety**: you can't free memory that's still in use, you can't double-free, and forgotten objects are eventually reclaimed.

GC is not "free." Every collector trades among:

- **Throughput** — % of time spent running app code vs running GC.
- **Latency / pause time** — duration of stop-the-world pauses.
- **Footprint** — extra memory the GC needs to operate.

You can optimize for two of three; the third suffers.

### 3.2 Reachability

A GC's job is to find **garbage**: objects unreachable from any live root. The algorithm is:

1. Start from the **GC roots**.
2. Traverse reference fields. Every object you reach is **live**.
3. Anything not reached is **garbage** — its memory can be reclaimed.

That's the whole core idea. Every collector implements this differently, but the model is the same.

### 3.3 GC Roots

The "starting points" of reachability. A few categories:

| Root category | Why it's a root |
|---|---|
| Local variables on thread stacks | A running thread can use them at any time. |
| Active thread objects themselves | Threads must not vanish while running. |
| Static fields of loaded classes | Class-level state. |
| JNI global references | Native code holds them. |
| Synchronization monitors | Locks held by threads. |
| System classes | Bootstrap-loaded classes. |
| (G1/ZGC) Remembered set entries | Inter-generational/inter-region references. |

If you can't draw a path of references from a root to your object, your object is dead.

### 3.4 Stop-the-world (STW) pauses

Most GCs need at least some phases where **all application threads are paused** so the GC can work on a consistent snapshot of memory. During an STW pause:

- No Java code runs.
- HTTP requests pile up.
- Health checks may fail.
- p99 latency spikes.

GCs differ by **how often** and **how long** STW pauses happen, and **which work** they do concurrently with the app. Modern collectors (G1, ZGC, Shenandoah) try to do as much as possible concurrently and keep STW pauses short.

### 3.5 Safepoints

The JVM can only stop threads at **safepoints** — well-defined bytecode points where the thread's state is consistent (typically loop back-edges, method entries/exits). Triggering a GC means setting a flag and waiting until **all** threads reach a safepoint.

If one thread takes a long time to reach a safepoint (e.g., a tight counted loop with `int` counter and no safepoint poll), the entire JVM pauses waiting. This is **time-to-safepoint (TTSP)** latency, and it's a real production issue. JFR shows it.

### 3.6 Young GC vs Old GC vs Full GC

| Type | Scope | Cost | Frequency |
|---|---|---|---|
| **Young (Minor)** | Young gen only | Cheap (small region, copy live) | Often |
| **Mixed (G1)** | Young + some Old regions | Medium | Periodically |
| **Old (concurrent)** | Old gen, much done concurrently | Higher | Less often |
| **Full** | Entire heap, single-threaded fallback | **Expensive, multi-second possible** | Should be rare; fix if frequent |

A "Full GC" in modern collectors is a **fallback** when concurrent collection can't keep up. Frequent Full GCs are an alarm bell.

---

## 4. Garbage Collectors Deep Dive

### 4.1 Serial GC

**How it works.** Single-threaded, stop-the-world for both young and old generations. Mark-copy in young, mark-sweep-compact in old.

**Pros.** Simple, low overhead, low memory footprint. Fast for tiny heaps.

**Cons.** All cores idle during collection. Pauses scale with heap size.

**When to use.** Small heaps (< 100 MB), single-CPU containers, CLI tools, build tools.

**When not to use.** Anything multi-threaded with throughput needs.

**Flag.** `-XX:+UseSerialGC`.

### 4.2 Parallel GC (Throughput collector)

**How it works.** Multiple GC threads working in parallel during STW pauses. Mark-copy young, mark-sweep-compact old. Both phases STW.

**Pros.** Highest throughput on large CPU-bound workloads. Simple. Good for batch jobs.

**Cons.** Pauses can be long (hundreds of ms to seconds on big heaps). Not latency-friendly.

**When to use.** Batch processing, ETL, scientific compute. Anywhere total job time matters more than tail latency.

**When not to use.** User-facing services where p99 matters.

**Flag.** `-XX:+UseParallelGC`. Was the default until Java 9.

### 4.3 G1 GC (Garbage-First)

**The default since Java 9.** This is the one you'll meet most often in production.

#### Heap layout

G1 splits the heap into **~2048 equal-sized regions** (default region size 1 MB — 32 MB depending on heap). Each region is dynamically tagged Eden, Survivor, Old, or Humongous. There's no contiguous Eden block — Eden is a *set* of regions.

```
[E][E][O][O][S][E][O][H][H][E][O][ ][O][E][S][O] ... 2048 regions
```

#### How it works

- **Young GCs (STW):** evacuate live objects from Eden + Survivor regions to other regions. Like Parallel, but per-region.
- **Concurrent marking cycle:** runs alongside the application, marks live objects in Old regions. Does not pause the app for the bulk of the work.
- **Mixed GCs (STW):** evacuate Young + a chosen subset of Old regions whose collection has highest "garbage payoff." Hence "Garbage-First": pick the regions with the most garbage.
- **Full GC (STW, fallback):** if G1 can't keep up — single-threaded compact of the whole heap (or parallel since recent JDKs). You don't want this.

#### Pause-time goal

You give G1 a target pause time (`-XX:MaxGCPauseMillis`, default 200 ms). G1 sizes Young and chooses how many Old regions to include in mixed GCs to **try** to meet that goal. It is a target, not a guarantee.

#### Pros

- Predictable pause times for mid-to-large heaps (4–32 GB).
- Concurrent marking decouples Old GC cost from heap size.
- Region-based design handles fragmentation gracefully.
- Mature, widely deployed default.

#### Cons

- More complex than Parallel; more bookkeeping (remembered sets, card tables) → ~5–10% memory overhead.
- Longer pauses than ZGC/Shenandoah on very large heaps.
- Humongous allocations (objects > ½ region size) can fragment and trigger full GCs.

#### When to use

- Default for almost any backend service.
- Heaps 4–32 GB with 100–500 ms p99 pause budget.

#### When not to use

- Sub-10 ms pause budgets → ZGC.
- Tiny heaps (< 1 GB) → Parallel may be simpler.
- Short-lived processes where startup dominates → Serial.

**Flag.** `-XX:+UseG1GC` (default).

### 4.4 ZGC

**A concurrent, region-based, low-latency collector.** Single-digit millisecond pauses on heaps of 100 GB+. Default pause goal: under 1 ms (yes, really).

#### How it works

ZGC does almost everything **concurrently** with the application:

- Concurrent marking.
- Concurrent relocation (moving live objects).
- Concurrent reference processing.

It uses **colored pointers** (mark bits stored in unused high bits of references) and **load barriers** that intercept object reads to fix up moved references on the fly. STW pauses are very short and bounded — basically just the root scanning.

Java 21 introduced **generational ZGC** (`-XX:+ZGenerational`), which adds the generational hypothesis to ZGC. Throughput is significantly higher than non-generational ZGC. **Use it.**

#### Pros

- Sub-millisecond pauses on enormous heaps.
- Pause times don't scale with heap size — they scale with root set size.
- Concurrent everything = throughput close to Parallel for many workloads.

#### Cons

- Higher CPU and memory overhead than G1 (10–20% more).
- Allocation rate (not heap size) is what stresses ZGC.
- Slightly less mature; fewer real-world tuning knobs (this is mostly a feature).

#### When to use

- Latency-critical services with strict tail-latency SLOs.
- Large heaps (32 GB – terabytes).
- Workloads where p99/p999 pauses are visible to users.

#### When not to use

- Tiny heaps where Parallel/G1 already meet your goals.
- CPU-starved environments where ZGC's overhead hurts.

**Flag.** `-XX:+UseZGC`. With Java 21+, prefer `-XX:+UseZGC -XX:+ZGenerational`.

### 4.5 Shenandoah

A low-latency collector developed by Red Hat, similar in goals to ZGC. Uses **Brooks pointers** (each object has a forwarding pointer used for concurrent compaction).

#### Pros

- Sub-10 ms pauses, often sub-millisecond on modern versions.
- Available in OpenJDK builds (RHEL/AdoptOpenJDK Temurin Eclipse).
- Concurrent compaction — no stop-the-world for moves.

#### Cons

- Forwarding pointer per object adds memory overhead.
- Less common; smaller community than G1/ZGC.

#### When to use

- Same use cases as ZGC, especially on Red Hat-backed JDK distributions.
- When you want low-latency on Java 11 (where ZGC was less mature).

#### When not to use

- Throughput-critical batch jobs.
- If you're already on a JDK with ZGC and have no specific reason to switch.

**Flag.** `-XX:+UseShenandoahGC`.

### 4.6 Comparison table

| GC | Pause goal | Concurrency | Heap sweet spot | Throughput | Memory overhead | Default in |
|---|---|---|---|---|---|---|
| Serial | High | None | < 100 MB | Low | Lowest | Tiny / single-CPU |
| Parallel | High | None (parallel STW) | 1–8 GB batch | **Highest** | Low | Pre-Java 9 |
| **G1** | Medium (~200 ms) | Concurrent marking | 4–32 GB | High | Medium | **Java 9+** |
| ZGC | Very low (< 1 ms) | Almost everything | 32 GB – TBs | High | Higher | Latency apps |
| Shenandoah | Very low | Almost everything | 8 GB – TBs | Medium-High | Higher | Red Hat builds |

---

## 5. GC Algorithms and Concepts

These are the **building blocks** every collector composes.

### 5.1 Mark-and-Sweep

1. **Mark.** Traverse from roots, mark every reachable object.
2. **Sweep.** Walk the heap, free unmarked objects.

**Pros.** Simple. Doesn't move objects (cheaper for large objects, native handles).

**Cons.** **Fragmentation**: free space is scattered, large allocations may fail even with plenty of total free memory.

**Used by.** Parts of CMS (deprecated). Conceptual basis for many concurrent markers.

### 5.2 Mark-and-Compact

1. Mark live objects.
2. **Compact**: slide all live objects to one end of the heap. Free space is a single contiguous block.

**Pros.** No fragmentation. Allocation is a pointer bump.

**Cons.** Moving objects is expensive; pointers must be updated. Usually requires STW (or a clever forwarding scheme like Brooks pointers / colored pointers).

**Used by.** Old gen of Parallel, Serial. G1 uses a region-level variant.

### 5.3 Copying (Stop-and-Copy)

1. Two semi-spaces: "from" and "to".
2. Allocate in "from" until it fills.
3. Copy live objects from "from" to "to".
4. Wipe "from" wholesale. Swap roles.

**Pros.** Very fast for sparse young gens. Compaction for free. No fragmentation.

**Cons.** Wastes half the memory (only one space active). Cost is proportional to live data, **not** to total allocated.

**Used by.** Eden → Survivor and Survivor → Survivor copies in all generational collectors.

### 5.4 Generational hypothesis

Already covered in §2.1. The combination is:

- **Young gen:** copying (fast for the "most things die young" case).
- **Old gen:** mark-compact or concurrent variants (because copying wastes space when most objects are live).

This combination is why generational GCs dominated: each phase uses the algorithm best suited to its data's age distribution.

### 5.5 Concurrent vs incremental

- **Concurrent** — GC threads run while app threads run.
- **Incremental** — GC work is broken into small chunks; the app pauses briefly between chunks.
- **Parallel** — multiple GC threads run during STW.

Modern low-latency collectors are **concurrent + parallel**.

### 5.6 Tri-color marking

A standard concurrent-marking technique:

- **White** — not yet visited (assumed garbage).
- **Gray** — visited, but children not yet scanned.
- **Black** — visited and children scanned.

Concurrent marking must handle the case where a black object gets a reference to a white object after it's already been scanned. This requires **write barriers** to track such mutations and re-scan them. SATB (Snapshot-At-The-Beginning, used by G1) and incremental update (used by some others) are two approaches.

### 5.7 Trade-offs

- **More frequent collections** → smaller per-collection cost, more total CPU.
- **Larger heap** → less frequent collections, but each one scans more.
- **More survivor space** → fewer premature promotions, more young-gen overhead.
- **Concurrent algorithms** → less pause, more total CPU + memory overhead.

There is no free lunch; every knob trades one cost for another.

---

## 6. JVM Tuning Basics

### 6.1 The first principle

**Don't tune blindly.** Measure first. The default G1 in Java 17/21 is an excellent starting point for nearly every Spring Boot service. Reach for tuning only when you have **data** showing it's needed.

### 6.2 Heap sizing

#### `-Xms` and `-Xmx`

- `-Xms` — initial heap size.
- `-Xmx` — max heap size.

**In production: set them equal.** Why:
- Avoids resize jitter (JVM growing/shrinking during traffic spikes).
- Eliminates a class of intermittent latency from heap expansion.
- Makes container memory accounting predictable.

```
-Xms4g -Xmx4g
```

#### Picking a value

- Start from your **actual live set** (steady-state used heap after a Young/Old GC). Multiply by **2–3×**.
  - Live set 1 GB → heap 2–3 GB.
- Less than 2× live set → constant GC churn.
- More than 3× live set → wasted memory, longer GC pauses.

#### Container-aware sizing

In containers, prefer **percentage of available memory** flags so the JVM respects cgroup limits:

```
-XX:InitialRAMPercentage=75.0
-XX:MaxRAMPercentage=75.0
```

Leave 25% for native memory, metaspace, code cache, thread stacks, OS, and headroom. See §13.

### 6.3 GC selection

| Goal | GC | Flag |
|---|---|---|
| Default backend service | G1 | `-XX:+UseG1GC` (default) |
| Latency-critical service | Generational ZGC | `-XX:+UseZGC -XX:+ZGenerational` |
| Batch / throughput | Parallel | `-XX:+UseParallelGC` |
| Tiny heap, single-CPU | Serial | `-XX:+UseSerialGC` |

### 6.4 Pause time vs throughput

Three classic levers:

| Want | Flag | Effect |
|---|---|---|
| Lower pauses | `-XX:MaxGCPauseMillis=100` | G1 picks smaller young / fewer old regions per mixed GC. May reduce throughput. |
| More throughput | Larger heap, larger young gen | Fewer GCs, but each takes longer. |
| Predictable steady state | `-Xms == -Xmx` | No resize-induced jitter. |

### 6.5 A sane starting config for Spring Boot in Kubernetes

```bash
java \
  -XX:InitialRAMPercentage=75.0 -XX:MaxRAMPercentage=75.0 \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+ParallelRefProcEnabled \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/log/app/heap.hprof \
  -XX:+ExitOnOutOfMemoryError \
  -Xlog:gc*,safepoint:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=10,filesize=20m \
  -jar app.jar
```

What each does:
- **RAMPercentage** — container-aware heap sizing.
- **G1GC** — default, balanced collector.
- **MaxGCPauseMillis=200** — pause goal.
- **ParallelRefProcEnabled** — parallelize Reference object processing (cheap win in big apps with many WeakReferences).
- **HeapDumpOnOutOfMemoryError** — when (not if) you OOM, you'll need this.
- **ExitOnOutOfMemoryError** — let the orchestrator restart you instead of running in a half-broken state.
- **`-Xlog:gc*`** — unified logging (Java 9+); essential for any post-mortem.

---

## 7. Advanced JVM Tuning

### 7.1 G1 tuning parameters

| Flag | Default | When to change |
|---|---|---|
| `-XX:MaxGCPauseMillis` | 200 | Lower for stricter latency; raises GC frequency. |
| `-XX:G1HeapRegionSize` | Auto (1–32 MB) | Set manually if you have many humongous allocations. |
| `-XX:G1NewSizePercent` | 5 | Min young size; raise if young is too small under load. |
| `-XX:G1MaxNewSizePercent` | 60 | Max young size. |
| `-XX:G1MixedGCCountTarget` | 8 | How many mixed GCs to spread Old reclamation across. |
| `-XX:InitiatingHeapOccupancyPercent` (IHOP) | Adaptive | Threshold to start concurrent marking. |
| `-XX:G1ReservePercent` | 10 | Headroom to avoid evacuation failures. |
| `-XX:ParallelGCThreads` | ~5/8 of CPUs | GC threads during STW; usually leave default. |
| `-XX:ConcGCThreads` | ¼ of ParallelGCThreads | Concurrent marker threads. |

**G1 anti-patterns:**

- Setting `MaxGCPauseMillis` very low (50 ms) on a multi-GB heap → G1 thrashes, throughput tanks.
- Forcing `NewRatio` / `NewSize` flags from Parallel — they don't apply to G1 in the same way.

### 7.2 ZGC tuning basics

ZGC has **few** knobs, deliberately. The main ones:

| Flag | Purpose |
|---|---|
| `-XX:+UseZGC` | Enable ZGC. |
| `-XX:+ZGenerational` (Java 21+) | Generational ZGC — almost always on. |
| `-XX:SoftMaxHeapSize` | Soft cap; ZGC tries to stay below this. |
| `-XX:ConcGCThreads` | Concurrent GC threads; default is fine. |
| `-XX:+UseLargePages` | If your kernel supports huge pages, helps ZGC. |

ZGC's main "tuning" is **giving it enough headroom**. If allocation rate exceeds the rate ZGC can free memory, you get an "Allocation Stall" (app threads block). The fix is a larger heap or a smaller allocation rate.

### 7.3 Thread tuning

#### Stack size

`-Xss512k` — reduce per-thread stack to 512 KB if you have thousands of threads (default is ~1 MB on Linux x64). Tomcat with 1000 worker threads at 1 MB = 1 GB of native memory just for stacks.

#### Thread pool sizing

- CPU-bound work → pool size ≈ number of CPUs.
- IO-bound work → larger pool, but profile to find the sweet spot.
- Virtual threads (Java 21) — for IO-bound work, replace bounded thread pools entirely.

#### Common-pool parallelism

`-Djava.util.concurrent.ForkJoinPool.common.parallelism=N` — controls parallel streams. Default = CPUs - 1. Lower it if parallel streams are competing with your other work.

### 7.4 Memory tuning strategies

#### Direct memory

`-XX:MaxDirectMemorySize=512m` — caps off-heap NIO buffers. Without this, Netty/NIO can leak direct memory until the OS kills you. **Always set it.**

#### Metaspace

`-XX:MaxMetaspaceSize=512m` — caps native class metadata. Set it to detect classloader leaks (you'll OOM on Metaspace instead of growing forever).

#### Code cache

`-XX:ReservedCodeCacheSize=512m` — raise on apps with thousands of compiled methods (microservices with lots of frameworks). Watch for "CodeCache is full" warnings in logs.

#### String dedup (G1)

`-XX:+UseStringDeduplication` — G1 deduplicates equal `String`s during GC. Saves memory in apps with many redundant strings (logging, JSON parsing, web frameworks).

### 7.5 JIT tuning (rarely needed)

| Flag | Use |
|---|---|
| `-XX:CompileThreshold` | Lower = compile sooner, longer warmup CPU. |
| `-XX:-TieredCompilation` | Disable tiered (rarely a good idea). |
| `-XX:+PrintCompilation` | See what's being compiled. |

Modern advice: **don't touch JIT flags** unless JFR shows you a specific problem. The default tiered JIT is excellent.

### 7.6 Class data sharing (CDS / AppCDS)

Pre-archive class metadata to disk → faster startup, lower memory.

```bash
java -XX:ArchiveClassesAtExit=app.jsa -jar app.jar
java -XX:SharedArchiveFile=app.jsa -jar app.jar
```

Worth setting up in containers where you start many short-lived JVMs (CI runners, serverless).

---

## 8. GC Logging and Analysis

### 8.1 Enabling logs (Java 9+ unified logging)

```
-Xlog:gc*,safepoint:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=10,filesize=20m
```

This logs:
- All `gc*` tags (gc, gc+heap, gc+phases, gc+age, gc+ergo …).
- `safepoint` (TTSP and pause durations).
- To `gc.log`, with rotation (10 files of 20 MB each).
- With timestamp, uptime, log level, and tags.

In Java 8, the equivalent is the older `-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log` family. Java 8 is end-of-life for everything we're discussing — use Java 17+ unified logging.

### 8.2 Reading a G1 log line

A young GC line looks like:

```text
[2025-05-02T10:42:31.123+0000][12.456s][info][gc] GC(42) Pause Young (Normal) (G1 Evacuation Pause) 1024M->256M(2048M) 32.456ms
```

Decoded:
- `GC(42)` — sequence number.
- `Pause Young (Normal)` — STW young collection.
- `(G1 Evacuation Pause)` — triggered because Eden filled.
- `1024M->256M(2048M)` — heap occupancy **before → after** out of total. So 768 MB of garbage was reclaimed.
- `32.456ms` — STW pause duration.

A concurrent-marking cycle looks like:

```text
GC(50) Concurrent Cycle
GC(50) Pause Remark 64M->63M(2048M) 5.123ms
GC(50) Pause Cleanup 63M->62M(2048M) 0.456ms
GC(50) Concurrent Cycle 234.567ms
```

A mixed GC:

```text
GC(60) Pause Young (Mixed) (G1 Evacuation Pause) 1900M->600M(2048M) 80.123ms
```

A Full GC (you do not want to see this often):

```text
GC(75) Pause Full (G1 Evacuation Pause) 2000M->1900M(2048M) 1234.567ms
```

### 8.3 What to extract

| Metric | How |
|---|---|
| **GC overhead** | sum(STW pauses) / wall time. > 5% is concerning, > 10% is bad. |
| **Pause distribution** | p50, p95, p99 of pause durations. |
| **Allocation rate** | Δ(used heap before young GC over time) / Δ(time). |
| **Promotion rate** | Δ(used old) per young GC. High = young gen too small. |
| **Full GC frequency** | Should be near zero in steady state. |
| **Time-to-safepoint** | From safepoint logs. > 10 ms = a thread is taking too long to stop. |

### 8.4 Tools for analysis

- **GCEasy** (gceasy.io) — paste a log, get a report. Free tier sufficient for ad-hoc work.
- **GCViewer** — open source desktop tool.
- **JDK Mission Control** — GC analysis from JFR recordings (better than raw logs for deep dives).

### 8.5 Health check based on a log

A healthy backend service should look like:

- Young GCs every few seconds, each 10–50 ms.
- Concurrent marking cycles occasionally; not constant.
- Mixed GCs around the same length as Young.
- **No** Full GCs in steady state.
- GC overhead < 5%.

If you see sustained Full GCs, escalating pause times, or allocation rate above what the GC can keep up with — that's the start of a §12 scenario.

---

## 9. Debugging Memory Issues

### 9.1 The two big classes of problem

1. **Memory leak** — heap usage grows monotonically. Eventually OOM.
2. **High allocation pressure** — heap usage cycles fine, but allocation rate is so high that GC overhead dominates.

These have different symptoms and fixes. Don't confuse them.

### 9.2 OutOfMemoryError types

#### `java.lang.OutOfMemoryError: Java heap space`

**Cause.** App needs more heap than `-Xmx` allows, **after** GC has freed everything possible.

**Diagnose.**
1. Get a heap dump (auto-dumped if `-XX:+HeapDumpOnOutOfMemoryError` is set).
2. Open in Eclipse MAT or VisualVM.
3. Look at the **Dominator Tree** and **Leak Suspects Report**.

**Fix.**
- If actual data: increase heap.
- If a leak: find the GC root holding garbage and remove it. Common culprits: static collections, caches without eviction, listeners that aren't unregistered, ThreadLocals not cleaned up in pooled threads.

#### `java.lang.OutOfMemoryError: GC overhead limit exceeded`

**Cause.** JVM is spending > 98% of CPU in GC and freeing < 2% of heap. It gives up rather than thrash forever.

**Diagnose.** Same as heap space — heap dump + MAT.

**Fix.** Same as heap space (usually). This is "your heap is full but the JVM noticed and bailed early" — it's the same problem.

#### `java.lang.OutOfMemoryError: Metaspace`

**Cause.** Class metadata exceeded `MaxMetaspaceSize` (or pushed native memory limit).

**Diagnose.**
- `jcmd <pid> VM.metaspace` — shows used/committed.
- `jcmd <pid> GC.class_stats` — count classes per loader (requires `-XX:+UnlockDiagnosticVMOptions -XX:+UnlockExperimentalVMOptions` and `-XX:+ClassUnloadingWithConcurrentMark`).
- Suspect classloader leaks: redeploys in app servers, dynamic proxy generation (Spring AOP, Hibernate enhancers).

**Fix.**
- Raise `MaxMetaspaceSize` (short-term).
- Find the leaking classloader and stop it from being retained. MAT can show "ClassLoader" objects and what holds them.

#### `java.lang.OutOfMemoryError: Direct buffer memory`

**Cause.** Off-heap NIO buffers exceeded `MaxDirectMemorySize`.

**Diagnose.**
- `jcmd <pid> VM.native_memory` (if NMT enabled) shows direct memory.
- Look for unfreed `ByteBuffer.allocateDirect()` (Netty leak detector helps).

**Fix.** Find leaks; raise `-XX:MaxDirectMemorySize`.

#### `java.lang.OutOfMemoryError: unable to create native thread`

**Cause.** OS limit on threads or native memory exhausted (each thread costs `Xss` plus kernel overhead).

**Diagnose.**
- `cat /proc/<pid>/status | grep Threads`
- `ulimit -u`
- Cgroup `pids.max`.

**Fix.** Reduce thread count (use bounded pools or virtual threads); raise OS limits; reduce `-Xss`.

### 9.3 Leak detection workflow

1. **Confirm it's a leak.** Look at heap usage over hours/days. Does the post-GC live set grow monotonically? If yes → leak.
2. **Trigger a heap dump** at peak:
   ```bash
   jcmd <pid> GC.heap_dump /tmp/heap.hprof
   ```
3. **Open in MAT.** Run "Leak Suspects" — usually points right at the offender.
4. **Walk the dominator tree.** Find the largest retained set. Its GC root path tells you who holds it.
5. **Common offenders:**
   - `static` `Map`/`List` used as a cache without eviction.
   - `ThreadLocal` set in a thread-pool thread, never cleared.
   - Listeners/observers registered but never deregistered.
   - Long-lived cache (Caffeine, EHCache) misconfigured.
   - JDBC `Statement`s/`ResultSet`s not closed.

### 9.4 Allocation-pressure detection

If GC overhead is high but heap is healthy after each GC, you have **too much allocation**, not a leak.

1. **JFR** with `Allocation in new TLAB` and `Allocation outside TLAB` events shows top allocators by stack.
2. Reduce object churn: reuse buffers, prefer primitives, avoid streams in hot loops, `String.format` only outside hot paths, `StringBuilder` not `+`.
3. Profile (async-profiler `-e alloc`) to find the worst allocators.

### 9.5 Tools cheat sheet

| Tool | Use |
|---|---|
| Heap dump + MAT | Leak diagnosis. |
| JFR | Live profiling, allocation, GC, lock contention. |
| `jcmd <pid> GC.heap_info` | Quick heap status. |
| `jcmd <pid> VM.native_memory` | Native memory tracking (with `-XX:NativeMemoryTracking=summary`). |
| async-profiler `-e alloc` | Top allocation stacks. |

---

## 10. JVM Monitoring Tools

### 10.1 `jcmd` — your Swiss Army knife

```bash
jcmd <pid> help                        # list commands
jcmd <pid> VM.version
jcmd <pid> VM.flags                    # all current flags
jcmd <pid> VM.system_properties
jcmd <pid> GC.heap_info
jcmd <pid> GC.heap_dump /tmp/h.hprof
jcmd <pid> GC.run                      # explicit GC (for diagnostic only)
jcmd <pid> Thread.print                # full thread dump
jcmd <pid> JFR.start name=rec settings=profile duration=60s filename=/tmp/rec.jfr
jcmd <pid> JFR.dump name=rec filename=/tmp/now.jfr
jcmd <pid> VM.native_memory summary    # if NMT enabled
jcmd <pid> Compiler.codecache          # code cache state
```

**Always reach for `jcmd` first.** It works on running JVMs and supersedes most older tools.

### 10.2 `jstat` — GC statistics in real time

```bash
jstat -gcutil <pid> 1000               # % usage of each region every 1s
jstat -gc <pid> 1000                   # absolute sizes
jstat -gccause <pid> 1000              # last GC cause
```

Output columns: `S0, S1, E, O, M, CCS, YGC, YGCT, FGC, FGCT, GCT`. Useful for a 30-second triage when you can't yet pull a JFR.

### 10.3 `jmap`

```bash
jmap -heap <pid>                       # heap config + usage (deprecated; use jcmd)
jmap -histo:live <pid> | head          # top classes by retained size
jmap -dump:live,format=b,file=h.hprof <pid>
```

Mostly superseded by `jcmd`. The `-histo:live` is still handy for quick "what's eating memory."

### 10.4 `jstack`

```bash
jstack <pid>                           # thread dump
jstack -l <pid>                        # include lock info
```

Use when:
- App is hung. Deadlock detection (`jstack -l` lists held/waited locks).
- High CPU. Compare two dumps a few seconds apart; threads stuck in the same place are suspects.

`jcmd <pid> Thread.print` does the same.

### 10.5 VisualVM

GUI: heap monitor, sampler (CPU/memory), thread view, GC view, plugins for profiling. Good for ad-hoc local investigation. Less suited for production (use JFR instead).

### 10.6 Java Flight Recorder (JFR)

The **production** profiler. Built into HotSpot. Low overhead (~1%). Records everything: GC, allocation, locks, IO, JIT, custom events.

```bash
# Start a 5-minute profile recording
jcmd <pid> JFR.start name=prof settings=profile duration=5m filename=/tmp/prof.jfr

# Or always-on continuous recording
java -XX:StartFlightRecording=name=cont,settings=profile,maxsize=200m,maxage=1h ...
```

Open the `.jfr` in **JDK Mission Control**. Top views:
- **Method Profiling** — where CPU goes.
- **Memory** — allocation by class and stack.
- **Garbage Collections** — full GC timeline + per-collection details.
- **Lock Instances** — contention.
- **Hot Methods** — JIT compilation events.

JFR is a strict superset of what `jstat`, `jmap -histo`, and `jstack` can tell you, with stack traces. **Make it the default in production.**

### 10.7 async-profiler

External profiler, low overhead, AsyncGetCallTrace based — produces flame graphs. Excellent for CPU and allocation profiling.

```bash
./profiler.sh -d 60 -f cpu.html <pid>          # CPU flame graph
./profiler.sh -d 60 -e alloc -f alloc.html <pid>
./profiler.sh -d 60 -e lock -f lock.html <pid>
```

Pair with JFR: JFR for built-in events, async-profiler for CPU flame graphs.

### 10.8 Production telemetry

Don't rely on logging in alone. Export:

- **Micrometer** → Prometheus. Built-in JVM metrics: heap, non-heap, GC pause counts, thread count, classes loaded.
- **Application logs** for GC (`-Xlog:gc*`).
- **JFR continuous recording** rolled to disk (or shipped to a JFR collector).

---

## 11. Performance Optimization

### 11.1 Reduce allocation pressure

Allocation is cheap, but enough of it makes GC dominant. Targets in order:

- **Streams in hot loops** — replace with for-loops. Streams allocate `Stream`, `Spliterator`, lambdas (sometimes), boxed values.
- **Boxing** — prefer `int[]` over `List<Integer>`, `IntStream` over `Stream<Integer>`.
- **String concatenation in loops** — use `StringBuilder` once, not `+` repeatedly.
- **Date/time formatting** — `DateTimeFormatter` is thread-safe, reuse it; don't recreate per call.
- **Logging** — guard with `isDebugEnabled` if argument construction allocates; use `{}` placeholders, not `+`.
- **JSON** — pre-compile `ObjectMapper` once; reuse; use `JsonNode` for routing, typed records inside.
- **Collections** — pre-size `new ArrayList<>(expected)` when you know capacity.

### 11.2 Escape analysis and stack allocation

If C2 proves an object never escapes its method, it can:

- **Scalar replace** — explode the object into individual locals; skip allocation entirely.
- **Eliminate locks** — if only one thread can see the object.

You enable this by **writing simple, local code** without leaks across method boundaries. Don't hand-roll allocations into long-lived state if a local will do.

### 11.3 Reduce GC pressure with object reuse

- **Buffer pools** for direct byte buffers (Netty, gRPC do this).
- **Object pools** are usually a bad idea on the JVM (allocation is faster than pool synchronization for small objects). Pool only **expensive-to-construct** things: connections, threads, large buffers.

### 11.4 Cache wisely

- Caching is a memory leak with rules.
- Use Caffeine with size + time eviction, not a raw `HashMap`.
- Watch heap dumps after a soak: caches are the most common "innocent leak."

### 11.5 Thread contention

Symptoms:
- High CPU, low throughput.
- JFR `Java Monitor Wait` events with high time.
- async-profiler `-e lock` shows hotspots.

Fixes:
- Replace `synchronized` with `ConcurrentHashMap`, `LongAdder`, `AtomicReference`.
- Reduce critical-section scope.
- Use `StampedLock` or read-write locks where reads dominate.
- Switch to lock-free data structures or partition the data.

### 11.6 CPU vs memory trade-offs

Common knobs:

- **More memory → less GC** (until pauses get longer per GC).
- **Larger caches → less recompute, more memory.**
- **Compressed OOPs** (`-XX:+UseCompressedOops`, default for heaps ≤ ~32 GB) — half-size pointers, more cache-friendly. **Stay below 32 GB heap if you can** to keep this benefit.
- **String dedup** — saves memory, costs a little GC CPU.

### 11.7 Warmup

Production traffic should hit a warmed-up JVM:

- Send synthetic warmup traffic before adding to load balancer.
- Use AppCDS / class data sharing to skip class loading.
- Consider Graal or AOT for short-lived JVMs.

### 11.8 Benchmarking honestly

- Use **JMH**, not `System.currentTimeMillis()` in a loop.
- Include warmup iterations.
- Run on hardware close to production.
- Repeat; report distribution, not a single number.

---

## 12. Real-World Debugging Scenarios

### 12.1 Scenario — high GC pauses

**Symptoms.**
- p99 latency spikes correlated with GC events.
- `gc.log` shows pauses of 1–3 seconds.
- Throughput is fine; tail latency is the problem.

**Investigation.**
1. Confirm with `jstat -gcutil` and `gc.log` — count Young vs Old vs Full GCs.
2. Look for **Full GC** lines. If frequent → escalation: heap too small or leak.
3. JFR → look at allocation rate and promotion rate.
4. Check object size distribution (G1 humongous allocations show up in logs).

**Common root causes.**
- Heap too small → constant GC.
- Young gen too small → frequent premature promotion → frequent old GCs.
- Humongous allocations (large arrays) → G1 fragmentation.
- Excessive allocation (some hot method allocating millions of objects/sec).

**Solutions.**
- Right-size heap (live set × 2–3).
- Reduce allocation in hot paths (see §11).
- Switch to ZGC if heap is large and pause-budget tight.
- For humongous allocations: increase `G1HeapRegionSize` or refactor to avoid the giant objects.

### 12.2 Scenario — memory leak

**Symptoms.**
- Heap usage **after** Old GC creeps up over hours/days.
- Eventually `OutOfMemoryError: Java heap space`.
- Grafana RSS chart looks like a sawtooth that never resets.

**Investigation.**
1. Confirm: `jstat -gcutil <pid> 60000` for 30 minutes; watch O column post-GC.
2. Take heap dump near the peak: `jcmd <pid> GC.heap_dump /tmp/h.hprof`.
3. Open in Eclipse MAT → Leak Suspects.
4. Walk dominator tree: who retains the largest object set?
5. Path to GC root → who's holding it?

**Common root causes.**
- `static` cache without eviction.
- `ThreadLocal` not cleaned up in a pooled thread.
- Listeners/callbacks not unregistered.
- A growing `Map` keyed by something that never shrinks (request IDs, etc.).
- Soft/Weak references not actually weak (held by another strong reference).

**Solutions.**
- Fix the offending code path: remove static cache, add eviction (Caffeine), clear ThreadLocal in a try/finally.
- Re-test under load; confirm post-GC heap is flat.

### 12.3 Scenario — CPU spike

**Symptoms.**
- One pod pinned at 100% CPU.
- Latency degraded but no errors.
- GC overhead is *not* high (so it's not GC).

**Investigation.**
1. `top -H -p <pid>` → identify the hot OS threads.
2. Convert thread IDs to hex.
3. `jstack <pid> | grep -A 20 <hex_tid>` → see what those threads are doing.
4. Or: async-profiler `-d 60 -f cpu.html <pid>` → CPU flame graph.
5. JFR continuous recording, dump `Method Profiling` view.

**Common root causes.**
- A regex with catastrophic backtracking on adversarial input.
- A tight loop on a poison message.
- A bad `equals`/`hashCode` causing full bucket scans on a `HashMap`.
- An unintended infinite retry.
- JIT deoptimization storms (rare, visible in JFR).

**Solutions.**
- Identify the hot stack from the flame graph; fix the algorithm or short-circuit.
- For regex: use bounded patterns or DFAs (`re2j`).
- For retry storms: cap retries, add backoff.

### 12.4 Scenario — thread deadlock

**Symptoms.**
- Some endpoints hang indefinitely.
- Thread count keeps growing as new requests pile up.
- Health checks may pass (event loop alive) while workers are stuck.

**Investigation.**
1. `jstack -l <pid>` (or `jcmd <pid> Thread.print`).
2. Look for **"Found one Java-level deadlock"** — JVM auto-detects classic deadlocks.
3. If no deadlock detected but threads are stuck: search for many threads in `BLOCKED` waiting on the same monitor.
4. Trace lock acquisition order across the stack traces.

**Common root causes.**
- Two locks acquired in different orders by different threads.
- Reentrant lock held by a thread waiting on a result that requires the same lock.
- DB connection pool exhausted → all worker threads waiting on connections held by other worker threads.
- Async chain that synchronously waits on its own pool (Reactor / CompletableFuture pitfalls).

**Solutions.**
- Always acquire locks in a consistent order; document it.
- Use `tryLock` with timeout instead of indefinite blocking.
- Never block a pool thread waiting on a result computed by the same pool.
- Size connection pools properly.

### 12.5 Scenario — slow response times

**Symptoms.**
- p50 fine, p99 awful.
- No CPU spike, no GC issue, no thread pool exhaustion.
- Sometimes a request takes 500 ms; usually 20 ms.

**Investigation.**
1. JFR profile during a peak. Look at:
   - `Java Thread Park` and `Monitor Wait` (lock contention?).
   - Network / IO events (slow downstream?).
   - GC events correlated with slow requests?
   - Safepoint operations (long TTSP)?
2. Distributed tracing (OpenTelemetry) — is the slowness in Java or in a downstream call?
3. Check downstream histograms (DB query times, external API latencies).

**Common root causes.**
- **Downstream latency** (DB, cache, external API). The JVM is fine.
- **Lock contention** under load.
- **GC tail pauses** that correlate with the bad requests (look at GC log timestamps).
- **Safepoint pauses** from a thread taking too long to safepoint (rare but real).
- **Cold cache** — JIT just deoptimized something; warmup again.

**Solutions.**
- Most "slow Java" is actually slow downstream. Fix the right system.
- For lock contention: §11.5.
- For GC: §12.1.
- For TTSP: usually a counted loop without safepoint poll; rewrite with `long` counter or break into batches.

---

## 13. JVM in Containers

This is where most production JVMs run, and where most production surprises happen.

### 13.1 Cgroup awareness

Modern JVMs (Java 10+, fully reliable in 11+) detect cgroup limits and use them for:

- Available memory (`-XX:MaxRAMPercentage`).
- Available CPUs (`Runtime.availableProcessors()`).

**On Java 17/21, you don't need `-XX:+UseContainerSupport`** — it's on by default. Keep your JVM up to date and this Just Works.

### 13.2 Memory: heap is not RSS

```
container memory limit
  ≥ heap (-Xmx)
  + metaspace
  + code cache
  + thread stacks (N × Xss)
  + direct buffers (MaxDirectMemorySize)
  + native libraries
  + GC bookkeeping
```

A pod with `memory: 2Gi` and `-Xmx2g` will OOMKill within minutes — because there's nothing left for everything else.

**Rule of thumb.** Heap = 50–75% of container memory.

```yaml
resources:
  requests: { memory: "2Gi" }
  limits:   { memory: "2Gi" }
env:
  - name: JAVA_OPTS
    value: >-
      -XX:InitialRAMPercentage=70.0
      -XX:MaxRAMPercentage=70.0
      -XX:MaxDirectMemorySize=256m
      -XX:MaxMetaspaceSize=256m
      -XX:ReservedCodeCacheSize=256m
      -Xss512k
```

### 13.3 OOMKilled vs OutOfMemoryError

| Signal | What it means |
|---|---|
| Pod **OOMKilled** (kernel) | Native RSS exceeded the cgroup limit. JVM never got a chance to react. |
| Java `OutOfMemoryError` | JVM ran out of heap (or metaspace, etc.) but the process is alive. |

If you see OOMKilled and **no** `OutOfMemoryError` in logs → it's native memory. Look at metaspace, direct buffers, threads, off-heap caches.

### 13.4 CPU limits and threads

`Runtime.availableProcessors()` reads the cgroup CPU quota. The JVM sizes thread pools (ForkJoinPool, GC threads, etc.) from this number.

**Pitfalls:**
- CPU limit of 0.5 cores → `availableProcessors() = 1` → ParallelGCThreads might be too few for the heap → Young GCs slow.
- Aggressive CPU throttling pauses Java threads at random — looks like GC pause but isn't.

**Solution.** Give containers enough CPU. Don't over-pack JVMs into small CPU slices.

### 13.5 Native Memory Tracking in containers

```
-XX:NativeMemoryTracking=summary
```

Then:

```bash
jcmd <pid> VM.native_memory summary
```

Shows **all** memory the JVM uses, broken down by category. Indispensable for OOMKilled diagnostics.

### 13.6 Container-friendly defaults checklist

```
-XX:InitialRAMPercentage=75.0
-XX:MaxRAMPercentage=75.0
-XX:MaxDirectMemorySize=...
-XX:MaxMetaspaceSize=...
-XX:ReservedCodeCacheSize=...
-Xss512k
-XX:+ExitOnOutOfMemoryError
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/app/heap-${HOSTNAME}.hprof
-Xlog:gc*,safepoint:file=/var/log/app/gc.log:time,uptime:filecount=10,filesize=20m
-XX:NativeMemoryTracking=summary
```

Use a writable volume mounted at `/var/log/app` so heap dumps and GC logs survive a crash.

### 13.7 Small-heap, low-latency: native images

Spring Boot 3 + GraalVM native image gives you fast startup and tiny RSS — at the cost of no JIT and limited reflection. Useful for CLI tools, lambdas, sidecar daemons. Production hotpath services usually still benefit from C2's deeper optimizations on the JVM.

---

## 14. Common Pitfalls

### 14.1 Wrong heap sizing

- Heap = container memory → OOMKilled.
- Heap < live set × 2 → constant GC.
- `-Xms` ≠ `-Xmx` in production → resize jitter.
- Crossing the 32 GB compressed-OOPs boundary unnecessarily → wasted memory.

### 14.2 Ignoring GC logs

You don't know what your service is doing if you don't have GC logs. **Enable them. Always.** Diagnosing post-incident from "the app got slow" without GC logs is guessing.

### 14.3 Over-tuning

Common mistakes:
- Copy-pasting GC flags from a blog post written in 2014.
- Setting 20 G1 flags "just in case."
- Switching collectors based on hearsay rather than data.

**Default G1 + correct heap size** beats most hand-tuned configurations.

### 14.4 Misunderstanding metrics

- Heap "used" metric: includes garbage. Look at **post-GC** used (Old gen after Old GC).
- "GC pause time" without context: a 50 ms pause is fine for some apps, terrible for others.
- "Allocation rate" without throughput: high allocation can be fine if throughput is high too.

Always pair a metric with the workload context.

### 14.5 Profiling on the wrong workload

Microbenchmarks lie. Local laptop != production hardware. Synthetic load != real user mix. Whenever possible, profile in **a real environment with real traffic** (canary, JFR continuous).

### 14.6 Forgetting native memory

Heap is not the whole story. Direct buffers, metaspace, code cache, thread stacks all live in native memory. If you don't bound them, the OOMKiller eventually does.

### 14.7 Treating Full GC as normal

It isn't. Frequent Full GCs mean your collector failed to keep up — usually because heap is too small or allocation is too high. Don't tune around them; fix the cause.

### 14.8 Mixing tuning advice across collectors

Parallel and G1 have different mental models. `-XX:NewRatio` is meaningful in Parallel, less so in G1. Read the docs for **the collector you're using**, not generic advice.

---

## 15. When NOT to Tune the JVM

### 15.1 When the bottleneck is the database

- Slow queries, missing indexes, lock contention, connection pool exhaustion.
- The JVM is sitting at 5% CPU waiting for DB I/O.

If your tracing shows 80% of request time is in `SELECT` calls, no JVM flag will help. Fix the query.

### 15.2 When the bottleneck is the network

- p99 dominated by external API calls.
- Requests stuck in `parking` waiting for response.
- Excessive cross-AZ traffic.

The JVM can't make the network faster. Cache, batch, parallelize, or move services closer.

### 15.3 When the algorithm is wrong

- O(n²) where O(n log n) was possible.
- Linear scans where a hash map was possible.
- Recomputing the same value in a hot loop.

Profile, fix the code, then revisit tuning.

### 15.4 When the architecture is wrong

- One huge monolith doing everything synchronously.
- Sync calls where async would do.
- No caching where caching is obvious.

Architectural fixes give 10× wins. JVM tuning gives 10–20% at most. Don't paint over a structural problem.

### 15.5 The litmus test

Before changing a JVM flag, answer:
1. **What metric** is bad?
2. **By how much** does it need to improve?
3. **Why** do you believe a JVM flag will fix it?
4. **What** will you measure to confirm the fix?

If any answer is fuzzy, don't tune yet — measure.

---

## 16. Production Best Practices

### 16.1 Always-on observability

- **GC logs to file** with rotation. Persist them.
- **JFR continuous recording** rolling on disk; preserved on crash.
- **Heap dump on OOM** to a persistent volume.
- **Micrometer metrics** to Prometheus: heap, non-heap, GC pause counts, thread count.
- **Distributed tracing** for end-to-end latency attribution.

### 16.2 Alerting thresholds (starting points)

| Signal | Warn | Page |
|---|---|---|
| Heap used post-GC | > 70% of `Xmx` for 10 min | > 85% for 5 min |
| GC overhead | > 5% over 10 min | > 10% |
| Full GC count | > 0 in 1 hour | > 3 in 10 min |
| p99 GC pause | > 200 ms | > 1 s |
| Container memory | > 90% of limit | OOMKilled events |
| Thread count | > 2× steady state | 5× |
| Code cache | > 90% full | "CodeCache is full" log |

### 16.3 Safe tuning approach

1. **Establish a baseline.** Metrics + JFR for a representative workload.
2. **Form a hypothesis.** "This pause time is caused by Y, fixable by tuning X."
3. **Change one thing.** Multi-flag changes hide which knob did what.
4. **Measure under the same load.** Compare to baseline; same workload, same hardware.
5. **Roll out gradually.** Canary → 10% → 50% → 100%. Watch dashboards.
6. **Document.** Why the flag exists, who added it, what it fixed. Future-you will thank you.

### 16.4 Rollback discipline

- Keep the previous flag set in version control.
- Tag deploys with their JVM args.
- On regression, **roll back, then investigate** — don't try to fix forward under fire.

### 16.5 Checklist for every Java service in production

- [ ] `-Xms = -Xmx`, sized from live-set evidence.
- [ ] Container memory ≥ heap × ~1.4 (room for non-heap).
- [ ] `-XX:MaxDirectMemorySize`, `-XX:MaxMetaspaceSize`, `-XX:ReservedCodeCacheSize` set.
- [ ] GC chosen and justified (G1 or generational ZGC).
- [ ] GC logs enabled with rotation.
- [ ] HeapDumpOnOutOfMemoryError + path on persistent volume.
- [ ] ExitOnOutOfMemoryError so orchestrator restarts cleanly.
- [ ] JFR continuous recording or on-demand profile recipe documented.
- [ ] Micrometer JVM metrics → Prometheus → dashboards.
- [ ] Alerts on heap, GC overhead, Full GC count, pause time.
- [ ] Runbook: how to take a heap dump, thread dump, JFR.
- [ ] Postmortem template for memory incidents.

---

## 17. Final Checklist

**You are a senior-level JVM performance engineer if you can:**

- Walk a junior engineer through the path of a `.java` file from compilation to native code, naming each subsystem (class loader, runtime data areas, interpreter, tiered JIT) and what it produces.
- Draw the heap layout for a generational GC, label Eden / Survivors / Old, and explain object allocation flow including TLABs, GC triggers, and promotion paths.
- Explain the generational hypothesis, why young/old separation works, and which collectors honor it (and which don't, and which now do — like generational ZGC in Java 21).
- Describe how G1 splits the heap into regions, what mixed GCs are, why "Garbage First" is the name, and which knobs *actually* matter for it.
- Compare Serial, Parallel, G1, ZGC, and Shenandoah on pause time, throughput, footprint, and operational complexity — and pick the right one for a given service.
- Recognize each `OutOfMemoryError` variant (heap, metaspace, GC overhead, direct buffer, native thread) and outline its diagnostic path.
- Take a heap dump, open it in MAT, follow the dominator tree to a leak suspect, and produce a fix.
- Take a JFR recording, find top allocators, identify lock contention, and correlate GC events with latency outliers.
- Read a unified GC log line and extract pause time, before/after heap, and trigger reason — and tell from a few minutes of log whether the service is healthy.
- Distinguish a memory leak from high allocation pressure, and produce different remediation plans for each.
- Configure JVM flags appropriately for a Spring Boot service in Kubernetes — including container-aware sizing, native memory caps, GC selection, GC logging, and heap-dump-on-OOM.
- Diagnose OOMKilled (container) vs `OutOfMemoryError` (Java) without confusing them, and identify which non-heap region overflowed.
- Walk through the five real-world scenarios (high GC pauses, memory leak, CPU spike, deadlock, slow responses) without notes, including the tools and the likely root causes.
- Refuse to tune the JVM when the bottleneck is in the database, network, algorithm, or architecture — and explain why.
- Defend default G1 with proper heap sizing as the right starting point for almost any backend service, and articulate the conditions under which you'd switch to generational ZGC.
- Establish always-on observability (GC logs, JFR, metrics, alerts, runbooks) before tuning, and follow a one-change-at-a-time tuning loop with rollback discipline.

If you can do all of the above without looking anything up, you can confidently own JVM performance for production systems at a senior level.
