# C# / .NET Performance

Read this once you've identified a C#/.NET hotspot. Apply the universal workflow
and Behavioral Equivalence Checklist from `SKILL.md` first. This file covers
.NET-specific hotspots, tools, idioms, and the traps where a perf change breaks
correctness (async deadlocks and LINQ double-enumeration are the classics). Apply
the technique that targets the bottleneck you measured.

## Table of contents
1. Profiling & benchmarking tools
2. Where .NET time actually goes
3. Allocation & GC pressure
4. LINQ pitfalls
5. Async correctness & performance
6. EF Core / data access
7. C#-specific behavior-preservation traps
8. Quick-reference checklist

---

## 1. Profiling & benchmarking tools

Measure before changing. For .NET specifically:

- **BenchmarkDotNet** — the gold standard for micro-benchmarks. Use it to *prove* a
  hot-path optimization helps (it handles warmup, JIT, and reports allocations per
  op). Don't claim a micro-optimization win without it.
- **dotnet-trace** / **dotnet-counters** — sampling profiler and live counters
  (GC rate, gen sizes, allocation rate, thread pool queue, exceptions/sec) for a
  running process, including in production-like environments.
- **dotnet-gcdump** / **dotnet-dump** — heap analysis for memory growth/leaks and
  what's surviving GC.
- **PerfView** — deep ETW analysis: allocations by type, GC pauses, CPU stacks,
  contention. Heavyweight but definitive.
- **Visual Studio Profiler / Rider** — integrated CPU, allocation, and timeline tools.
- **EF Core logging / MiniProfiler** — log generated SQL and timings to catch N+1 and
  slow queries (see §6).

For a web app, profile under realistic concurrency, not a single request — thread
pool starvation and contention only show under load.

### Writing a benchmark to prove the win

When the win is non-obvious or worth a permanent regression guard, write a
**BenchmarkDotNet** benchmark comparing old vs new — it handles warmup/JIT and, with
`[MemoryDiagnoser]`, reports allocations per op. Assert both versions return equal
results first (in a test), *then* trust the timing:

```csharp
[MemoryDiagnoser]
public class HotPathBench
{
    private readonly Input _data = MakeRepresentativeInput(); // realistic size

    [Benchmark(Baseline = true)]
    public Result Old() => OldImpl(_data);

    [Benchmark]
    public Result New() => NewImpl(_data);
}
// run: dotnet run -c Release   (Release only — Debug numbers are meaningless)
```

Paste the result table (Mean, Ratio, Allocated) for the user — that's the proof. For
I/O-bound paths that don't isolate cleanly (a slow endpoint or query), don't force a
micro-benchmark: measure end-to-end with a `Stopwatch` harness, or compare logged EF
query counts/timings before vs after, and say that's what the numbers represent.

---

## 2. Where .NET time actually goes

- **Allocations & GC** — excessive gen0 churn (allocating in hot loops/per request),
  Large Object Heap allocations, and the GC pauses they cause. Often the dominant
  cost in throughput-sensitive services.
- **Boxing** — value types silently boxed (into `object`, non-generic collections,
  string formatting, `params object[]`) allocate and pressure the GC.
- **Sync-over-async / thread pool starvation** — blocking on async work exhausts the
  thread pool under load, tanking throughput (and risking deadlock — see §5).
- **LINQ overhead & repeated enumeration** — convenient but allocates iterators and
  can re-execute an expensive pipeline multiple times unintentionally.
- **Inefficient data access** — N+1 queries, fetching whole entities when a
  projection would do, tracking entities you only read, missing indexes.
- **String work** — concatenation in loops, repeated formatting, unnecessary
  substrings/splits.

---

## 3. Allocation & GC pressure

The highest-leverage .NET optimization is usually *allocating less*.

- **`Span<T>` / `ReadOnlySpan<T>`** — slice arrays/strings/buffers without copying;
  parse and process in place. Huge for hot parsing/serialization paths.
- **`stackalloc`** — stack-allocate small fixed buffers instead of heap arrays (mind
  the size limit; combine with `Span<T>`).
- **`ArrayPool<T>.Shared`** — rent/return reusable buffers instead of allocating
  per operation. Return in a `finally`; don't use the buffer after returning it.
