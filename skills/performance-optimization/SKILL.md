---
name: performance-optimization
description: >-
  Make code faster, leaner, and smoother WITHOUT breaking functionality. A
  safety-first, measure-driven methodology for performance work in any language
  or runtime. Use this skill whenever the user wants to optimize, speed up,
  profile, reduce latency, fix a bottleneck, lower memory or CPU usage,
  eliminate jank/stutter/frame drops, shrink bundle size, fix slow queries or
  N+1 problems, reduce allocations or GC pressure, or generally improve
  performance — and ALSO whenever you are about to change code for performance
  reasons mid-task, even if the user never said the word "performance". Ships
  language-specific references for C++/gamedev, React/Next.js, and C#/.NET on
  top of universal rules that keep every optimization behavior-preserving so it
  never introduces regressions, race conditions, stale data, or subtle behavior
  changes.
---

# Performance Optimization

## The one rule

**Faster but wrong is worthless.** A performance change is only acceptable if the
code's *observable behavior* stays identical: same outputs for the same inputs,
same side effects, same error handling, same edge cases, same ordering and
threading guarantees. Speed is the goal; correctness is the constraint that is
never traded away.

This is the discipline that separates real optimization from the most common
failure mode — making code "look faster" by quietly deleting a guard clause,
reordering floating-point math, caching something mutable, or parallelizing code
that shares state. Those changes pass a casual glance and a happy-path test, then
break in production. This skill exists to make that failure mode structurally
hard to fall into.

## Mindset

- **Be a scientist, not a vandal.** Form a hypothesis ("this loop is the
  bottleneck"), gather evidence, change one thing, measure the result. No guessing.
- **The slow code is usually right.** Assume existing code does what it does for a
  reason until proven otherwise. That "redundant" null check, that extra copy,
  that eager evaluation — treat each as load-bearing until you understand it.
- **Optimize the thing that matters.** 90% of runtime usually lives in a tiny
  fraction of the code. Optimizing anything else is churn that adds risk and
  complexity for zero user-visible gain.
- **Readability is a feature.** A 3% speedup that makes code unmaintainable is
  almost always a bad trade. Only sacrifice clarity for measured, meaningful wins
  in proven hot paths — and comment *why*.

---

## The workflow

Follow these phases in order. Do not skip Phase 0 and Phase 1 — optimizing
without establishing ground truth and finding the real bottleneck is how you
spend effort speeding up code that was never slow while breaking code that was.

### Phase 0 — Establish ground truth

Before changing anything, pin down three things:

1. **What "correct" means.** Find the existing tests. Run them and confirm they
   pass *now* — this is your regression net. If there are no tests covering the
   code you're about to touch, that is a risk: either add a minimal
   characterization test (capture current outputs for representative inputs,
   including edge cases) or flag to the user that you'll be optimizing without a
   safety net and proceed with extra caution. Never optimize untested code
   silently.
2. **What is actually slow.** Get a concrete symptom: a slow endpoint, a frame
   that drops below budget, a function that dominates a profile, a query that
   takes seconds, a bundle that's too large. "This feels slow" is a starting
   point, not a target.
3. **A baseline measurement.** Capture a number you can compare against:
   wall-clock time, p50/p99 latency, frame time, allocations, bundle bytes,
   memory high-water mark. Without a baseline you cannot prove improvement, and
   "it feels faster" is how placebo optimizations get shipped.

### Phase 1 — Find the real bottleneck (measure, don't guess)

Intuition about performance is famously unreliable; even experts guess wrong.
**Profile or instrument before you optimize.**

- Prefer a real profiler for the language/runtime (see the reference files for
  which tools to reach for). Run it against a realistic workload, not a toy input.
- If you can't profile easily, add targeted instrumentation: time the suspect
  regions, count iterations/allocations/calls, log sizes. Measure, then remove
  the instrumentation.
- When neither is practical, fall back to *complexity analysis*: reason about
  Big-O, data sizes, and call frequency to locate the hot path — but treat the
  conclusion as a hypothesis to verify, and say so.

Identify the **single dominant cost** before touching anything. The biggest wins
are almost always algorithmic or architectural (an O(n²) loop, an N+1 query, a
render storm, a per-frame allocation), not micro-optimizations.

### Phase 2 — Optimize in order of impact

Work top-down through the optimization hierarchy. Each tier below is typically an
order of magnitude less impactful — and more risky relative to its payoff — than
the one above it. Exhaust the cheap, high-impact tiers before descending.

1. **Do less work.** The fastest code is the code that doesn't run. Eliminate
   redundant work, avoid recomputation (cache/memoize *pure* results), short-circuit
   early, batch operations, remove duplicate queries/requests, skip work whose
   result is unused.
2. **Better algorithm or data structure.** Reduce complexity class: the right
   container for the access pattern, a hash lookup instead of a linear scan,
   sorting once instead of repeatedly, the right index on a query. This is where
   100×–1000× wins live.
3. **Better data layout & access.** Make memory access cheap: contiguous data,
   cache-friendly layout, fewer pointer chases, fewer allocations, less garbage.
   On hot paths this often beats algorithmic cleverness.
4. **Parallelism & concurrency.** Use multiple cores / async I/O *only where it's
   safe and the work is genuinely independent*. This tier carries the highest risk
   of introducing correctness bugs — see the behavioral-equivalence rules below.
5. **Micro-optimizations.** Branch hints, intrinsics/SIMD, avoiding small
   allocations, loop unrolling. Last resort, smallest payoff, and only with a
   measurement proving it helps. Most "clever" micro-optimizations are noise the
   compiler already does.

### Phase 3 — Preserve behavior (the equivalence gate)

Before applying any change, pass it through the **Behavioral Equivalence
Checklist** (next section). If a change can alter observable behavior, it is not
a safe optimization — either find a behavior-preserving version or escalate the
tradeoff to the user explicitly instead of deciding silently.

### Phase 4 — Apply, verify, and measure — one change at a time

- **One optimization per step.** Bundling changes makes it impossible to attribute
  gains or localize a regression. Apply, verify, then move on.
- **Re-run the tests after every change.** They must still pass. A green baseline
  going red is a stop-everything signal, not a "fix it later".
- **Re-measure after every change.** Confirm the change actually helped against the
  Phase 0 baseline — and where the win isn't self-evident, make that proof tangible
  by writing a benchmark (see "Prove it with numbers" below). Optimizations that
  don't measurably help get **reverted** — they add complexity and risk for nothing.
  This happens more often than you'd expect; keep only what pays.
- **Revert freely.** A reverted optimization is a successful experiment, not a
  failure. Keeping a change that didn't help (or that you can't verify is safe) is
  the actual failure.

