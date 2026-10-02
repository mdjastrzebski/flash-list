<!-- index-codebase: generated from 527d7678 on 2026-10-02. Code wins when it disagrees with this doc. -->
# Viewability

**Purpose:** Decides which items count as "viewable" under a `ViewabilityConfig` and calls `onViewableItemsChanged` (plus every `viewabilityConfigCallbackPairs` entry) with `viewableItems` and `changed` `ViewToken`s. It does **not** decide what gets rendered. The render window ("engaged indices") and visible-index math belong to [recycler-core](recycler-core.md) and [layout-managers](layout-managers.md). Viewability only filters the candidate indices it is handed.

**Paths:** `src/recyclerview/viewability/` (`ViewabilityManager.ts`, `ViewabilityHelper.ts`, `ViewToken.ts`). Driven from `src/recyclerview/RecyclerViewManager.ts`, `src/recyclerview/RecyclerView.tsx` and `src/recyclerview/hooks/useRecyclerViewController.tsx`.

**Depends on:** [recycler-core](recycler-core.md) (scroll offset, `computeVisibleIndices`), [layout-managers](layout-managers.md) (`getLayout`, window size) · **Used by:** [public-api](public-api.md) (props and the `recordInteraction` / `recomputeViewableItems` ref methods)

## How it works

There are three layers:

- `RecyclerViewManager` owns one `ViewabilityManager`, created in its constructor as `itemViewabilityManager`.
- `ViewabilityManager` owns one `ViewabilityHelper` per config: one for `onViewableItemsChanged` + `viewabilityConfig`, and one per entry in `viewabilityConfigCallbackPairs`.
- Each `ViewabilityHelper` holds its own state (`possiblyViewableIndices`, `viewableIndices`, `lastReportedViewableIndices`, pending `timers`).

### Flow: scroll → viewability recomputed → `minimumViewTime` timer → callback

1. `RecyclerView.tsx` `onScrollHandler` receives the native scroll event. `velocityTracker.computeVelocity` synchronously calls `recyclerViewManager.updateScrollOffset(...)`, which updates the engaged-indices tracker's `scrollOffset` (see [scrolling](scrolling.md)).
2. If `recyclerViewManager.isInitialScrollComplete` is true, the handler calls `recordInteraction()`. Then it calls `recyclerViewManager.computeItemViewability()`.
3. `RecyclerViewManager.computeItemViewability()` returns early when `shouldListenToVisibleIndices` is false (no callbacks registered). Otherwise it passes candidate indices to `ViewabilityManager.updateViewableItems(indices)`. The candidates are `computeVisibleIndices().toArray()`, or for `masonry` `getEngagedIndices().toArray()`.
4. `ViewabilityManager.updateViewableItems` computes `scrollOffset = getAbsoluteLastScrollOffset() - firstItemOffset`, so offsets are relative to the first item and the `ListHeaderComponent` is excluded. It then fans out to every helper with `horizontal`, `getWindowSize()` and `getLayout`.
5. `ViewabilityHelper.updateViewableItems` does four things:
   - caches the candidates in `possiblyViewableIndices`;
   - throws if both thresholds are set;
   - returns early if `waitForInteraction` is set and the user has not interacted yet;
   - filters the candidates through `isItemViewable`, then stores the result in `viewableIndices`.
6. With `minimumViewTime > 0` (the default is **250**), the helper schedules a `setTimeout` per call and tracks it in `timers`. When the timer fires, `checkViewableIndicesChanges(snapshot)` keeps only the snapshot indices that are **still** in the latest `viewableIndices`. It diffs them against `lastReportedViewableIndices` and, if anything changed, calls back.
7. `ViewabilityManager.createViewabilityHelper`'s `mapViewToken` turns indices into `ViewToken`s (`item` from `props.data`, `key` from `keyExtractor` or `String(index)`, `timestamp: Date.now()`). Then it calls the **current** prop callback.

