# qorion-perf

A Claude Code plugin for **safety-first, measure-driven performance optimization** in any language or runtime.

Makes code faster, leaner, and smoother **without breaking functionality**. The core discipline: a performance change is only acceptable if the code's *observable behavior* stays identical — same outputs, same side effects, same error handling, same edge cases, same ordering and threading guarantees. Speed is the goal; correctness is the constraint that is never traded away.

## What it does

When you ask Claude to optimize, speed up, profile, reduce latency, fix a bottleneck, lower memory/CPU usage, eliminate jank, shrink bundle size, fix slow queries or N+1 problems, or reduce allocations/GC pressure, this plugin's skill activates and guides the work through a strict methodology:

- **Phase 0 — Establish ground truth:** pin down what "correct" means, what is actually slow, and a baseline measurement.
- **Phase 1 — Find the real bottleneck:** profile or instrument before optimizing. No guessing.
- **Phase 2 — Optimize in order of impact:** do less work → better algorithm → better data layout → parallelism → micro-optimizations.
- **Phase 3 — Preserve behavior:** every change passes a Behavioral Equivalence Checklist.
- **Phase 4 — Apply, verify, measure:** one change at a time, re-run tests, re-measure, revert what doesn't pay.
- **Phase 5 — Report honestly:** before/after numbers, not adjectives.

### Language-specific references

On top of the universal workflow, the skill ships focused references for:

- **C++ / game development** — frame-budget thinking, data-oriented design, allocation discipline, SIMD, determinism traps.
- **React / Next.js** — render behavior, correct memoization, bundle size, Server vs Client Components, caching layers.
- **C# / .NET** — allocations and GC pressure, LINQ pitfalls, async correctness, EF Core (N+1, tracking, projection).

## Installation

This repository is also its own marketplace, so you can install the plugin
directly from GitHub — no need to wait for a community listing.

```
/plugin marketplace add qorion-net/qorion-perf
/plugin install qorion-perf@qorion
```

Or with the CLI:

```bash
claude plugin marketplace add qorion-net/qorion-perf
claude plugin install qorion-perf@qorion
```

Restart Claude Code (or run `/reload-plugins`) after installing. Verify with
`/plugin` or `claude plugin list` — you should see `qorion-perf` enabled.

> Once the plugin is also approved in the public community marketplace, you'll be
> able to install it with
> `/plugin marketplace add anthropics/claude-plugins-community` followed by
> `/plugin install qorion-perf@claude-community`.

### Local testing (development)

```bash
claude --plugin-dir ./qorion-perf
```

## Usage

There is nothing to "run" — the plugin ships a single **model-invoked skill**.
Claude activates it automatically whenever your request involves performance, so
you just describe what you want in plain language:

```
The /products endpoint feels slow under load — can you find out why and fix it?
Our React dashboard stutters when filtering the table.
This LINQ query allocates a ton on the hot path; reduce the garbage.
Profile this game loop, we're dropping frames in the physics step.
```

When the skill activates it will, at the start of any performance investigation,
**offer to switch on `ultracode`** (`/effort ultracode`) for a deeper,
multi-agent analysis — your choice, it never forces it.

You can also invoke the skill explicitly:

```
/qorion-perf:performance-optimization
```

### What happens once it's active

1. **Establishes ground truth** — finds/runs existing tests, pins down the real
   symptom, captures a baseline measurement.
2. **Finds the real bottleneck** — profiles or instruments before changing
   anything; no guessing.
3. **Optimizes in order of impact** — do less work → better algorithm/data
   structure → better data layout → parallelism → micro-optimizations.
4. **Preserves behavior** — every change passes a Behavioral Equivalence
   Checklist so outputs, side effects, ordering, and error handling stay
   identical.
5. **Verifies and measures** — one change at a time, re-runs tests, re-measures,
   reverts anything that doesn't measurably help.
6. **Reports honestly** — before/after numbers, what was left alone and why, and
   any tradeoffs.

For domain-specific work it automatically pulls in the matching reference only
when relevant — **C++/gamedev**, **React/Next.js**, or **C#/.NET** — keeping the
core skill lightweight until that depth is actually needed.

## License

MIT — see [LICENSE](./LICENSE).