### Phase 5 — Report honestly

When done, tell the user:

- **What changed and why** — the bottleneck found and the fix applied, per change.
- **Measured impact** — before/after numbers, not adjectives. "p99 480ms → 110ms"
  beats "much faster".
- **What you deliberately left alone** — and why (e.g. "the validation loop looks
  hot but it's correctness-critical and runs once per request, not in the inner
  loop, so I left it").
- **Any tradeoff or risk** — reduced readability, a new dependency, an assumption
  the speedup relies on, anything the user should sign off on.
- **Honesty about uncertainty.** If a change is plausibly faster but you couldn't
  measure it, say so plainly. Never report unverified speedups as facts.

---

## Prove it with numbers — write the benchmark

"Measure" doesn't only mean an internal sanity check. When the win isn't
self-evident, make the proof **tangible and re-runnable**: write an actual
benchmark / perf test that runs the old and new versions against the same workload,
run it, and show the user the numbers. A benchmark you leave behind also becomes a
regression guard the next person can re-run.

**A good benchmark does two things at once:**
1. **Verifies equivalence** — run both versions on the same inputs and confirm they
   produce identical output *before* trusting the timing. A "faster" version that's
   wrong is not a speedup. This is the equivalence rule, made executable.
2. **Measures honestly** — warm up, run enough iterations to beat noise, and report
   the metric that matters (time/op, p99, allocations, bundle bytes, query count)
   with some sense of variance. One noisy run isn't evidence.

**Show the real output.** Paste the benchmark's actual table/numbers, not a
paraphrase. "412 ns → 78 ns, 4 allocations → 0" is proof; "much faster" is a claim.

**Use judgment about when to write one** — the user doesn't want ceremony on every
trivial change:

- **Write a real benchmark when** the win is non-obvious or could plausibly be
  neutral/negative; it's a hot path worth a permanent regression guard; the code
  isolates cleanly into a harness; or the user wants proof.
- **A quick timing confirmation is enough when** the fix is an obvious, large
  algorithmic/structural win on a clearly-hot path (O(n²)→O(n log n), killing an
  N+1): confirm the number moved, but a full benchmark suite is overkill.
- **Fall back to profiler numbers / in-place instrumentation when** the code can't be
  isolated into a harness or the real workload can't be reproduced offline (e.g.
  latency dominated by a specific production dataset). Report those numbers instead,
  and label them for what they are.
- **Never fabricate.** If you genuinely can't measure, say so — don't present an
  estimated or assumed speedup as a measured one.

