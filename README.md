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

Once the plugin is available in a marketplace:

```
/plugin marketplace add anthropics/claude-plugins-community
/plugin install qorion-perf@claude-community
```

### Local testing (development)

```bash
claude --plugin-dir ./qorion-perf
```

## Usage

The skill is model-invoked — Claude activates it automatically based on the task. You can also invoke it directly:

```
/qorion-perf:performance-optimization
```

## License

MIT — see [LICENSE](./LICENSE).