### Other triggers

| Trigger | Path |
|---|---|
| Every commit of the cell tree | `ViewHolderCollection` `onCommitEffect` → `RecyclerView.tsx` → `computeItemViewability()` (covers data and layout changes without a scroll) |
| `ref.recordInteraction()` | `useRecyclerViewController` → `RecyclerViewManager.recordInteraction` → `ViewabilityManager.recordInteraction` (sets `hasInteracted` once, then re-evaluates the cached candidates) |
| `ref.recomputeViewableItems()` | `RecyclerViewManager.recomputeViewableItems` → `clearLastReportedViewableIndices()` + `computeItemViewability()` |
| Unmount | `useRecyclerViewManager` effect cleanup → `RecyclerViewManager.dispose` → `ViewabilityManager.dispose` clears pending timers |

## Key types and entry points

| Symbol | File | Role |
|---|---|---|
| `ViewabilityManager` | `src/recyclerview/viewability/ViewabilityManager.ts` | Builds helpers from props at construction. Fans out updates and maps indices to `ViewToken`s. |
| `ViewabilityManager.shouldListenToVisibleIndices` | same | `true` if at least one helper exists. Gates all work. |
| `ViewabilityHelper` | `src/recyclerview/viewability/ViewabilityHelper.ts` | Per-config threshold math, `minimumViewTime` timers and diffing |
| `ViewabilityHelper.isItemViewable` | same | Pixel math for `itemVisiblePercentThreshold` / `viewAreaCoveragePercentThreshold` |
| `ViewToken<T>` | `src/recyclerview/viewability/ViewToken.ts` | `{ item, key, index, isViewable, timestamp }` |
| `ViewabilityConfigCallbackPair(s)` | `src/FlashListProps.ts` | Public types for the pairs prop |
| `RecyclerViewManager.computeItemViewability` / `recomputeViewableItems` / `recordInteraction` | `src/recyclerview/RecyclerViewManager.ts` | The only bridge from the rest of the list into viewability |
| `ErrorMessages.multipleViewabilityThresholdTypesNotSupported` | `src/errors/ErrorMessages.ts` | Thrown when both threshold types are set |

## Invariants and gotchas