Each language reference gives the idiomatic harness for its stack (BenchmarkDotNet
for C#, nanobench/Google Benchmark for C++, the React/Next measurement options) with
a concrete skeleton.

---

## Behavioral Equivalence Checklist

This is the core of the skill. Most performance regressions aren't crashes — they
are *silent behavior changes* that look fine and pass happy-path tests. Run every
proposed change against this list. If you can't confidently check a box, the
change is not safe as-is.

**Outputs & logic**
- [ ] Same return values / outputs for the same inputs — including empty, null,
  zero, negative, boundary, and malformed inputs, not just the typical case.
- [ ] Edge cases preserved: off-by-one boundaries, empty collections, single-element
  collections, max/min values, unicode, timezones.
- [ ] No "redundant" check removed without proving it can never fire. Guard clauses,
  bounds checks, null checks, and defensive copies are load-bearing until proven
  otherwise.

**Numerical correctness**
- [ ] Floating-point results unchanged where they matter. Reordering, fusing
  (FMA), `-ffast-math`, vectorization, and parallel reduction all change rounding.
  This silently breaks anything depending on bit-exact results: lockstep/replay
  netcode, financial math, hashing, golden-file tests, cross-platform determinism.
- [ ] Integer overflow / wraparound / precision behavior unchanged.

**Side effects & ordering**
- [ ] Side effects happen the same number of times, in the same order, with the
  same observable timing. Don't turn eager into lazy (or vice versa) when the
  *when* of a side effect, log, mutation, or I/O is observable.
- [ ] Iteration / result ordering preserved when anything depends on it. Switching
  collection types or parallelizing can scramble order silently.
- [ ] Laziness/streaming semantics preserved (e.g. a deferred/lazy sequence that
  must not be fully materialized, or must be enumerated exactly once).

**Errors & contracts**
- [ ] Error handling identical: same exceptions/errors, same types, same conditions,
  same messages where code or users depend on them. Don't swallow, change, or
  defer errors for speed.
- [ ] Public API / contract unchanged: signatures, nullability, thread-safety
  guarantees, ownership/lifetime semantics, documented behavior. Optimize the
  *implementation*, preserve the *interface* — unless the user explicitly approves a
  contract change.

**Concurrency (highest-risk tier)**
- [ ] No new shared mutable state introduced by parallelization. Confirm the
  parallelized work is genuinely independent and touches no shared data without
  synchronization.
- [ ] No race conditions, torn reads/writes, deadlocks, or ordering assumptions
  broken. Concurrency bugs are nondeterministic and often invisible in tests —
  reason about them explicitly, don't test-and-hope.
- [ ] Async semantics preserved: don't block on async (deadlock risk), don't change
  what context/scheduler work resumes on when it's observable.

**Caching & memoization**
- [ ] Anything cached/memoized is actually *pure and stable*. Caching mutable or
  time-varying data produces stale results — a correctness bug, not a perf win.
- [ ] Cache invalidation is correct, or the cached value is provably immutable for
  its lifetime. Cache keys capture every input that affects the result.

When a faster approach *can't* preserve behavior (e.g. the user genuinely wants
approximate-but-fast, or to drop a determinism guarantee), that's a legitimate
choice — but it's the **user's** choice. Surface the tradeoff and let them decide;
never make it silently.

---

## When NOT to optimize

Performance work has a cost (risk, complexity, time). Don't pay it when:

- **It isn't measurably slow.** No symptom, no profile, no complaint → don't
  "pre-optimize." Premature optimization adds risk and obscures intent for no gain.
- **The code isn't hot.** A function called once at startup doesn't need a faster
  algorithm. Optimize the inner loop, not the setup.
- **The win is negligible.** A 2% improvement that complicates the code or adds a
  dependency rarely justifies itself.
- **The bottleneck is elsewhere.** No point shaving CPU off a function when the
  request spends 95% of its time waiting on the database or network. Fix the
  dominant cost.
- **You'd be guessing.** If you can't identify the real bottleneck and can't
  measure, say so rather than optimizing on vibes. Targeted "make this specific
  thing fast" is fine; "make it faster" with no evidence and no way to measure is
  a prompt to instrument first.

A correct, clear, fast-enough solution beats a clever, fragile, marginally faster
one almost every time.

---

## Language-specific references

The universal workflow and equivalence checklist above apply everywhere. For
domain-specific hotspots, tools, idioms, and traps, read the matching reference
file **after** you've identified the bottleneck — pull in only what's relevant to
the code in front of you:

- **C++ / game development** → `references/cpp.md`
  Frame-budget thinking, data-oriented design and cache behavior, allocation
  discipline (pools/arenas), move semantics, SIMD, the determinism trap for
  netcode/replays, avoiding undefined behavior, and profiling tools.

- **React / Next.js** → `references/react.md`
  Render behavior and correct memoization (and why over-memoization is its own
  bug), dependency-array and stale-closure traps, bundle size and code splitting,
  Server vs Client Components, data-fetching waterfalls and streaming, Next.js
  caching layers, hydration-mismatch dangers, and profiling tools.

- **C# / .NET** → `references/csharp.md`
  Allocations and GC pressure (`Span<T>`, pooling), LINQ pitfalls (multiple
  enumeration, deferred execution), async correctness (`ConfigureAwait`,
  `ValueTask`, sync-over-async deadlocks), struct/boxing, EF Core (N+1, tracking,
  projection), and profiling tools.

If the language or framework in front of you isn't one of these, apply the
universal workflow and checklist directly — they are language-agnostic by design.
Reach for the same evidence-driven, behavior-preserving discipline regardless of
stack.
