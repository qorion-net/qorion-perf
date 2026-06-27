# React / Next.js Performance

Read this once you've identified a React/Next bottleneck. Apply the universal
workflow and Behavioral Equivalence Checklist from `SKILL.md` first. This file
covers framework-specific hotspots, tools, and the traps that make a "perf fix"
introduce bugs. Apply the technique that targets the bottleneck you measured —
don't sprinkle `useMemo` everywhere on principle.

## Table of contents
1. Profiling tools
2. Where React/Next time actually goes
3. Render optimization (and the over-memoization trap)
4. Bundle size & loading
5. Next.js: Server Components, data fetching, caching
6. React/Next behavior-preservation traps
7. Quick-reference checklist

---

## 1. Profiling tools

Measure before changing. Don't guess which component re-renders.

- **React DevTools Profiler** — records renders, shows which components rendered,
  how long, and *why* (enable "Record why each component rendered"). This is the
  primary tool for render-cost problems.
- **`<Profiler>` API** — programmatic render timing for specific subtrees.
- **Chrome DevTools Performance panel** — main-thread work, long tasks, layout
  thrashing, scripting vs rendering breakdown.
- **Lighthouse / Web Vitals** — LCP, INP, CLS, TTFB. Optimize the metric the user
  actually cares about; a faster render that doesn't move a Web Vital may not matter.
- **Bundle analyzers** — `@next/bundle-analyzer` / `webpack-bundle-analyzer` /
  `vite-bundle-visualizer` to see what's actually shipping and find the bloat.
- **Next.js build output** — per-route JS size and rendering strategy (Static /
  SSG / ISR / Dynamic) is printed on build; read it.

### Showing the numbers (React is trickier — match metric to bottleneck)

Rendering doesn't micro-benchmark cleanly, so don't force a BenchmarkDotNet-style
harness onto it. Prove the win with the metric that matches the bottleneck, and show
the actual figure before vs after:

- **Expensive pure logic** you moved or memoized (sort/filter/transform): benchmark
  the *function* in isolation with `vitest bench` / `tinybench` (or `console.time`
  over N iterations) — verify same output first.
- **Bundle size** (code-splitting, dependency swaps): diff the analyzer or Next build
  output. "Route JS 248 kB → 96 kB" is concrete, easy proof.
- **Render cost** (re-render storms, memoization): React DevTools Profiler before vs
  after — compare commit count and render duration for the subtree — or a
  programmatic `<Profiler onRender={cb}>` logging the actual ms.
- **User-facing latency**: Lighthouse / Web Vitals (LCP, INP, TTFB) before vs after —
  the number that actually matters to the user.

Paste the real before/after figure for whichever applies, and don't claim a render
speedup you didn't actually see move in the Profiler.

---

## 2. Where React/Next time actually goes

- **Unnecessary re-renders** — a parent re-renders and cascades through children
  that didn't need to update; new object/array/function props break referential
  equality and defeat memo; context changes re-render every consumer.
- **Expensive work on every render** — heavy computation in the render body,
  re-creating large data structures, re-sorting/filtering big lists each render.
- **Large/eager bundles** — shipping everything up front, heavy libraries pulled
  into the initial bundle, no code splitting.
- **Data-fetching waterfalls** — sequential awaits that should be parallel; a
  component fetches, then its child fetches, then its child fetches.
- **Long lists** — rendering thousands of DOM nodes instead of virtualizing.
- **Hydration cost** — large client trees hydrating on load (Next.js).

---

## 3. Render optimization (and the over-memoization trap)

First, **confirm renders are actually the bottleneck** with the Profiler. A
component re-rendering is cheap unless its render does expensive work or the tree
is huge. Don't optimize renders that aren't slow.

When they are slow:

- **`React.memo`** to skip re-rendering a component when its props are unchanged —
  effective only if the props are referentially stable (see below).
- **`useMemo`** to cache an expensive computation across renders.
- **`useCallback`** to keep a function reference stable so memoized children / effect
  deps don't see it as "new" every render.
- **Stabilize prop references.** The usual reason `memo` "doesn't work" is a parent
  passing a fresh `{}`, `[]`, or `() => {}` every render. Hoist constants, memoize
  derived objects, or restructure so stable references flow down.
- **Lift state down / split components** so a frequently-changing piece of state
  doesn't re-render an expensive subtree. Often this beats memoization entirely.
- **Context:** split contexts by update frequency, or memoize the provider `value`,
  so a change to one field doesn't re-render every consumer.
- **Virtualize long lists** (`@tanstack/react-virtual`, `react-window`) — render only
  visible rows.

### The over-memoization trap
`useMemo`/`useCallback`/`memo` are **not free** and are not a default to apply
everywhere:
- Each one costs memory and a dependency comparison every render. Wrapping cheap
  computations adds overhead without benefit and clutters the code.
- More importantly, **incorrect dependency arrays turn memoization into a
  correctness bug** (stale closures — see traps). Every `useMemo`/`useCallback` you
  add is a dependency array you must get exactly right.
- Memoize what's *measurably* expensive or what *needs* a stable reference for a
  downstream `memo`/effect. Don't memoize on reflex.