- **`ObjectPool<T>`** (Microsoft.Extensions.ObjectPool) — pool expensive-to-create
  reusable objects (e.g. `StringBuilder`).
- **`readonly struct` / `ref struct`** — avoid defensive copies; `ref struct`
  guarantees stack-only (like `Span`).
- **Avoid LOH allocations** (objects ≥ 85,000 bytes) — they're collected only in
  gen2 and fragment; pool or stream large buffers instead.
- **`StringBuilder`** for building strings in loops; **string interpolation** is fine
  for one-offs but allocates — avoid it in hot loops and high-volume logging (use
  structured logging with message templates so args aren't formatted unless logged).
- Choose the right collection and **pre-size it** (`new List<T>(capacity)`,
  `Dictionary` capacity) to avoid repeated internal reallocation. Use `Dictionary`/
  `HashSet` for lookups instead of `List.Contains` (O(n)).

Verify with `dotnet-counters` (allocation rate, gen0/1/2 counts) or BenchmarkDotNet's
memory diagnoser that allocations actually dropped.

---

## 4. LINQ pitfalls

LINQ is readable and usually fine — optimize it only when it's on a measured hot
path. The traps:

- **Multiple enumeration (also a correctness trap).** An `IEnumerable<T>` query is
  lazy; enumerating it twice (`.Count()` then `foreach`, or iterating in a loop)
  *re-runs the whole pipeline* — and if the source is a DB query or has side effects,
  you get duplicate work, duplicate queries, or different results each time.
  Materialize once with `.ToList()`/`.ToArray()` when you'll enumerate more than once.
- **Deferred execution semantics.** `Where`/`Select` don't run until enumerated. Code
  may *rely* on this laziness (e.g. infinite/streaming sequences, or executing the
  query at a specific later point against fresh data). Don't force eager
  materialization where the deferral is intentional — that changes behavior.
- **Allocation overhead in hot loops.** Each LINQ operator allocates an iterator and
  closures capture variables. In a tight inner loop, a plain `for`/`foreach` can be
  meaningfully faster and allocation-free. Only rewrite where measured.
- **Avoid redundant passes.** `.Where(...).First()` is fine (short-circuits);
  `.Count() > 0` should be `.Any()`; `.Where(...).Count()` can be `.Count(predicate)`.
  Don't run multiple full passes when one suffices.
- **(EF Core) keep it queryable.** Don't call `.ToList()` early and then filter
  in-memory — that pulls the whole table into the app. Filter/project in the
  `IQueryable` so it translates to SQL (see §6).

---

## 5. Async correctness & performance

Async is mostly about *throughput and scalability*, and it's where perf changes most
often introduce serious bugs.

- **Never block on async (no sync-over-async).** `.Result`, `.Wait()`,
  `.GetAwaiter().GetResult()` on an async call can **deadlock** in contexts with a
  synchronization context (classic ASP.NET, WPF/WinForms UI) and starves the thread
  pool under load everywhere. Make the call chain async end-to-end instead.
  "Optimizing" by blocking is a bug, not a speedup.
- **`ConfigureAwait(false)` in library code.** Avoids capturing the synchronization
  context on continuations — reduces overhead and avoids deadlocks. Use it in
  general-purpose/library code. In ASP.NET Core there's no sync context so it's moot;
  in UI code, code *after* the await that touches UI must run on the UI thread, so
  don't blanket-apply it where the captured context is needed.
- **Parallelize independent async work** with `Task.WhenAll` rather than awaiting
  sequentially — the most common easy async latency win. But don't share non-thread-
  safe state (e.g. one `DbContext`) across concurrent tasks (see §6).
- **`ValueTask` / `ValueTask<T>`** for hot async methods that frequently complete
  synchronously (e.g. cache hits) to avoid a `Task` allocation per call. Respect its
  rules: await it once, don't block on it, don't reuse it.
- **`IAsyncEnumerable<T>`** to stream async sequences without buffering everything.
- **CPU-bound vs I/O-bound:** async is for I/O. Don't wrap CPU work in
  `Task.Run` inside a request handler expecting a speedup — it just shuffles thread
  pool threads. Parallelize CPU work with `Parallel.For`/PLINQ where it's safe.

---

## 6. EF Core / data access

Data access is the most common real bottleneck in a .NET web/enterprise app — and
the place where a "speedup" most easily returns wrong data.

- **Kill N+1 queries.** Lazy loading or per-item queries in a loop fire one query per
  row. Use `Include`/`ThenInclude` (or split queries / explicit projection) to load
  related data in one round trip. Log the generated SQL to confirm the count dropped.
- **Project to what you need.** `Select` into a DTO/anonymous type so SQL returns only
  the needed columns instead of hydrating full entities. Big win for read paths.
- **`AsNoTracking()` for read-only queries.** Skips change-tracker overhead and
  snapshots. **But** — entities come back untracked, so saving changes on them won't
  persist. Only apply it where the result is genuinely read-only; applying it to a
  query whose results get modified and saved silently breaks the write.
- **Filter and paginate in the database.** Keep `Where`/`OrderBy`/`Skip`/`Take` in the
  `IQueryable` so they translate to SQL. Calling `.ToList()` before filtering pulls the
  whole table into memory — a correctness/perf disaster on large tables.
- **Watch client-side evaluation.** Expressions EF can't translate may execute
  in-memory after pulling rows; verify the query translates as intended.
- **Indexes & batching.** Ensure queried/sorted columns are indexed; use bulk
  operations (`ExecuteUpdate`/`ExecuteDelete` or a bulk library) instead of loading,
  mutating, and saving thousands of entities one by one.
- **One `DbContext` per unit of work, never shared across concurrent tasks.** A
  `DbContext` is not thread-safe; parallelizing queries over a shared context throws or
  corrupts. Use a context per parallel operation (and mind the pool).

---

## 7. C#-specific behavior-preservation traps

Beyond the universal checklist, in C#/.NET specifically:

- **Sync-over-async deadlock** (§5) — the single most damaging "optimization." Never
  block on async to avoid making a chain async.
- **`AsNoTracking()` on a write path** (§6) — silently stops changes from persisting.
- **LINQ multiple-enumeration** (§4) — re-runs side-effecting/DB queries; materialize
  before enumerating twice. And don't force-materialize a deliberately-deferred query.
- **Struct vs class is observable.** Changing a `class` to a `struct` for allocation
  reasons changes copy/assignment semantics, equality, nullability, and behavior when
  passed around or stored in collections (mutations on a copy don't stick). Not a
  drop-in swap.
- **`ConfigureAwait(false)` where context is needed** — in UI/legacy ASP.NET code,
  continuation code that touches the UI/`HttpContext` must run on the captured context.
  Don't blanket-apply it across such code.
- **Boxing changes equality/identity** in subtle cases; removing/adding boxing can
  alter reference-equality behavior.
- **`Parallel.For`/PLINQ over shared state** — introduces races. Ensure the body is
  independent and touches no shared mutable state without synchronization. Parallel
  LINQ can also reorder results unless `AsOrdered()` is used.
- **Floating-point / `decimal`.** Reordering or vectorizing FP math changes rounding;
  for money use `decimal` and don't "optimize" it into `double`.

After an aggressive async/parallel change, exercise it under concurrency — race and
thread-pool bugs don't appear in single-threaded tests.

---

## 8. Quick-reference checklist

- [ ] Profiled (dotnet-trace/counters, or BenchmarkDotNet for a micro-opt) and
  identified the dominant cost (allocations? GC? data access? blocking?).
- [ ] Baseline captured (latency, allocation rate, query count, or BDN numbers).
- [ ] Reduced allocations where hot (`Span`, pooling, pre-sizing) and verified the
  allocation rate actually dropped.
- [ ] Data access: N+1 eliminated, projecting needed columns, `AsNoTracking` only on
  read paths, filtering in SQL not memory.
- [ ] Async stays async end-to-end (no `.Result`/`.Wait()`); independent work
  parallelized without sharing a `DbContext`.
- [ ] No struct/class swap that changes semantics; no parallel access to shared state.
- [ ] Re-measured: the change actually helped. Reverted micro-opts that didn't pay.
