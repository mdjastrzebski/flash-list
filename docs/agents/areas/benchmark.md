<!-- index-codebase: generated from 527d7678 on 2026-10-02. Code wins when it disagrees with this doc. -->
# Benchmark

**Purpose:** Public, in-app performance helpers. They auto-scroll a FlashList (or a FlatList, for comparison) end to end while sampling JS-thread FPS, then report min, avg and max FPS plus simple suggestions. They do **not** measure UI-thread FPS, blank area, memory or render counts. Blank-area tracking (`onBlankArea`) was a v1 native feature and is gone in v2.

**Paths:** `src/benchmark/` (`useBenchmark.ts`, `useFlatListBenchmark.ts`, `useDataMultiplier.ts`, `JSFPSMonitor.ts`, `AutoScrollHelper.ts`, `roundToDecimalPlaces.ts`), `documentation/docs/guides/list-profiling.md`

**Depends on:** [public-api](public-api.md) (`FlashListRef`, `ErrorMessages`) · **Used by:** app developers, `fixture/react-native`

## How it works

1. `useBenchmark(ref, callback, params)` schedules `startBenchmark` after `startDelayInMs` (default 3000) unless `startManually` is set. The hook returns `{ startBenchmark, isBenchmarkRunning }`.
2. `startBenchmark` throws `ErrorMessages.dataEmptyCannotRunBenchmark` if `ref.current.props.data` is empty. It then creates a `JSFPSMonitor` and calls `startTracking()`.
3. Each of `repeatCount` iterations (default 1) runs `runScrollBenchmark`:
   - It reads `getWindowSize()` and `getChildContainerDimensions()` from the ref.
   - It calls `autoScroll` from 0 to `content - window`, then back.
   - Every step is `ref.scrollToOffset({offset, animated:false})`.
4. `autoScroll` loops on `requestAnimationFrame` and advances `7px/ms × speedMultiplier` scaled by elapsed time. The comment says that is about a "very fast scroll on Android/iOS". It resolves `false` if the `Cancellable` was cancelled.
5. `JSFPSMonitor` counts rAF callbacks:
   - **average** = total frames ÷ total seconds;
   - **min/max** = taken over 1-second windows;
   - if the run lasted under 1s, min = max = average;
   - values are rounded to 1 decimal.
6. Suggestions: average FPS < 35 gives "too much work". `data.length < 200` gives "use `useDataMultiplier`".
7. `callback(result)` gets `{ js, suggestions, interrupted, formattedString? }`. `formattedString` (from `getFormattedString`) is set only when the run was not cancelled.

`useFlatListBenchmark` follows the same flow against a `FlatList` ref. It needs a mandatory `targetOffset`, because FlatList exposes no content size, and it produces no suggestions. `useDataMultiplier(data, count)` returns `[arr]`, with `data` repeated cyclically to `count` items and objects shallow-cloned.

## Key types and entry points

All of these are exported from `src/index.ts`.

| Symbol | File | Role |
|---|---|---|
| `useBenchmark`, `BenchmarkParams`, `BenchmarkResult` | `useBenchmark.ts` | FlashList benchmark hook |
| `useFlatListBenchmark`, `FlatListBenchmarkParams` | `useFlatListBenchmark.ts` | Same flow for FlatList (adds `targetOffset`) |
| `useDataMultiplier` | `useDataMultiplier.ts` | Inflates a dataset for meaningful runs |
| `JSFPSMonitor`, `JSFPSResult` | `JSFPSMonitor.ts` | rAF-based JS FPS sampler. Each instance is single-use |
| `autoScroll`, `Cancellable` | `AutoScrollHelper.ts` | Generic timed scroll driver for anything that has a `scroll(x, y)` callback |
| `getFormattedString` | `useBenchmark.ts` | Not exported from the package. Reused by the FlatList hook |

## Invariants and gotchas

- **It measures only the JS thread.** rAF frequency reflects JS-thread availability. UI-thread jank and blank cells are invisible to it.
- **`BenchmarkParams.sumNegativeBlankAreaValues` is dead.** It is declared but read nowhere. It is a leftover of v1 blank-area tracking, which was removed in #1750 (`0d0b66e7`) and `9d1aef13`.
- **`JSFPSMonitor.startTracking()` throws `fpsMonitorAlreadyRunning` on a second call.** Create a new instance per run (the hooks do).
- **`useBenchmark` captures `params.startManually` and `startDelayInMs` once.** Its mount `useEffect` has empty deps (lint-disabled), so later changes to those params are ignored. On unmount it clears the timer and cancels; the in-flight run then calls `callback` with `interrupted: true`.
- **Repeated `startBenchmark` calls are ignored** while `isBenchmarkRunning` is true. Callers should check `result.interrupted` before showing results (all fixture screens do).
- **The empty-data error is thrown inside the `setTimeout` path when the run starts automatically**, so it surfaces as an uncaught error rather than to the caller.
- **`useDataMultiplier` copies `data` without making keys unique.** Its JSDoc says to drop `keyExtractor` with FlatList; with FlashList, duplicated keys can trigger `exceededMaxRendersWithoutCommit` ([public-api](public-api.md)).
- **Benchmark numbers in dev mode are not representative** (`documentation/docs/fundamentals/usage.md`, point 5). Use a release build.

## Design decisions

- **Driven by `scrollToOffset` instead of native flings**, so runs are deterministic and repeatable across devices. The 7px/ms speed is a hard-coded approximation (comment in `AutoScrollHelper.ts`).
- **Shipped in the library, not as a dev tool**, so users can profile their own lists in their own app. It was added as a built-in benchmark in v1 (`c38f210f`, #136) and kept in v2.
- **`startManually`** was added so runs can be triggered by a button instead of a timer (`a303c2f4`, #1757).

## How the fixture app uses it

| Screen (`fixture/react-native/src`) | Hook | Params |
|---|---|---|
| `twitter/TwitterBenchmark.tsx` ("Twitter Benchmark") | `useBenchmark` | auto start, `speedMultiplier: 2`, `repeatCount: 5`, shows the result in `alert` |
| `ManualBenchmarkExample.tsx` | `useBenchmark` | `startManually`, `repeatCount: 3`, `speedMultiplier: 1.5` |
| `ManualFlatListBenchmarkExample.tsx` | `useFlatListBenchmark` | `startManually`, `targetOffset: 100000`, `repeatCount: 2`, `speedMultiplier: 2` |

The screens are registered in `NavigationTree.tsx` and `ExamplesScreen.tsx`. The fixture imports `@shopify/flash-list` via `link:../../` (that is, `dist/`), so run `yarn build` first. CI does not run benchmarks.

## Where to change things

| To… | Start at |
|---|---|
| Add a metric or suggestion | `runBenchmark`/`computeSuggestions` in `useBenchmark.ts`, `BenchmarkResult`, `getFormattedString` |
| Change scroll speed or path | `autoScroll` in `AutoScrollHelper.ts`, `runScrollBenchmark` in both hooks |
| Change FPS sampling | `JSFPSMonitor.updateLoopCompute` |
| Update the user docs | `documentation/docs/guides/list-profiling.md` |

## Tests

None. `src/__tests__` has no benchmark tests. You can only verify changes by running the fixture screens above on a device in release mode.