(React Compiler, where adopted, automates much of this — if the project uses it,
prefer letting it handle memoization over hand-wrapping, and don't fight it.)

---

## 4. Bundle size & loading

- **Code splitting** — `next/dynamic` or `React.lazy` + `Suspense` to defer
  components that aren't needed on first paint (modals, heavy editors, below-the-fold
  widgets, route-level chunks).
- **Tree shaking** — import only what you use (`import { x } from 'lib'`, not the
  whole namespace); prefer libraries that are ESM and side-effect-free. Check the
  bundle analyzer for accidental full-library imports.
- **Replace heavy dependencies** — a multi-hundred-KB date/util/chart library pulled
  in for one function is a common, large, easy win. Check what each dependency costs.
- **`next/image`** — automatic resizing, modern formats, lazy loading; almost always
  a win for LCP. Set correct `sizes`/dimensions to avoid layout shift.
- **`next/font`** — self-host and preload fonts; eliminates a render-blocking
  request and font-swap layout shift.
- **Defer/lazy non-critical client JS** — analytics, chat widgets, anything not
  needed for first interaction.

---

## 5. Next.js: Server Components, data fetching, caching

### Server vs Client Components (App Router)
- **Default to Server Components.** They ship zero JS to the client — moving a
  component server-side removes it from the bundle entirely. Only mark `"use client"`
  where you need interactivity (state, effects, browser APIs, event handlers).
- Push `"use client"` boundaries **down the tree** — make small interactive leaves
  clients, not whole pages, so server-rendered content stays out of the bundle.

### Kill data-fetching waterfalls
- **Parallelize independent fetches** — `Promise.all` / start requests concurrently
  rather than sequential `await`s. Sequential awaits of independent data are the most
  common Next.js latency bug.
- **Stream with Suspense** — let the shell render and stream slower data in, instead
  of blocking the whole page on the slowest query.
- **Fetch where the data is used** and dedupe — Next dedupes identical `fetch`es in a
  render pass; rely on that rather than prop-drilling fetched data.

### Caching (know your version)
Caching semantics differ significantly between Next.js versions (notably the
defaults changed around v15, and `"use cache"` is newer). **Check the project's
Next.js version and the current docs before relying on caching behavior** — getting
this wrong produces either stale data (a correctness bug) or no caching (a perf
miss). Levels to be aware of: the `fetch`/data cache, the full route cache, the
router cache, and explicit revalidation (`revalidatePath`/`revalidateTag`/time-based).
Choose Static vs ISR vs Dynamic per route deliberately based on how fresh the data
must be.

---

## 6. React/Next behavior-preservation traps

Beyond the universal checklist, in React/Next specifically:

- **Incomplete dependency arrays = stale closures (the big one).** Removing a
  dependency from `useMemo`/`useCallback`/`useEffect` to "reduce re-renders" makes the
  closure capture stale values — the function silently operates on old state/props.
  This is a real, hard-to-spot bug, not an optimization. Keep deps complete; respect
  `react-hooks/exhaustive-deps`. If a value shouldn't trigger re-runs, fix it
  structurally (ref, reducer, functional update), don't just delete it from the array.
- **`key` correctness.** Using array index as `key` for a reorderable/filterable list
  breaks React's reconciliation — component state attaches to the wrong item, inputs
  show wrong values, animations glitch. Use a stable unique id. A perf-motivated key
  change can introduce subtle state-bleed bugs.
- **Hydration mismatches (Next.js/SSR).** A change that makes server and client render
  different HTML (using `window`/`Date.now()`/`Math.random()` during render, locale
  differences, conditionals on client-only state) causes hydration errors and visual
  flicker. A perf "fix" that moves logic across the server/client boundary can trigger
  this — verify the page still hydrates cleanly.
- **`memo` with unstable props is a no-op, not a fix.** Wrapping a component in `memo`
  while still passing fresh objects/functions every render adds a comparison cost and
  changes nothing. Don't claim a speedup you didn't measure.
- **Don't break Rules of Hooks** when refactoring for performance — no conditional
  hooks, no hooks in loops/callbacks.
- **`useTransition`/`useDeferredValue` change *when* updates apply,** not just speed.
  Deferring an update that needs to be synchronous (e.g. controlled input) can make
  the UI feel laggy or behave wrong. Use them for genuinely non-urgent updates.
- **Effect timing.** Moving work between render, `useEffect`, and `useLayoutEffect`
  changes when it runs relative to paint — can introduce flicker or change observable
  ordering of side effects.

---

## 7. Quick-reference checklist

- [ ] Profiled with React DevTools / Lighthouse; identified the real cost (renders?
  bundle? waterfall? list size?).
- [ ] Baseline captured (render time, bundle bytes, or the relevant Web Vital).
- [ ] Memoization added only where measured-expensive or needed for stable refs —
  and every dependency array is complete and correct.
- [ ] Moved work server-side / split client boundaries where it removes bundle JS
  (Next.js).
- [ ] Independent fetches parallelized; no new waterfall.
- [ ] Caching choice matches required data freshness for this Next.js version.
- [ ] Page still hydrates cleanly; list keys stable; no stale closures introduced.
- [ ] Re-measured: the metric actually improved. Reverted memoization that didn't pay.
