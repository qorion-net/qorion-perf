# C++ / Game Development Performance

Read this once you've identified a C++ hotspot. Apply the universal workflow and
Behavioral Equivalence Checklist from `SKILL.md` first — this file covers the
domain-specific hotspots, tools, idioms, and traps. Don't apply techniques from
here blindly; apply the one that targets the bottleneck you actually measured.

## Table of contents
1. Profiling tools
2. The frame-budget mindset
3. Where C++ time actually goes
4. Optimization techniques (in impact order)
5. The determinism trap (critical for games)
6. C++-specific behavior-preservation traps
7. Quick-reference checklist

---

## 1. Profiling tools

Measure with a real profiler before changing anything. Reach for:

- **Tracy** — frame-by-frame profiler, ideal for games; instrument zones and see
  per-frame cost, allocations, lock contention, GPU timing.
- **In-engine profilers** — Unreal Insights / `stat` commands, Unity Profiler if
  relevant, or the engine's own frame profiler. Often the fastest path to the hot
  frame.
- **perf** (Linux) / **VTune** (Intel) / **Instruments** (macOS) — sampling
  profilers for CPU hotspots and cache/branch analysis.
- **Compiler reports** — `-fopt-info`, `-Rpass=` (Clang) to see what got
  vectorized/inlined; godbolt.org to inspect generated assembly for a hot function.
- **Sanitizers as correctness guards** — ASan/UBSan/TSan don't measure speed, but
  run your test workload under them after optimizing to catch UB, races, and
  lifetime bugs an aggressive optimization may have introduced.

For games, profile a *representative frame under load*, not an idle menu. The hot
path in a stress scene is what matters.

### Writing a micro-benchmark to prove the win

For a hot kernel where the win isn't obvious, write a micro-benchmark with
**nanobench** (single header, trivial to drop in) or **Google Benchmark**. Confirm
the new version produces identical output first, then measure — it reports ns/op,
ops/s, and error % per variant so you can see both speed and noise:

```cpp
#include <nanobench.h>
ankerl::nanobench::Bench().run("old", []{ ankerl::nanobench::doNotOptimizeAway(old_impl(data)); });
ankerl::nanobench::Bench().run("new", []{ ankerl::nanobench::doNotOptimizeAway(new_impl(data)); });
```

Paste the printed table for the user. **Two C++ benchmarking traps:** numbers are
meaningless unless you build with optimizations on (release, not `-O0`) and pin the
machine to stable conditions (CPU frequency scaling / turbo make runs jump around);
and use `doNotOptimizeAway` (or equivalent) so the compiler doesn't delete the work
you're trying to time. For whole-frame work that doesn't isolate into a harness,
compare Tracy zone timings (or a `std::chrono::steady_clock` timer averaged over N
frames) before vs after instead.

---

## 2. The frame-budget mindset

Games live or die on consistency, not average throughput. The unit is the **frame
budget**:

- 60 FPS → 16.6 ms per frame. 120 FPS → 8.3 ms. 30 FPS → 33.3 ms.
- A function isn't "fast enough" in the abstract — it's fast enough if it fits its
  slice of the frame budget *in the worst case*, every frame.
- **Spikes matter more than averages.** A 16 ms average with an occasional 50 ms
  hitch feels worse than a steady 20 ms. Hunt the p99 frame, the stutter, the
  hitch — not just the mean. A single per-frame allocation that occasionally
  triggers a heap walk can blow the budget intermittently.
- Separate **fixed-timestep** work (simulation/physics) from **variable** work
  (rendering). Optimizing the wrong one wastes effort.

---

## 3. Where C++ time actually goes

The usual suspects, roughly in order of how often they dominate:

- **Cache misses / poor data layout** — the #1 hidden cost in modern C++. The CPU
  spends most of its time waiting on memory. Chasing pointers through scattered
  heap allocations (linked lists, vectors of pointers, AoS with cold fields) stalls
  constantly.
- **Allocations in hot loops** — `new`/`delete`/`malloc` per frame or per entity is
  slow and fragments the heap. Hidden allocations hide in `std::string`,
  `std::function`, growing `std::vector`, lambdas captured by value into `std::function`.
- **Copies** — passing big objects by value, returning by value without move,
  copying containers, copying in range-for. Often invisible until you look.
- **Virtual dispatch in tight loops** — indirection + missed inlining + branch
  misprediction. Real cost, but don't "fix" it by breaking polymorphism (see traps).
- **Algorithmic** — O(n²) collision checks, re-sorting every frame, rebuilding
  structures that could be incremental.

---

## 4. Optimization techniques (in impact order)

### Data-oriented design / cache-friendliness (usually the biggest win)
- **Contiguous storage.** Prefer `std::vector` over node-based containers
  (`list`, `map`, `set`) for anything iterated hot. Contiguity makes the prefetcher
  work for you.
- **Structure of Arrays (SoA) vs Array of Structures (AoS).** When a loop touches
  only a few fields of many objects, store those fields in parallel arrays so each
  cache line carries only hot data. This is the classic ECS win.
- **Hot/cold splitting.** Keep frequently-accessed fields together; move rarely-used
  fields (debug names, config) to a separate struct so they don't pollute cache lines.
- **`reserve()`** containers when the size is known to avoid reallocation churn.
- **Reduce pointer chasing.** Indices into a contiguous array often beat pointers.

