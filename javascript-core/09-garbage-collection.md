# 09 — Garbage Collection

> **"The garbage collector is your silent partner. It cleans up after you — but only if you stop holding on to things you no longer need. The moment you understand how it decides what to keep, you start writing code that works with it, not against it."**

Garbage collection is automatic in JavaScript, but it is not free and it is not magic. Every long-lived SPA eventually confronts GC-related problems: pauses, memory pressure, degraded performance. This document goes deeper than `08-memory-management.md` — covering the specific algorithms V8 uses, the tuning knobs available, how GC interacts with the rendering pipeline, and production strategies for GC-friendly code.

---

## 📚 Table of Contents

1. [Why GC Matters for Frontend Engineers](#1-why-gc-matters-for-frontend-engineers)
2. [V8 Heap Architecture in Detail](#2-v8-heap-architecture-in-detail)
3. [Scavenge — Young Generation GC](#3-scavenge--young-generation-gc)
4. [Major GC — Old Generation Collection](#4-major-gc--old-generation-collection)
5. [Incremental Marking](#5-incremental-marking)
6. [Concurrent and Parallel GC](#6-concurrent-and-parallel-gc)
7. [Write Barriers](#7-write-barriers)
8. [Object Promotion Rules](#8-object-promotion-rules)
9. [GC and the Rendering Pipeline](#9-gc-and-the-rendering-pipeline)
10. [Allocation Rate — The Hidden Enemy](#10-allocation-rate--the-hidden-enemy)
11. [Object Pooling — Reducing GC Pressure](#11-object-pooling--reducing-gc-pressure)
12. [Hidden Classes and GC Efficiency](#12-hidden-classes-and-gc-efficiency)
13. [Large Object Space](#13-large-object-space)
14. [GC in Node.js vs Browser](#14-gc-in-nodejs-vs-browser)
15. [Profiling GC Activity](#15-profiling-gc-activity)
16. [GC-Friendly Coding Patterns](#16-gc-friendly-coding-patterns)
17. [Good Practices](#17-good-practices)
18. [Bad Practices](#18-bad-practices)
19. [Interview-Level Explanation](#19-interview-level-explanation)
20. [Exercises](#20-exercises)

---

## 1. Why GC Matters for Frontend Engineers

Most frontend developers treat GC as invisible infrastructure. It works until it doesn't — and when it doesn't, the symptoms are subtle and hard to diagnose:

- **Jank**: frames that take 60–200ms instead of 16ms
- **Progressive slowdown**: app gets slower the longer it runs
- **Memory warnings**: browser tab killed after hours of use
- **Animation stuttering**: smooth animation broken by periodic freezes

All of these can be caused by GC pressure — either too many objects being allocated (triggering frequent collections) or too many objects staying alive (requiring expensive major collections).

```
GC impact on frame timing:

Ideal 60fps:
│▌│▌│▌│▌│▌│▌│▌│▌│▌│▌│  (each ▌ = 16ms frame)

Minor GC (Scavenge, ~1ms):
│▌│▌│▌█│▌│▌│▌█│▌│▌│▌│  (barely noticeable — quick recovery)

Major GC (Mark-Sweep, ~50ms):
│▌│▌│▌│▌│████████│▌│▌│  (3 frames dropped — visible jank)

Major GC (large heap, ~200ms):
│▌│▌│████████████████████████│▌│  (12+ frames — severe freeze)
```

The engineer's job: keep the heap small, reduce allocation rate, and architect code so major GCs are rare.

---

## 2. V8 Heap Architecture in Detail

V8 divides its heap into several regions, each with different collection strategies:

```
V8 HEAP (full layout):

┌──────────────────────────────────────────────────────────────────┐
│                         YOUNG GENERATION                          │
│                                                                    │
│  ┌─────────────────────────────┐                                  │
│  │  Semi-Space (From)          │  Active allocation space         │
│  │  Size: 1–8MB                │  New objects allocated here      │
│  └─────────────────────────────┘                                  │
│  ┌─────────────────────────────┐                                  │
│  │  Semi-Space (To)            │  Survivors copied here           │
│  │  Size: 1–8MB                │  during Scavenge                 │
│  └─────────────────────────────┘                                  │
│  Total young gen: 2–16MB                                          │
└────────────────────────────┬─────────────────────────────────────┘
                              │ objects surviving 2+ Scavenges
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                         OLD GENERATION                            │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Old Space                                                  │  │
│  │  General objects promoted from young gen                   │  │
│  │  Size: up to ~1.5GB (32-bit) / unlimited (64-bit)          │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Code Space                                                 │  │
│  │  JIT-compiled machine code for JS functions                │  │
│  │  Executable memory — special permissions required          │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Map Space (Shape Space)                                    │  │
│  │  Hidden classes (object shapes/maps)                       │  │
│  │  Describe the structure of objects                         │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Large Object Space                                         │  │
│  │  Objects > 256KB — never moved (too expensive)             │  │
│  │  Collected by mark-sweep only (no compaction)              │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### Key Sizes (V8 defaults, Chrome)

| Space                              | Default Size                          | Configurable?          |
| ---------------------------------- | ------------------------------------- | ---------------------- |
| Young generation (each semi-space) | 1–8MB                                 | Via flags              |
| Old space                          | Up to ~1.5GB (32-bit) / 4GB+ (64-bit) | Via flags              |
| Total heap limit                   | ~1.5GB (32-bit) / 4GB+ (64-bit)       | `--max-old-space-size` |

---

## 3. Scavenge — Young Generation GC

The Scavenge algorithm is a **copying collector** that runs on the young generation. It is designed to be extremely fast (~1ms) at the cost of using double the memory (two semi-spaces).

### The Algorithm

```
BEFORE SCAVENGE:
From-Space (active):
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ A  │ B  │ C  │ D  │ E  │ F  │    │    │
│LIVE│DEAD│LIVE│DEAD│LIVE│DEAD│free│free│
└────┴────┴────┴────┴────┴────┴────┴────┘

To-Space (empty):
┌────┬────┬────┬────┬────┬────┬────┬────┐
│    │    │    │    │    │    │    │    │
│free│free│free│free│free│free│free│free│
└────┴────┴────┴────┴────┴────┴────┴────┘

SCAVENGE RUNS:
1. Traverse from roots, find live objects (A, C, E)
2. COPY live objects to To-Space (compacted)
3. Update all references to point to new locations
4. Swap From/To labels

AFTER SCAVENGE:
Old From-Space (now entirely free):
┌────┬────┬────┬────┬────┬────┬────┬────┐
│    │    │    │    │    │    │    │    │
│free│free│free│free│free│free│free│free│
└────┴────┴────┴────┴────┴────┴────┴────┘

New To-Space (now the active From-Space):
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ A' │ C' │ E' │    │    │    │    │    │
│LIVE│LIVE│LIVE│free│free│free│free│free│
└────┴────┴────┴────┴────┴────┴────┴────┘
(A', C', E' are copies of A, C, E at new locations)
```

### Why Scavenge Is Fast

- **No fragmentation**: live objects always packed at the start of To-Space
- **Allocation is O(1)**: just increment a pointer (bump allocation)
- **Only processes live objects**: dead objects are simply abandoned (not swept)
- **Small working set**: young generation is tiny (≤16MB total)
- **Locality**: newly allocated objects often reference each other — copying them together improves cache performance

### Object Aging in the Young Generation

Objects have an **age counter** that increments each time they survive a Scavenge without being promoted:

```
Object lifecycle in young gen:
  Allocated → Scavenge 1 (survived, age=1) → Scavenge 2 (survived, age=2) → PROMOTED to old gen

  Most objects die young (age=0):
  → never survive their first Scavenge
  → collected for free (just abandoned, never processed)
```

---

## 4. Major GC — Old Generation Collection

When the old generation fills up (or approaches its limit), V8 triggers a **Major GC**. This uses the **Mark-Sweep-Compact** algorithm.

### Phase 1 — Marking

```
MARKING ALGORITHM (Tri-color marking):

Colors:
  WHITE: not yet visited (potentially garbage)
  GREY:  discovered but children not yet processed
  BLACK: fully processed (live, children also marked)

Algorithm:
  1. Mark all roots GREY, add to worklist
  2. While worklist not empty:
     a. Take a GREY object
     b. Mark all its WHITE references GREY, add to worklist
     c. Mark the object itself BLACK
  3. When worklist empty:
     WHITE objects = garbage (never reached)
     BLACK objects = live (all reachable objects)
```

### Phase 2 — Sweeping

```
SWEEPING:
  Walk the entire old space
  For each object:
    BLACK → clear color mark (reset to white for next GC)
    WHITE → reclaim memory (add to free list or return to OS)

Free list: a linked list of available memory chunks
Next allocation: find a chunk of suitable size from free list
```

### Phase 3 — Compaction (Optional)

Sweeping leaves gaps (fragmentation). Compaction moves live objects together:

```
BEFORE COMPACT:
┌────┬─────┬────┬─────┬────┬─────┬────┐
│LIVE│ GAP │LIVE│ GAP │LIVE│ GAP │LIVE│
└────┴─────┴────┴─────┴────┴─────┴────┘

AFTER COMPACT:
┌────┬────┬────┬────┬──────────────────┐
│LIVE│LIVE│LIVE│LIVE│   large free gap  │
└────┴────┴────┴────┴──────────────────┘
```

Compaction is expensive — it moves objects and must update ALL references to them. V8 only compacts when fragmentation is high enough to justify the cost.

### Cost of Major GC

```
Time complexity:
  Mark:    O(live objects in old gen)
  Sweep:   O(size of old gen)
  Compact: O(live objects) × O(references to each)

Real-world timing (rough estimates):
  Old gen 10MB:   ~5–15ms
  Old gen 100MB:  ~50–150ms
  Old gen 500MB:  ~200–500ms
  Old gen 1GB+:   ~500ms–2s (severe page freeze)
```

---

## 5. Incremental Marking

To avoid long stop-the-world pauses during the mark phase, V8 uses **incremental marking**: split the mark phase into small increments, interleaved with JavaScript execution.

```
Without incremental marking:
JS ████████████████  GC MARK (100ms pause) ████████████████  JS
                     ↑ user sees frozen page here

With incremental marking:
JS █ GC█ JS █ GC█ JS █ GC█ JS █ GC█ JS █ GC█  JS
   (each GC increment ~5ms — barely noticeable)
```

### The Challenge: The Mutator Problem

While GC marks incrementally, JavaScript (the "mutator") continues to modify the object graph. New references may be created between already-marked objects:

```
Incremental marking in progress:
  Object A: BLACK (already fully marked)
  Object B: WHITE (not yet reached)

JavaScript runs and does:
  A.ref = B; // A now points to B

GC continues marking...
  A is BLACK — children already processed — won't revisit B
  B remains WHITE — looks like garbage!
  B would incorrectly be collected!
```

This is solved by **write barriers**.

---

## 6. Concurrent and Parallel GC

V8 further reduces pause times by running GC work on background threads:

### Concurrent Marking

```
Main thread:    JS ████████████████████████████████████ JS
Background:        GC MARK ████████████████████████
                   (marks concurrently with JS execution)

Stop-the-world: ← tiny pause only to finalize marking →
```

Main thread only pauses briefly at the end to process objects modified during concurrent marking (handled via write barriers).

### Parallel GC

Multiple background threads work on GC simultaneously:

```
Main thread:    ← short stop-the-world pause →
Background 1:   GC MARK ████████
Background 2:   GC MARK ████████
Background 3:   GC MARK ████████
Background 4:   GC MARK ████████
                (4 threads = ~4x faster marking)
```

### Current V8 GC Pipeline

```
Modern V8 GC (simplified):

1. Concurrent marking starts (background, no pause)
2. JavaScript continues normally
3. Write barriers track mutations during step 1 & 2
4. Short pause: finalize marking, process write barrier logs
5. Concurrent sweeping starts (background)
6. Concurrent compaction (if needed, background)
7. JavaScript resumes — sweeping/compaction finish in background
```

The main thread only pauses for steps 4 — typically 1–5ms even for large heaps.

---

## 7. Write Barriers

A **write barrier** is code automatically inserted by V8 around object property assignments. It notifies the GC about reference changes so incremental/concurrent marking stays consistent.

```javascript
// Your code:
object.property = value;

// What V8 actually executes:
object.property = value;
writeBarrier(object, value); // automatically inserted by JIT compiler
```

### Types of Write Barriers in V8

**Generational Write Barrier (for Scavenge):**

```
When: an old-generation object's property is set to point to a young-gen object
Why:  Scavenge only scans young gen + "remembered set" of old→young pointers
Action: add the old-gen object to the "remembered set"

Without this: old-gen object pointing to young-gen object — Scavenge might
collect the young-gen object (not found from roots) even though it's alive
```

**Tri-color Write Barrier (for incremental/concurrent marking):**

```
When: a BLACK object's property is set to point to a WHITE object
Why:  BLACK objects won't be revisited — WHITE child would be incorrectly collected
Action: either re-grey the BLACK object, or mark the WHITE object GREY

This ensures no live objects are incorrectly collected during incremental/concurrent marking
```

Write barriers have a small runtime cost — every property assignment has overhead. This is why extremely hot property writes in tight loops can be slower than you expect in benchmarks.

---

## 8. Object Promotion Rules

Not all objects in the young generation get promoted. V8 uses two criteria:

### Criteria 1 — Survival Count

An object is promoted to old gen after surviving a configurable number of Scavenges (default: 2). This means objects that live for more than 2 GC cycles are considered "long-lived" and moved to more expensive but less frequent collection.

### Criteria 2 — Size

Objects larger than a threshold (~1MB) are allocated directly in old gen (or Large Object Space). Allocating them in young gen would immediately fill it and trigger constant Scavenges.

### Promotion Overhead

```javascript
// ❌ Allocation pattern that causes immediate promotion stress
function processRequests(requests) {
  requests.forEach((req) => {
    // Large result object — may be promoted to old gen immediately
    const result = {
      data: new Array(50_000), // large enough to skip young gen
      metadata: buildMetadata(req),
      timestamp: Date.now(),
    };
    sendResult(result);
    // result goes out of scope — but it's already in old gen
    // stays until next major GC
  });
}
```

```javascript
// ✅ Process in-place, avoid large intermediate objects
function processRequests(requests) {
  // Preallocate once, reuse across requests
  const buffer = new Float64Array(50_000);

  requests.forEach((req) => {
    fillBuffer(buffer, req); // fill preallocated buffer
    sendBuffer(buffer, req); // no new large allocation
  });
}
```

---

## 9. GC and the Rendering Pipeline

GC pauses can cause dropped frames. Understanding where in the rendering pipeline GC can interrupt helps you mitigate its impact.

```
Frame rendering timeline:

Input events
     │
     ▼
JavaScript (rAF callbacks + event handlers)
     │  ← GC PAUSE CAN HAPPEN HERE (during JS execution)
     ▼
Style recalculation
     │
     ▼
Layout
     │
     ▼
Paint
     │
     ▼
Composite / Present
     │
     ▼
─── 16.67ms budget ───
```

### When GC Runs

- **Scavenge**: triggered when the young generation fills up (~every few MB allocated)
- **Major GC**: triggered when old generation reaches its limit
- **Both**: can interrupt JavaScript execution mid-frame

### The Idle GC Strategy

V8 tries to run GC during idle time — when the browser is between frames and not executing JavaScript:

```
Frame 1  Frame 2  [IDLE TIME]  Frame 3  Frame 4
│▌       │▌       ├── GC ──┤   │▌       │▌
                  ↑
           GC runs here during idle time
           No frames dropped!
```

V8's idle GC is triggered via `requestIdleCallback`-like mechanisms internally. But it only works if your app creates enough idle time — which requires not holding the main thread busy continuously.

### Practical Implication

```javascript
// ❌ Continuous main thread work — no idle time for GC
function animateForever() {
  heavyCalculation(); // keeps main thread busy
  requestAnimationFrame(animateForever);
}

// The GC never gets idle time — it's forced to interrupt JS mid-frame
// Result: periodic janky frames when GC can't wait any longer

// ✅ Yield idle time — allows GC to run without interrupting frames
function animateForever() {
  lightCalculation(); // only what's needed for this frame
  requestAnimationFrame(animateForever);
  // Between frames: idle time available for GC
}
```

---

## 10. Allocation Rate — The Hidden Enemy

**Allocation rate** is how much memory your code allocates per second. It's a more impactful metric than heap size for GC pressure, because:

- High allocation rate → Scavenge triggers frequently
- Frequent Scavenge → more objects promoted to old gen
- More old gen objects → more frequent and expensive major GC
- Major GC → long pauses, dropped frames

```
The cascade:

High alloc rate
  → young gen fills up quickly
  → frequent Scavenges (every few ms)
  → many objects survive Scavenges (get promoted)
  → old gen fills up quickly
  → frequent major GCs
  → long pauses
  → dropped frames / jank
```

### Measuring Allocation Rate

```javascript
// Simple allocation rate monitor
class AllocationRateMonitor {
  constructor() {
    this._lastHeap = 0;
    this._lastTime = 0;
    this._rafId = null;
  }

  start() {
    const measure = (now) => {
      const heap = performance.memory?.usedJSHeapSize ?? 0;
      const dt = now - this._lastTime;

      if (this._lastTime > 0 && dt > 0) {
        const heapDelta = heap - this._lastHeap;
        // Only count increases (GC runs cause negative delta)
        if (heapDelta > 0) {
          const rateMBps = ((heapDelta / dt) * 1000) / 1024 / 1024;
          if (rateMBps > 10) {
            // > 10 MB/s is high
            console.warn(`High alloc rate: ${rateMBps.toFixed(1)} MB/s`);
          }
        }
      }

      this._lastHeap = heap;
      this._lastTime = now;
      this._rafId = requestAnimationFrame(measure);
    };

    this._rafId = requestAnimationFrame(measure);
  }

  stop() {
    if (this._rafId) cancelAnimationFrame(this._rafId);
  }
}
```

### Common High-Allocation Patterns

```javascript
// ❌ Allocates new array every frame in animation loop
function animate() {
  const positions = elements.map((el) => ({
    // new array + N objects per frame
    x: el.x + el.vx,
    y: el.y + el.vy,
  }));
  render(positions);
  requestAnimationFrame(animate);
}
// At 60fps: 60 × N new objects per second — very high allocation rate

// ✅ Mutate in-place — zero new allocations per frame
function animate() {
  elements.forEach((el) => {
    // no new array, no new objects
    el.x += el.vx;
    el.y += el.vy;
  });
  render(elements);
  requestAnimationFrame(animate);
}
```

---

## 11. Object Pooling — Reducing GC Pressure

An object pool pre-allocates objects and **reuses** them instead of creating and discarding them. This reduces allocation rate and therefore GC pressure.

### When to Use Object Pools

- Objects created and destroyed at high frequency (animation particles, game entities)
- Objects with predictable structure (same shape, same properties)
- Hot paths where allocation cost is measurable
- Real-time systems where GC pauses are unacceptable

### Generic Object Pool Implementation

```javascript
class ObjectPool {
  constructor(factory, initialSize = 10) {
    this._factory = factory;
    this._pool = [];
    this._active = new Set();

    // Pre-populate pool
    for (let i = 0; i < initialSize; i++) {
      this._pool.push(this._factory());
    }
  }

  /**
   * Acquire an object from the pool (or create new if empty)
   */
  acquire() {
    const obj = this._pool.length > 0 ? this._pool.pop() : this._factory(); // pool exhausted — allocate new

    this._active.add(obj);
    return obj;
  }

  /**
   * Return an object to the pool for reuse
   */
  release(obj) {
    if (!this._active.has(obj)) return; // not from this pool

    this._active.delete(obj);
    this._reset(obj); // clear state before reuse
    this._pool.push(obj);
  }

  /**
   * Release all active objects back to pool
   */
  releaseAll() {
    this._active.forEach((obj) => {
      this._reset(obj);
      this._pool.push(obj);
    });
    this._active.clear();
  }

  /**
   * Override to reset object state on release
   */
  _reset(obj) {
    // Clear all properties to default state
    // Specific to object type — override in subclass
  }

  get activeCount() {
    return this._active.size;
  }
  get pooledCount() {
    return this._pool.length;
  }
}
```

### Particle System with Object Pool

```javascript
// Particle pool — zero GC pressure in animation loop
class ParticlePool extends ObjectPool {
  constructor(size = 1000) {
    super(
      () => ({
        x: 0,
        y: 0,
        vx: 0,
        vy: 0,
        life: 0,
        maxLife: 0,
        r: 255,
        g: 255,
        b: 255,
        a: 1,
        size: 4,
        active: false,
      }),
      size,
    );
  }

  _reset(p) {
    p.x = p.y = 0;
    p.vx = p.vy = 0;
    p.life = p.maxLife = 0;
    p.r = p.g = p.b = 255;
    p.a = 1;
    p.size = 4;
    p.active = false;
  }

  emit(x, y, options = {}) {
    const p = this.acquire();
    p.x = x;
    p.y = y;
    p.vx = options.vx ?? (Math.random() - 0.5) * 4;
    p.vy = options.vy ?? (Math.random() - 0.5) * 4;
    p.maxLife = p.life = options.life ?? 60; // frames
    p.r = options.r ?? 255;
    p.g = options.g ?? 100;
    p.b = options.b ?? 0;
    p.active = true;
    return p;
  }
}

const pool = new ParticlePool(500);
const activeParticles = [];

function update() {
  // Update existing particles
  for (let i = activeParticles.length - 1; i >= 0; i--) {
    const p = activeParticles[i];
    p.x += p.vx;
    p.y += p.vy;
    p.vy += 0.1; // gravity
    p.a = p.life / p.maxLife;
    p.life--;

    if (p.life <= 0) {
      pool.release(p); // return to pool — no GC needed
      activeParticles.splice(i, 1);
    }
  }

  // Emit new particles on click
}

// Zero new allocations during animation — all objects reused from pool
```

### Vector Pool for Math Operations

```javascript
// Instead of creating { x, y } objects constantly:
class Vec2Pool extends ObjectPool {
  constructor() {
    super(() => ({ x: 0, y: 0 }), 100);
  }

  _reset(v) {
    v.x = 0;
    v.y = 0;
  }

  get(x, y) {
    const v = this.acquire();
    v.x = x;
    v.y = y;
    return v;
  }
}

const vecPool = new Vec2Pool();

function computeForces(entities) {
  for (const a of entities) {
    for (const b of entities) {
      if (a === b) continue;

      // ✅ Reuse vector from pool — no new allocation
      const diff = vecPool.get(b.x - a.x, b.y - a.y);
      const dist = Math.hypot(diff.x, diff.y);
      const force = 1 / (dist * dist);

      a.vx += diff.x * force;
      a.vy += diff.y * force;

      vecPool.release(diff); // immediately return — reuse in next iteration
    }
  }
}
```

---

## 12. Hidden Classes and GC Efficiency

V8 uses **hidden classes** (also called shapes or maps) to describe the structure of objects. Objects with the same shape share a hidden class and can be accessed with optimized machine code.

Hidden classes affect GC because:

1. The Map Space holds all hidden class objects
2. Each new hidden class is an allocation in Map Space
3. Polymorphic code creates many hidden classes → Map Space pressure

### Hidden Class Creation

```javascript
// Each of these creates a DIFFERENT hidden class
const a = { x: 1, y: 2 }; // shape: {x, y}
const b = { y: 2, x: 1 }; // shape: {y, x} — DIFFERENT! order matters
const c = { x: 1 }; // shape: {x}
const d = { x: 1, y: 2, z: 3 }; // shape: {x, y, z}
```

### Transition Chain

When you add properties to an object, V8 creates a new hidden class (transition):

```javascript
const obj = {}; // shape: {} (empty)
obj.x = 1; // shape: {} → {x}  (new hidden class)
obj.y = 2; // shape: {x} → {x,y} (new hidden class)
obj.z = 3; // shape: {x,y} → {x,y,z} (new hidden class)

// This creates 4 hidden classes and 3 transitions in Map Space
```

### GC-Friendly: Pre-define Shape in Constructor

```javascript
// ✅ All instances have the same hidden class from creation
function Point(x, y) {
  this.x = x; // shape defined once in constructor
  this.y = y; // always in same order
}

// All Point instances: shape {x, y}
// ONE hidden class for all 10,000 instances
const points = Array.from({ length: 10_000 }, (_, i) => new Point(i, i));
```

```javascript
// ❌ Different shapes — many hidden classes
function makePoint(includeZ) {
  const p = { x: 0, y: 0 };
  if (includeZ) p.z = 0; // some have {x,y,z}, some have {x,y}
  return p;
}
// Two hidden classes — code working with these is polymorphic (slower)
```

---

## 13. Large Object Space

Objects larger than 256KB are allocated directly in the **Large Object Space** (LOS). This has important characteristics:

```
Large Object Space properties:
  - Objects are NEVER moved (too expensive to copy)
  - Not compacted — fragmentation can accumulate
  - Collected by mark-sweep only (no copying)
  - Allocation of a large object can immediately trigger Major GC
    if old gen is near its limit
```

### What Goes Into Large Object Space

```javascript
// These go directly to Large Object Space:
const bigArray = new Array(100_000); // ~800KB — LOS
const bigBuffer = new ArrayBuffer(1_000_000); // ~1MB — LOS
const bigString = "x".repeat(500_000); // ~1MB — LOS
const bigTyped = new Float64Array(100_000); // ~800KB — LOS

// Compiled code can also go here (JIT-compiled hot functions)
```

### LOS Implications

```javascript
// ❌ Allocating large objects repeatedly triggers Major GC
function processChunks(data) {
  while (data.length > 0) {
    const chunk = data.splice(0, 100_000); // 100k items → LOS allocation
    process(chunk);
    // chunk becomes garbage → LOS needs sweeping → Major GC pressure
  }
}

// ✅ Reuse a single large buffer
function processChunks(data) {
  const buffer = new Float64Array(100_000); // allocate ONCE in LOS
  let offset = 0;

  while (offset < data.length) {
    const end = Math.min(offset + buffer.length, data.length);
    buffer.set(data.slice(offset, end));
    processBuffer(buffer, end - offset);
    offset = end;
  }
  // buffer reused — no repeated LOS allocations
}
```

---

## 14. GC in Node.js vs Browser

While both use V8, Node.js and browser environments have different GC considerations:

### Memory Limits

```javascript
// Default V8 heap limits:
// 32-bit: ~1.5GB total heap
// 64-bit: up to OS limits (but default old space is ~1.5GB without flags)

// Node.js — increase old space for memory-intensive applications:
// node --max-old-space-size=4096 app.js  (4GB)

// Check current limits in Node.js:
const v8 = require("v8");
console.log(v8.getHeapStatistics());
// {
//   total_heap_size: ...,
//   total_heap_size_executable: ...,
//   used_heap_size: ...,
//   heap_size_limit: ...,     ← the important one
//   ...
// }
```

### Node.js GC Control

```javascript
// Node.js exposes more GC control than browsers
const v8 = require("v8");

// Get heap statistics
v8.getHeapStatistics();

// Get heap space statistics (per-space breakdown)
v8.getHeapSpaceStatistics();
// Returns array of: { space_name, space_size, space_used_size, ... }

// Force GC (requires --expose-gc flag)
// node --expose-gc app.js
if (global.gc) {
  global.gc(); // force full GC cycle — useful in tests, not production
}
```

### Browser vs Node.js GC Behavior

| Aspect          | Browser                           | Node.js                             |
| --------------- | --------------------------------- | ----------------------------------- |
| GC idle time    | Between render frames             | Between event loop ticks            |
| Memory limit    | ~4GB on 64-bit (browser enforced) | Configurable via flags              |
| GC forcing      | Not exposed to JS                 | `global.gc()` with `--expose-gc`    |
| Heap stats      | `performance.memory` (limited)    | `v8.getHeapStatistics()` (detailed) |
| Background tabs | Throttled (less GC idle time)     | Not applicable                      |
| Memory pressure | Browser can kill tab              | Process can be OOM-killed           |

---

## 15. Profiling GC Activity

### Chrome DevTools — Performance Tab

```
Identifying GC in Performance timeline:

1. Open DevTools → Performance
2. Click Record
3. Use the app for 30 seconds
4. Stop
5. Look for:
   - "Minor GC" events (yellow): Scavenge runs — should be quick (~1ms)
   - "Major GC" events (yellow): Mark-Sweep — should be rare
   - Long "GC" tasks with red warning triangles
   - Sawtooth pattern in memory graph (healthy)
   - Climbing memory graph (potential leak)

In the Memory panel at the bottom:
   - Blue line: JS heap (should oscillate, not climb)
   - Green line: DOM nodes (should stay stable or decrease)
   - Yellow line: Event listeners (should stay stable)
```

### Memory Timeline Patterns

```
HEALTHY (sawtooth):
MB │    ╭╮  ╭╮  ╭╮  ╭╮  ╭╮
   │   ╭╯╰╮╭╯╰╮╭╯╰╮╭╯╰╮╭╯╰╮
   │───╯   ╰╯   ╰╯   ╰╯   ╰╯
   └────────────────────────▶ time
   (GC runs regularly, heap returns to baseline)

HIGH ALLOCATION RATE (rapid sawtooth):
MB │╭╮╭╮╭╮╭╮╭╮╭╮╭╮╭╮╭╮╭╮╭╮
   ││╰││╰││╰││╰││╰││╰││╰││╰│
   └────────────────────────▶ time
   (GC running very frequently — reduce allocation rate)

MEMORY LEAK (climbing baseline):
MB │                    ╭───
   │               ╭───╯
   │          ╭───╯
   │     ╭───╯
   │────╯
   └────────────────────────▶ time
   (baseline rises after each GC — objects accumulating)
```

### V8 GC Tracing (Node.js)

```bash
# Verbose GC logging in Node.js
node --trace-gc app.js

# Output example:
[45327:0x...] 1234 ms: Scavenge 12.3 (14.1) -> 8.7 (14.1) MB
#               │          │     │       │         │     │
#               │          │     │       │         └──── after heap
#               │          │     │       └────────────── before heap
#               │          │     └────────────────────── after used
#               │          └──────────────────────────── before used
#               └─────────────────────────────────────── timestamp
```

---

## 16. GC-Friendly Coding Patterns

### Pattern 1 — Avoid Object Allocation in Hot Loops

```javascript
// ❌ 60fps × N new objects = massive allocation rate
function renderParticles(particles) {
  particles.forEach((p) => {
    const screenPos = { x: p.x * scale, y: p.y * scale }; // new object per particle per frame
    drawAt(screenPos.x, screenPos.y);
  });
}

// ✅ No allocation — use parameters directly
function renderParticles(particles, scale) {
  particles.forEach((p) => {
    drawAt(p.x * scale, p.y * scale); // no allocation
  });
}
```

### Pattern 2 — Reuse ArrayBuffers for Data Processing

```javascript
// ✅ Process streaming data with a fixed buffer
class StreamProcessor {
  constructor(bufferSize = 65536) {
    this._buffer = new Uint8Array(bufferSize); // allocated once
    this._view = new DataView(this._buffer.buffer);
  }

  process(chunk) {
    // copy chunk into existing buffer
    this._buffer.set(chunk);
    return this._parseBuffer(chunk.byteLength);
  }

  _parseBuffer(length) {
    // read from this._view — no new allocations
    const result = [];
    for (let offset = 0; offset < length; offset += 4) {
      result.push(this._view.getFloat32(offset, true));
    }
    return result;
  }
}
```

### Pattern 3 — String Interning for Repeated Strings

```javascript
// ❌ Many identical string objects — waste of heap
function processEvents(events) {
  return events.map((e) => ({
    type: e.type, // 'click' string created per event
    category: "user-input", // 'user-input' string created per event
    // ...
  }));
}

// ✅ Intern strings — reuse same reference
const EVENT_CATEGORIES = Object.freeze({
  USER_INPUT: "user-input",
  SYSTEM: "system",
  NETWORK: "network",
});

function processEvents(events) {
  return events.map((e) => ({
    type: e.type,
    category: EVENT_CATEGORIES.USER_INPUT, // same string reference every time
  }));
}
```

### Pattern 4 — Batch DOM Work to Reduce GC Pressure from Layout

```javascript
// ❌ Read-write-read-write: forces layout AND allocates GC'd intermediate results
elements.forEach((el) => {
  const rect = el.getBoundingClientRect(); // alloc rect object
  el.style.transform = `translate(${rect.left}px, ${rect.top}px)`;
  // rect becomes garbage immediately
});

// ✅ Read once, write once — fewer allocations AND no layout thrashing
const rects = elements.map((el) => el.getBoundingClientRect()); // N rect objects
elements.forEach((el, i) => {
  // Template literal still allocates, but only N strings (not 2N)
  el.style.transform = `translate(${rects[i].left}px, ${rects[i].top}px)`;
});
```

---

## 17. Good Practices

### ✅ Minimize object allocations in animation loops

```javascript
// ✅ Pre-compute everything outside the loop
const TWO_PI = Math.PI * 2; // constant — not recomputed per frame
const centerX = canvas.width / 2;
const centerY = canvas.height / 2;

function drawCircles(ctx, circles) {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  for (const c of circles) {
    ctx.beginPath();
    ctx.arc(c.x, c.y, c.r, 0, TWO_PI); // no allocation
    ctx.fill();
  }
}
```

### ✅ Use TypedArrays for numeric data

```javascript
// ✅ Exact memory, no GC overhead, cache-friendly
const positions = new Float32Array(entityCount * 2); // [x0, y0, x1, y1, ...]
const velocities = new Float32Array(entityCount * 2);
```

### ✅ Pre-allocate result arrays when size is known

```javascript
// ❌ Array grows dynamically — multiple reallocations
function transform(data) {
  const result = [];
  for (const item of data) result.push(process(item)); // may reallocate
  return result;
}

// ✅ Pre-allocate to exact size
function transform(data) {
  const result = new Array(data.length); // exact size — one allocation
  for (let i = 0; i < data.length; i++) result[i] = process(data[i]);
  return result;
}
```

### ✅ Use `structuredClone` for deep copies (modern, optimized)

```javascript
// ✅ Native API — more efficient than JSON.parse(JSON.stringify(...))
const clone = structuredClone(original);
// - Handles circular references
// - Handles Date, Map, Set, TypedArray, etc.
// - Implemented in native code (faster than JS-based deep clone)
```

---

## 18. Bad Practices

### ❌ Creating objects as function return values in hot loops

```javascript
// ❌ Every call creates a new { x, y } object
function getPosition(entity) {
  return { x: entity.x, y: entity.y }; // new object per call
}

for (const e of entities) {
  const pos = getPosition(e); // alloc × entities.length per frame
  render(pos.x, pos.y);
}
```

### ❌ Spreading objects unnecessarily

```javascript
// ❌ `{ ...defaults, ...options }` creates a new object every call
function createConfig(options) {
  return { ...defaults, ...options }; // if called per frame — massive alloc
}

// ✅ For hot paths: use Object.assign on a pooled object, or pass defaults explicitly
```

### ❌ Closures in rAF callbacks that create new functions per frame

```javascript
// ❌ New function created every rAF callback
function animate() {
  const handler = (e) => processEvent(e); // new closure per frame
  element.addEventListener("click", handler); // and adds another listener!
  requestAnimationFrame(animate);
}

// ✅ Define handler once, outside the loop
const handler = (e) => processEvent(e);
element.addEventListener("click", handler);

function animate() {
  requestAnimationFrame(animate);
}
```

### ❌ Concatenating strings in loops

```javascript
// ❌ Creates intermediate strings on each concatenation
let result = "";
for (let i = 0; i < 10000; i++) {
  result += items[i].toString(); // O(n²) allocations
}

// ✅ Collect and join — O(n) allocations
const parts = new Array(items.length);
for (let i = 0; i < items.length; i++) parts[i] = items[i].toString();
const result = parts.join("");
```

---

## 19. Interview-Level Explanation

> **"How does V8's garbage collector work? How do you optimize for it?"**

**Strong answer:**

> "V8 uses a generational garbage collector based on the observation that most objects die young. Memory is divided into young and old generations.
>
> The young generation uses a copying collector called Scavenge. It has two semi-spaces — only one is active at a time. When the active space fills up, V8 copies all live objects to the other space, compacting them, then declares the old space entirely free. This is very fast, typically under 1ms, because only live objects are processed. Objects that survive two Scavenges are promoted to the old generation.
>
> The old generation uses a Mark-Sweep-Compact algorithm. The mark phase traverses the entire object graph from roots, marking reachable objects. The sweep phase reclaims unmarked objects. Compaction, which is optional and expensive, moves live objects together to reduce fragmentation. Major GC can pause execution for 10–500ms depending on heap size.
>
> To reduce pause times, V8 uses incremental marking — splitting the mark phase into small increments interleaved with JS execution. Write barriers are automatically inserted around property assignments to keep the GC's view of the object graph consistent during incremental marking. V8 also does concurrent and parallel marking on background threads, so the main thread only pauses briefly for finalization.
>
> For optimization: the biggest lever is reducing allocation rate. Each allocation creates GC pressure — high allocation rate in animation loops leads to frequent Scavenges, object promotions, and eventually frequent major GCs. Techniques include object pooling for frequently created/destroyed objects, TypedArrays for numeric data (no GC overhead, cache-friendly), pre-allocating arrays when the size is known, and avoiding object creation inside hot loops. The goal is to create fewer, longer-lived objects rather than many short-lived ones — working with the generational hypothesis rather than against it."

---

## 20. Exercises

### Exercise 1 — Identify GC pressure sources

Rate each code snippet (Low / Medium / High GC pressure) and explain why:

```javascript
// a)
function a(items) {
  return items.map((x) => x * 2);
}

// b)
function b(items) {
  const result = new Array(items.length);
  for (let i = 0; i < items.length; i++) result[i] = items[i] * 2;
  return result;
}

// c)
function c(items) {
  const result = new Float64Array(items.length);
  for (let i = 0; i < items.length; i++) result[i] = items[i] * 2;
  return result;
}

// d) — called 60fps in animation loop
function d() {
  return elements.map((el) => ({
    x: el.getBoundingClientRect().left,
    y: el.getBoundingClientRect().top,
  }));
}
```

<details>
<summary>Answer</summary>

```
a) Medium — .map() allocates a new array + N closure callbacks internally
   (though engines optimize this)

b) Medium — new Array(n) + N assignments — similar to .map() but slightly
   less overhead (no closure allocation per iteration)

c) Low — TypedArray: contiguous memory, no per-element heap objects,
   no GC overhead, cache-friendly. Best choice for numeric data.

d) HIGH — called every frame (60fps):
   - elements.map() allocates new array per frame
   - N new { x, y } objects per frame
   - 2N getBoundingClientRect() calls (triggers layout thrashing too)
   - All objects become garbage immediately after the frame
   At 60fps with 100 elements: 6000+ new objects/second

   Fix:
   const rects = elements.map(el => el.getBoundingClientRect()); // 1 read phase
   // store rects as class property, update via ResizeObserver / scroll events
   // only re-read when layout actually changes
```

</details>

---

### Exercise 2 — Build a fixed-size object pool

Implement a circular buffer object pool with a fixed maximum size. When the pool is full and a new object is needed, the oldest acquired object is forcibly recycled:

```javascript
class FixedPool {
  constructor(factory, maxSize) {
    /* ... */
  }
  acquire() {
    /* ... */
  }
  release(obj) {
    /* ... */
  }
}
```

<details>
<summary>Solution</summary>

```javascript
class FixedPool {
  constructor(factory, maxSize = 100) {
    this._factory = factory;
    this._maxSize = maxSize;
    this._free = [];
    this._acquired = [];

    // Pre-fill pool
    for (let i = 0; i < maxSize; i++) {
      this._free.push(factory());
    }
  }

  acquire() {
    let obj;

    if (this._free.length > 0) {
      obj = this._free.pop();
    } else if (this._acquired.length >= this._maxSize) {
      // Recycle the oldest acquired object (FIFO eviction)
      obj = this._acquired.shift();
      this._onRecycle?.(obj);
    } else {
      // Grow pool (shouldn't happen if maxSize pre-allocated)
      obj = this._factory();
    }

    this._acquired.push(obj);
    return obj;
  }

  release(obj) {
    const idx = this._acquired.indexOf(obj);
    if (idx === -1) return; // not from this pool

    this._acquired.splice(idx, 1);
    this._reset(obj);
    this._free.push(obj);
  }

  _reset(obj) {
    // Override or pass in constructor
    if (typeof obj.reset === "function") obj.reset();
  }

  onRecycle(fn) {
    this._onRecycle = fn;
    return this;
  }

  get stats() {
    return {
      free: this._free.length,
      acquired: this._acquired.length,
      total: this._free.length + this._acquired.length,
    };
  }
}

// Usage
const pool = new FixedPool(() => ({ x: 0, y: 0, active: false }), 50);
const p1 = pool.acquire();
p1.x = 10;
p1.y = 20;
pool.release(p1); // returns to pool
```

</details>

---

### Exercise 3 — Measure your app's allocation rate

Open the Chrome DevTools Memory tab, select "Allocation instrumentation on timeline", record for 10 seconds while interacting with any JavaScript-heavy web app (or your own project). Then answer:

1. What is the total amount allocated during the 10 seconds?
2. Which JavaScript functions allocated the most (visible in the "Allocation Stack" view)?
3. Are there blue bars (allocations) that are never turned grey (collected)? What objects are those?
4. What's one change you'd make to reduce the allocation rate?

---

## 🔗 Related Topics

- [`javascript-core/08-memory-management.md`](./08-memory-management.md) — Memory lifecycle and leak patterns
- [`performance/05-memory-leaks.md`](../performance/05-memory-leaks.md) — Finding and fixing leaks in production
- [`debugging/02-memory-tab.md`](../debugging/02-memory-tab.md) — DevTools Memory tab in depth
- [`performance/10-canvas-optimization.md`](../performance/10-canvas-optimization.md) — Object pooling for canvas

---

<div align="center">

**Next:** [`javascript-core/10-async-patterns.md`](./10-async-patterns.md) →

</div>