- **Helpers are fixed at mount.** `ViewabilityManager` is constructed once, inside `RecyclerViewManager`'s constructor, which `useRecyclerViewManager` creates in `useState`. Adding `onViewableItemsChanged` after mount, or adding or removing pairs, has no effect. The `viewabilityConfig` object is captured too ("Changing viewabilityConfig on the fly is not supported" in `FlashListProps.ts`). Only the *callbacks* are read fresh.
- **Pair callbacks are looked up by array index** (`viewabilityConfigCallbackPairs?.[index]`) when they fire. Reordering the pairs array swaps which callback receives which config's results.
- **`minimumViewTime` defaults to 250 ms, unlike RN `FlatList`'s 0.** Both appearing and disappearing are reported only after the delay. Each scroll event schedules its own timer. An index is reported only if it was viewable when its timer was scheduled **and** is in the latest computed `viewableIndices` when it fires.
- **`timestamp` is the callback time** (`Date.now()` inside `mapViewToken`), not the moment the item became visible. So it lags by at least `minimumViewTime`.
- **`item` is read from `props.data` when the callback fires.** After a data change it may be a different item or `undefined`. When it is `undefined`, `key` falls back to `String(index)`.
- **A fully visible item is always viewable**, whatever the threshold. An item with zero visible pixels never is. With no threshold configured, any visible pixel counts (`itemVisiblePercentThreshold ?? 0`).
- **Setting both thresholds throws on every update, not at mount.** The check sits inside `ViewabilityHelper.updateViewableItems`, so the error surfaces on the first viewability computation (first commit).
- **`waitForInteraction`:** `hasInteracted` is set once and never reset. Programmatic initial scrolls (`initialScrollIndex`) do not count, because `onScrollHandler` skips `recordInteraction()` until `isInitialScrollComplete` is set. That flag is flipped in `useRecyclerViewController` (#2130, fixes #1943).
- **`recomputeViewableItems` re-reports everything currently viewable as newly visible** (it clears `lastReportedViewableIndices`) and is still subject to `minimumViewTime`.
- **`getLastScrollOffset` vs `getAbsoluteLastScrollOffset`** (in `RecyclerViewManager`): the first is already relative to the first item; the second adds `firstItemOffset` back. Viewability uses the absolute one and subtracts `firstItemOffset` itself.
- **Inverted lists:** `inverted` is a view transform ([scrolling](scrolling.md)). Offsets and layouts stay in content space, so viewability needs no inverted branch. **Inferred** from the absence of any `inverted` handling in `viewability/`.
- `ViewabilityHelper` runs on every scroll event (see its class doc comment). Keep it allocation-light.

## Design decisions

- **Default `minimumViewTime` of 250 ms.** The code comment in `ViewabilityHelper.updateViewableItems` says: "Default of 0 can impact performance when user scrolls fast."
- **Callbacks are wrapped and read lazily from `rvManager.props`.** This avoids stale closures when the parent passes a new callback each render (commit 26967953, "Fix stale callback in viewability configs").
- **`recomputeViewableItems` recomputes visible indices from scratch** instead of reusing cached ones. Reusing stale indices after a data change caused index-out-of-bounds errors (#2087, commit 4c000e02).
- **Masonry passes engaged indices instead of visible indices as candidates.** The comment in `computeItemViewability` says "Using higher buffer for masonry to avoid missing items". **Inferred:** masonry columns are not contiguous, so the visible-index range can miss items in the shorter columns.
- **Timers are cleared on unmount** so callbacks don't fire after the list is gone (#1159, commit 242ffce9).

## Where to change things

| To… | Start at |
|---|---|
| Change threshold or pixel math | `ViewabilityHelper.isItemViewable` |
| Change timing or debounce semantics | `ViewabilityHelper.updateViewableItems` / `checkViewableIndicesChanges` |
| Add a `ViewToken` field or change key derivation | `ViewabilityManager.createViewabilityHelper` (`mapViewToken`) + `ViewToken.ts` |
| Support changing configs after mount | `ViewabilityManager` constructor. It would need rebuilding from `RecyclerViewManager.updateProps`. |
| Change which indices are candidates | `RecyclerViewManager.computeItemViewability` |
| Add a new trigger point | Call `recyclerViewManager.computeItemViewability()`. The existing callers are `RecyclerView.tsx` `onScrollHandler` and `onCommitEffect`. |
| Change interaction gating | `RecyclerView.tsx` `onScrollHandler` (`isInitialScrollComplete`), `ViewabilityManager.recordInteraction` |

## Tests

- `src/__tests__/ViewabilityHelper.test.ts` is a pure unit test of `ViewabilityHelper`, using fake timers and a local `updateViewableItems` helper. It covers thresholds, horizontal mode, `minimumViewTime`, `waitForInteraction`, and the both-thresholds error.
- `src/__tests__/RecyclerView.test.tsx`, `describe("Viewability with initialScrollIndex")`, covers the `waitForInteraction` + `initialScrollIndex` integration (#2130).
- No unit test covers `ViewabilityManager` directly (pair index lookup, `mapViewToken`, `recomputeViewableItems`).

```bash
yarn test ViewabilityHelper --forceExit
yarn test RecyclerView.test -t "Viewability" --forceExit
# If `yarn` resolves to Yarn 4 via corepack and fails with "not present in your lockfile":
node_modules/.bin/jest ViewabilityHelper --forceExit
```

Manual check on a device: the fixture screens `fixture/react-native/src/Carousel.tsx`, `Masonry.tsx` and `twitter/Twitter.tsx` use viewability callbacks.