### Allocation discipline
- **No allocation in the inner loop / per frame.** Allocate up front and reuse.
- **Object pools** for entities/particles/events that churn — reuse instead of
  `new`/`delete`.
- **Arena / bump allocators** (`std::pmr::monotonic_buffer_resource`) for per-frame
  scratch data: allocate fast, free everything at once at frame end.
- **`std::pmr` containers** to route container allocations through a pool/arena.
- Watch for hidden allocations: small-string-optimization limits, `std::function`
  heap fallback, `vector` growth, `shared_ptr` control blocks.

### Avoid copies
- Pass read-only params by `const&` (or by value for cheap/trivially-copyable types).
- Use `std::move` for transfers; mark moves `noexcept` so containers use them.
- Use `emplace_back` over `push_back` for in-place construction.
- Bind range-for by reference: `for (const auto& x : container)`, never plain `auto`
  for non-trivial types.
- Return by value and let RVO/move handle it — don't fight the compiler with output
  params unless measured.

### Reduce indirection (carefully)
- Devirtualize *only* where the dynamic type is provably fixed, or batch by type so
  the same virtual is called across a homogeneous run. Never collapse a genuine
  polymorphic hierarchy just for speed — that changes behavior (see traps).
- `final` on classes/methods lets the compiler devirtualize safely.
- Prefer `constexpr`/templates to push work to compile time where it doesn't
  explode build times or binary size.

### SIMD and micro-optimizations (last, with measurement)
- Let the compiler auto-vectorize first (`-O2`/`-O3`, restrict aliasing, simple
  loops). Check the opt report before hand-writing intrinsics.
- Hand-written SIMD only for proven hot kernels — mind alignment, tail handling, and
  that it changes FP rounding (determinism trap).
- Branch hints (`[[likely]]`/`[[unlikely]]`), `__restrict`, loop unrolling: trust
  the profiler, not folklore. The compiler already does most of this.

---

## 5. The determinism trap (CRITICAL for games)

This is the C++ optimization mistake most likely to silently break a game.

Many games depend on **bit-exact reproducibility**:
- **Lockstep multiplayer** (RTS, fighting games, many indie netcode models) runs the
  same simulation on every client and only syncs inputs. If two machines compute
  even slightly different floats, they desync.
- **Replays / kill-cams / ghosts** replay recorded inputs and must reproduce the
  exact same world.
- **Deterministic physics / procedural generation** with a fixed seed.

These optimizations **change floating-point results** and will silently break the
above:
- `-ffast-math` / `/fp:fast` — reassociates and fuses FP math; the single most
  common cause of desyncs introduced by "optimization."
- **FMA contraction** (`-ffp-contract=fast`) — fuses `a*b+c`, changing rounding.
- **Auto-vectorization / SIMD** of FP reductions — changes the order of additions,
  and FP addition isn't associative.
- **Parallel reductions** — same problem: order-dependent rounding.
- Reordering FP operations, changing transcendental implementations, or swapping a
  math library.

**Rule:** before applying any FP-affecting optimization, determine whether the code
is on a deterministic path. If it is (or you can't be sure), keep strict FP
(`-ffp-contract=off`, `/fp:strict`-style guarantees) on that path and optimize
elsewhere — integer math, memory layout, and allocation are all
determinism-safe. If the user genuinely wants to trade determinism for speed, that
is their call to make explicitly, not a silent side effect of a flag.

---

## 6. C++-specific behavior-preservation traps

Beyond the universal checklist, in C++ specifically:

- **Don't introduce undefined behavior for speed.** Type-punning via reinterpret
  casts, breaking strict aliasing, unaligned access, signed overflow, reading
  uninitialized memory — UB may *look* faster and pass tests, then explode under a
  different compiler/optimization level. Verify with UBSan.
- **Object lifetime & ownership.** "Optimizing away" a copy can leave a dangling
  reference/pointer to a temporary. Moving instead of copying leaves the source in a
  valid-but-unspecified state — make sure nothing reads it after.
- **Exception safety.** Don't break the strong/basic exception guarantee while
  reorganizing for speed. Marking something `noexcept` that can actually throw means
  `std::terminate` on throw.
- **`const` / thread-safety contracts.** `const_cast`-ing away const to mutate, or
  removing synchronization you assumed was "unnecessary," breaks the guarantees
  callers rely on.
- **Iterator/reference invalidation.** Reserving, resizing, or swapping container
  internals can invalidate iterators/pointers held elsewhere.
- **Integer types & overflow.** Narrowing an index type or changing signedness to
  "save a register" can change wraparound and comparison behavior.

After an aggressive C++ optimization pass, re-run the test suite under ASan/UBSan
(and TSan if you touched threading). These catch the lifetime/UB/race bugs that
optimization most often introduces and that plain tests miss.

---

## 7. Quick-reference checklist

- [ ] Profiled a representative frame/workload; identified the single dominant cost.
- [ ] Baseline frame time / p99 / allocation count captured.
- [ ] Targeted the right tier: layout & allocation before micro-opts.
- [ ] Checked the determinism path before any FP-affecting change.
- [ ] No new UB; verified under UBSan/ASan (TSan if threading changed).
- [ ] Lifetimes, exception safety, and `const`/thread guarantees intact.
- [ ] Re-measured: the change actually fits the budget better. Reverted if not.
