# W-TinyLFU Visualizer

An interactive, single-file visualization of the **W-TinyLFU** caching algorithm — the admission policy
popularized by Java's [Caffeine](https://github.com/ben-manes/Caffeine) cache. Open `index.html` in any
browser and step through how a frequency-aware cache admits, promotes, and evicts entries in real time.

> No build step, no dependencies, no server. Just one self-contained `index.html`.

---

## Table of Contents

- [What is W-TinyLFU?](#what-is-w-tinylfu)
- [Quick Start](#quick-start)
- [Cache Architecture](#cache-architecture)
- [Controls](#controls)
- [Workloads](#workloads)
- [Visualization Panels](#visualization-panels)
- [Algorithm Details](#algorithm-details)
- [Design Notes & Caveats](#design-notes--caveats)

---

## What is W-TinyLFU?

W-TinyLFU combines three ideas into one high-hit-rate cache:

1. **A small LRU *Window*** that captures **recency** (recently seen keys).
2. **A *TinyLFU* admission filter** backed by a **Count-Min Sketch** that captures **frequency**
   (how often keys have been seen over time).
3. **An SLRU *Main*** space split into **Protected** (frequently re-accessed) and
   **Probationary** (newly admitted, unproven) segments.

New keys start in the Window. When the Window overflows, its least-recently-used entry becomes a
**candidate** that must *beat* the Main cache's **victim** on estimated frequency to be admitted.
This gives W-TinyLFU both **scan resistance** (one-time floods are rejected) and **high hit rates on
skewed (Zipf-like) workloads**.

---

## Quick Start

```bash
# from the project directory
open index.html        # macOS
# or
start index.html       # Windows
# or just double-click the file
```

Press **▶ Play** to auto-run, or **⏭ Step** to advance one access at a time and read the per-step
breakdown.

https://more-stars-great.github.io/W-TinyLFU-Viz/

---

## Cache Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                       Total Capacity                          │
│                                                               │
│  ┌──────────┐   ┌──────────────────────────────────────────┐ │
│  │  Window   │   │                  Main                     │ │
│  │ (recency) │   │  ┌────────────────┬─────────────────────┐ │ │
│  │  default  │   │  │   Protected    │    Probationary     │ │ │
│  │    5      │   │  │   80% of main  │    ~20% of main     │ │ │
│  │           │   │  │ (2nd-hit keys) │  (new admissions)   │ │ │
│  └──────────┘   │  └────────────────┴─────────────────────┘ │ │
│                 └──────────────────────────────────────────┘ │
│                                                               │
│  TinyLFU gate: candidate (Window LRU) vs Main victim —         │
│  admitted only when freq(candidate) > freq(victim)             │
└──────────────────────────────────────────────────────────────┘
```

### Data flow per access

```
access(key) → bump Count-Min Sketch for key
   │
   ├─ HIT in Window      → move to Window MRU
   ├─ HIT in Probationary→ promote to Protected (demote Protected LRU if full)
   ├─ HIT in Protected   → move to Protected MRU
   └─ MISS               → add to Window
                           └─ on Window overflow → candidate = Window LRU
                                ├─ Main has space  → admit to Probationary (no freq check)
                                └─ Main full       → freq(candidate) > freq(victim) ?
                                                     ├─ yes → admit candidate, evict victim
                                                     └─ no  → reject candidate (drop)
```

There is **no direct Window → Protected path**. Every key must graduate through the full pipeline:
`Window → Probationary → (re-hit) → Protected`. This two-stage confirmation is what keeps Protected
(80% of capacity) reserved for genuinely hot keys.

---

## Controls

| Control | Description |
|--------|-------------|
| **▶ Play / ⏸ Pause** | Auto-advance accesses. |
| **⏭ Step** | Advance exactly one access and pause — best for learning. |
| **↺ Reset** | Clear all state and the frequency sketch. |
| **Speed** | Playback speed (0.25× – 8×). |
| **负载 / Workload** | Access pattern generator (see [Workloads](#workloads)). |
| **总容量 / Capacity** | Total cache size (6 – 1000). |
| **Window** | Window segment size (1 – capacity−1). |
| **草图宽 / Sketch width** | Count-Min Sketch width per hash row (8 / 16 / 32). |

> **View mode is automatic:** capacity ≤ 40 → per-item sliding animation; capacity > 40 → aggregate
> bar view (each row shows occupancy + the most recent keys).

---

## Workloads

The workload selector determines how the next key is sampled from a pool of **1500 keys**
(62 single characters + two-character combinations). An explanatory bar under the selector updates
live with the choice.

| Workload | Rule | Models | What you'll see |
|----------|------|--------|-----------------|
| **Random** | Uniform random pick over all 1500 keys. | Hash-like / uniform traffic with no hot spots. | Few repeats → low hit rate; Protected stays thin, Probationary carries the cache. The hardest case. |
| **Zipf** *(default)* | Power-law, `s = 1.5` (`P(rank r) ∝ 1/r^1.5`). | Realistic web/search/API/item popularity. | A few very hot keys promote into Protected and stay; high hit rate. W-TinyLFU's sweet spot. |
| **Scan** | Sequential sweep `0,1,2,…,1499`, then repeats. | Full-table scans, bulk export, backups. | A flood of one-time keys → demonstrates **scan resistance** (mass `❌ reject`). |
| **Loop** | Cycle through only the first 5 keys (`A–E`). | A tiny fixed working set polled repeatedly. | ~100% hit rate, but you can observe "the last couple of keys stay stuck in Window". |

---

## Visualization Panels

1. **Access Stream** — the last 16 accesses (green = hit, red = miss).
2. **Step Breakdown** *(本步解析)* — a readable, phase-by-phase explanation of the current step:
   `① sketch update → ② access result → ③ admission/promotion/rejection → ④ aging`.
   Frequency comparisons spell out `candidate(freq) vs victim(freq) → verdict`.
3. **Operation History** — a compact rolling log of the last 8 steps (`✓ hit / ↓ admit / ✗ reject / ＋ enter window`).
4. **Cache Structure** — live Window / Protected / Probationary layout (per-item or aggregate).
5. **Count-Min Sketch** — the 4-row frequency grid; cell color = estimated frequency
   (blue → red). Shows the aging countdown and reset count.
6. **Admission Decision** — the candidate-vs-victim frequency bars and verdict, shown when an
   admission decision is made.
7. **Stats** — accesses, hits, misses, hit rate, aging count.

---

## Algorithm Details

### Sizing

```
mainSize        = capacity − windowSize
protectedSize   = floor(mainSize × 0.8)     // hard cap on Protected
probationarySize= mainSize − protectedSize  // ~20%, steady-state share (NOT a hard cap)
sampleSize      = 10 × capacity             // sketch aging period
```

### Count-Min Sketch

- **4 hash rows**, each `sketchWidth` cells wide.
- **4-bit counters** (saturating at **15**).
- `freq(key) = min` of the 4 hashed cells (the classic Count-Min min estimator).
- **Aging:** every `sampleSize` accesses, all counters are halved with ceiling
  (`cell = (cell + 1) >>> 1`), so a counter at 1 never decays to 0 — once-seen keys retain a
  minimum frequency. This keeps the sketch focused on *recent* traffic.

### TinyLFU admission

- Victim = **Probationary LRU** (the global LRU of Main).
- Admit the candidate iff `freq(candidate) > freq(victim)` (strictly greater). Ties reject the
  candidate (favoring the incumbent).
- While Main is not yet full, candidates are admitted unconditionally (no frequency check) — this
  is the warm-up/fill phase.

### Promotion (SLRU within Main)

- A hit in **Probationary** promotes the key to **Protected**.
- If Protected is full, its LRU is **demoted** to Probationary (placed at Probationary's MRU).
- A hit in **Protected** simply refreshes to MRU.

---

## Design Notes & Caveats

This is an **educational** visualization. A few deliberate simplifications versus a production
cache (e.g., Caffeine):

- **Fixed Window size.** Here `windowSize` is a constant you set. Real Caffeine uses an **adaptive
  hill-climbing optimizer** that continuously retunes the Window↔Main split based on observed hit
  rate (growing the Window for recency-heavy workloads, shrinking it for frequency-heavy ones).
- **Probationary has no independent hard cap.** It shares Main's space with Protected; its effective
  ceiling is `mainSize − protected.length`, so it can hold up to all of Main while Protected is
  empty during warm-up, and only settles to ~20% once Protected fills. The displayed denominator
  reflects this dynamic ceiling (it shrinks as Protected grows) — this is correct, not overflow.
- **Strict `>` admission.** Production tunables sometimes favor the incumbent differently; this
  implementation rejects on ties.
- **Frequency is the only admission signal.** Real caches layer in heuristics (e.g., Caffeine's
  adaptive window, sampling); this implementation keeps the policy pure for clarity.
- The victim selection falls back to Protected's LRU only if Probationary is empty, which is
  effectively unreachable once Main is full (Protected is capped below `mainSize`). It is retained
  as a harmless guard.

### Try this

- **Scan workload + Step** → watch the frequency gate reject one-time floods.
- **Zipf + Play** → watch hot keys climb into Protected while the hit rate climbs.
- **Loop workload** → reproduce the "hot keys stuck in Window" effect (Window eviction requires new
  keys to push them out; with a tiny fully-cached working set, nothing does).
- **Reduce capacity to ≤ 40** → switch to the per-item sliding animation and step slowly.

---

*Single file, vanilla JS, no dependencies. Built for understanding.*
