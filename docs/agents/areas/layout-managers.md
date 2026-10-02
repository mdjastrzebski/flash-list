<!-- index-codebase: generated from 527d7678 on 2026-10-02. Code wins when it disagrees with this doc. -->
# Layout managers

**Purpose:** Decide where every item goes. The layout manager stores one `RVLayout` (`x`, `y`, `width`, `height`, plus measured flags and min-size constraints) per data index, estimates sizes for items that have not been measured yet, merges real measurements, recomputes positions, reports the content size, and answers "which indices fall inside this range of offsets" by binary search.
It does **not** measure views. It does not decide what gets rendered or recycled ([recycler-core](recycler-core.md)), does not scroll or correct offsets ([scrolling](scrolling.md)), and does not position sticky headers ([sticky-headers](sticky-headers.md)).

**Paths:** `src/recyclerview/layout-managers/` (`LayoutManager.ts`, `LinearLayoutManager.ts`, `GridLayoutManager.ts`, `MasonryLayoutManager.ts`), `src/utils/AverageWindow.ts`, `src/recyclerview/utils/findVisibleIndex.ts`, `src/recyclerview/utils/measureLayout.ts`. Selection and wiring: `RecyclerViewManager.getLayoutManagerClass` / `updateLayoutParams` / `modifyChildrenLayout` in `src/recyclerview/RecyclerViewManager.ts`.

**Depends on:** nothing outside utils. · **Used by:** [recycler-core](recycler-core.md) (RecyclerViewManager, EngagedIndicesTracker, ViewHolder styles), [scrolling](scrolling.md) (`scrollToIndex`, initial scroll, offset correction read `getLayout`), [viewability](viewability.md), [sticky-headers](sticky-headers.md), [public-api](public-api.md) (`getLayout`, `getChildContainerDimensions` on the ref).

## How it works

### Which manager is used
`RecyclerViewManager.getLayoutManagerClass` picks the class from props:

| Props | Class |
|---|---|
| `masonry` (any `numColumns`) | `RVMasonryLayoutManagerImpl` |
| `numColumns > 1`, not horizontal | `RVGridLayoutManagerImpl` |
| anything else (incl. `horizontal`) | `RVLinearLayoutManagerImpl` |

`masonry` + `horizontal` and `numColumns > 1` + `horizontal` throw (`ErrorMessages.masonryAndHorizontalIncompatible`, `numColumnsAndHorizontalIncompatible`). Toggling `horizontal` on a mounted list throws `horizontalPropCannotBeToggled` (remount with a `key` instead).

`RecyclerViewManager.updateLayoutParams` runs on **every render** (a dependency-less `useLayoutEffect` in `RecyclerView.tsx`). If the current instance is not of the required class (first render, `numColumns` crossing 1, `masonry` toggled), it constructs a new one and passes the old one as `previousLayoutManager`: the **`layouts` array is reused** (same array object) but the average windows, span tracker and per-class state start fresh. Otherwise it calls `layoutManager.updateLayoutParams(params)`. `optimizeItemArrangement` defaults to `true` here.

### Window size, bounded size and coordinates
`RecyclerView.tsx` measures two things synchronously in a layout effect (new architecture only, `measureLayout` in `src/recyclerview/utils/measureLayout.ts`):
- `measureParentSize(internalViewRef)`: the outer container.
- `measureFirstChildLayout(firstChildViewRef)`: `viewToMeasureBoundedSize`, a zero-height (vertical) or zero-width (horizontal) view placed right after the header inside the ScrollView.

The window passed to the layout manager is `{ width: firstChild.width, height: outer.height }` for vertical lists and `{ width: outer.width, height: firstChild.height }` for horizontal ones. The cross-axis value is the **bounded size** (`boundedSize` in each subclass), so it already excludes `contentContainerStyle` padding. The first child's offset becomes `RecyclerViewManager.firstItemOffset` (header height). **Layout coordinates are relative to the item container, not the scroll content**: callers add or subtract `firstItemOffset`.

### End-to-end flow: data change → measure → reposition → visible indices
1. **Data prop changes.** `useRecyclerViewManager` runs `recyclerViewManager.processDataUpdate()` in a `useMemo([data])`, before render. That calls `modifyChildrenLayout([], data.length)` → `RVLayoutManager.modifyLayout([], n)`.
2. **`modifyLayout` resizes the array.** If it shrank, `layouts` and `spanTracker` are truncated, the minimum recompute index becomes `n - 1`, and stale `layoutInfo` entries with `index >= n` are filtered out (#2088). If it grew, every new index gets `getLayout(i)` (creates an estimated layout) and `getSpan(i)`, then `recomputeLayouts(oldLength, n - 1)` runs for the whole tail.
3. **Render.** `ViewHolderCollection` spreads `getLayout(index)` into each `ViewHolder`. `ViewHolder` turns it into absolute `left`/`top`, a `width` only if `enforcedWidth`, and `minHeight` (used for row equalization). The container gets `width`/`height` from `getLayoutSize()`.
4. **Measure.** The second layout effect in `RecyclerView.tsx` (skipped while nested child lists are pending) calls `measureItemLayout` on every ViewHolder in `refHolder`. Values are rounded with `PixelRatio.roundToNearestPixel`. If a value is within one physical pixel of the stored layout (`areDimensionsEqual`), the stored value is kept, so float jitter does not trigger relayouts.
5. **`modifyChildrenLayout(layoutInfo, n)` → `modifyLayout`** takes the minimum of these:
   - `computeEstimatesAndMinMaxChangedLayout`: indices that were never measured, whose size differs, or that are at or after `lastSkippedLayoutIndex`. It also feeds every measurement into the per-type `heightAverageWindow` / `widthAverageWindow`, and tracks `currentMaxIndexWithChangedLayout`.
   - `lastSkippedLayoutIndex` (see "Bounded relayout" below).
   - `computeMinIndexWithChangedSpan`: calls `overrideItemLayout` for each measured index and compares the result with `spanTracker`. On a change it calls `handleSpanChange` (Grid and Masonry set `fullRelayoutRequired`).
   - `processLayoutInfo` (subclass): writes the measured sizes and sets `isWidthMeasured` / `isHeightMeasured`. Grid and Masonry return `0` after `updateAllWidths()` when `fullRelayoutRequired` is set.
6. **Recompute** `_recomputeLayouts(min, getMaxRecomputeIndex(min))`, where the max is `max(min, currentMaxIndexWithChangedLayout) + maxItemsToProcess`, clamped to the last index. The subclass `recomputeLayouts` walks the range, positioning each item from the previous one.
7. **Repaint decision.** If the manager set `requiresRepaint`, `modifyChildrenLayout` clears it and returns `true`. `RecyclerView` then calls `setRenderId`, which re-renders and re-measures before paint, capped by `RenderTimeTracker.maxRendersWithoutCommit` (40, which logs `exceededMaxRendersWithoutCommit`). Otherwise engaged indices are recomputed ([recycler-core](recycler-core.md)) and the layout is committed.
8. **Visible and engaged indices.** `EngagedIndicesTracker.updateScrollOffset` / `computeVisibleIndices` call `getVisibleLayouts(start, end)`, which runs two binary searches (`findFirstVisibleIndex`, `findLastVisibleIndex` in `findVisibleIndex.ts`) over `layouts` on `y` (or `x` when horizontal) and returns a `ConsecutiveNumbers` range. It also reads `getLayoutSize()` to clamp the buffer at the end of the list.

Size changes that originate inside an item (`onLayout` → `RecyclerView.validateItemSize`, `useLayoutState`, `useRecyclingState`) call `recyclerViewContext.layout()`. That bumps state, and steps 4–8 run again.

### Estimates for unmeasured items
- `MultiTypeAverageWindow(5, 200)`: a running average of the last 5 measurements **per `getItemType` value**, separately for width and height. Before any item of a type has been measured, the estimate is **200**. There is no `estimatedItemSize` prop in v2 (see `CLAUDE.md`).
- `RVLayoutManager.getLayout(index)` **re-estimates on every call** while either measured flag is false. Reading a layout therefore has side effects: unmeasured items follow the latest average whenever they are read or recomputed. This is why `applyInitialScrollAdjustment` recomputes `0..initialScrollIndex` *before* reading the target offset (348c9ea5, #2133, fixes #1797).
- `AverageWindow.addValue` clamps the average at `>= 0` because float error could drift it negative and compound (5efb1fcc, #2053).

### Linear (`RVLinearLayoutManagerImpl`)
- Vertical: `width = boundedSize`, and `isWidthMeasured` / `enforcedWidth` are `true` even in the estimate. Each `y` is the previous `y + height`. Content height is the last item's `y + height`.
- Horizontal: both dimensions are estimated, then measured. Each `x` is the previous `x + width`. How the list's own height is determined depends on `hasSize`, which is **computed once in the constructor** as `windowSize.height > 0`:
  - **Bounded height** (the parent gave the list a height): `recomputeLayouts` sets `minHeight = boundedSize` on every item, so items stretch to the list's height, and the content height is `boundedSize`.
  - **Unbounded height** (sized by content): `normalizeLayoutHeights` looks for the tallest item in the measured batch, counting only items whose `height > minHeight` (natural height, not stretched). If it is taller than the recorded `tallestItemHeight`, every layout gets that `height` and `minHeight` and the content height becomes the tallest item's height. If the tallest item shrank, every `minHeight` is cleared and `requiresRepaint` is set so that items are re-measured at their natural heights. Height never shrinks just because the tallest item scrolled away. This replaces v1's extra measuring item (`documentation/docs/v2-changes.md`).
- A change in `ViewHolderCollection`'s cross-axis size triggers `layout()` on the parent context, so a nested horizontal list's height reaches the outer list. With many nested lists, `unmarkChildLayoutAsPending` runs as soon as a height exists, to avoid "Maximum update depth exceeded" (40fb21db, #1899).

### Grid (`RVGridLayoutManagerImpl`): rows of equal height
- Item width is `boundedSize / maxColumns * span`, and is always enforced (`enforcedWidth`). Only height is measured.
- `recomputeLayouts(start, end)` steps back to `locateFirstIndexInRow(start - 1)`, which scans backwards for `x === 0`. It then places items left to right and wraps when `x + width > boundedSize + 0.9` (`checkBounds` allows 0.9 px of float tolerance). On every wrap, `processAndReturnTallestItemInRow(i - 1)` equalizes the row that just ended, and the next row starts at that row's `tallest.y + tallest.height`. The final row is equalized only when `end` is the last index.
- Row equalization (`processAndReturnTallestItemInRow`): the "tallest" item is the tallest one whose `height > minHeight`, i.e. its natural height rather than a stretched one. All items in the row get `minHeight = height = tallest.height`, and the tallest item's own `minHeight` is reset to 0. If some item is more than 1 px taller than that natural tallest, the row's tallest item has shrunk. The row is then reset to `minHeight = 0` with `requiresRepaint = true` and re-measured. If every item is at its `minHeight`, `maxHeight` is set to `MAX_SAFE_INTEGER` to force that reset (be88e939). Rows are skipped while none of their items is measured.
- `getLayoutSize().height` = the last row's `y` + the tallest height in that row (`computeTotalHeightTillRow`).
- `isInLastRow` (Grid only, compares `y`) lets `ViewHolderCollection` drop `ItemSeparatorComponent` for the whole last row. Without it, the measured separator height of the other items would stretch the last item through equalization (8fdd9e86, #2114, fixes #1868).

### Masonry (`RVMasonryLayoutManagerImpl`): column assignment
- Width uses the same formula as Grid. Height is measured, with no row equalization. `columnHeights[]` tracks the bottom of each column, and `getLayoutSize().height = max(columnHeights)`.
- `optimizeItemArrangement` (default `true`):
  - `span === 1`: the item goes into the shortest column (`placeSingleColumnItem`, ties go to the leftmost column).
  - `span > 1`: every start column is tried, and the one that minimizes the sum of column heights wins (`placeOptimizedMultiColumnItem`).
  - `false`: items are placed sequentially, cycling through columns via `currentColumn` (`placeItemSequentially`). An item that does not fit in the remaining columns wraps to column 0.
- `recomputeLayouts(start, …)` **ignores `endIndex` and always runs to the end of the list**. When `start > 0`, `updateColumnHeightsToIndex` first rebuilds `columnHeights` from items `0..start-1`, inferring their columns from `x` and `width`. Every recompute is therefore O(n).
- Because columns are chosen by measured height, a height change in one item can move later items to different columns. `FlashListProps.optimizeItemArrangement` has a TODO about resize glitchiness.

## Key types and entry points
| Symbol | File | Role |
|---|---|---|
| `RVLayoutManager` | `layout-managers/LayoutManager.ts` | Abstract base: `layouts[]`, estimates, `modifyLayout`, `getVisibleLayouts`, bounded relayout, span tracking |
| `RVLinearLayoutManagerImpl` | `layout-managers/LinearLayoutManager.ts` | Single column or row; horizontal height normalization |
| `RVGridLayoutManagerImpl` | `layout-managers/GridLayoutManager.ts` | Wrap by width; equal-height rows; `isInLastRow` |
| `RVMasonryLayoutManagerImpl` | `layout-managers/MasonryLayoutManager.ts` | Column packing with `columnHeights` |
| `RVLayout`, `RVDimension`, `RVLayoutInfo`, `LayoutParams`, `SpanSizeInfo` | `layout-managers/LayoutManager.ts` | Data shapes; `RVLayoutInfo` = `{index, dimensions}` measured this pass |
| `modifyLayout` / `processLayoutInfo` | `LayoutManager.ts` / subclasses | Merge measurements, pick the recompute start |
| `recomputeLayouts` / `_recomputeLayouts` | subclasses / base | Positioning; base wrapper handles skipped tails |
| `estimateLayout`, `getEstimatedWidth/Height` | subclasses / base | Sizes for unmeasured items |
| `getVisibleLayouts` | `LayoutManager.ts` → `utils/findVisibleIndex.ts` | Binary search for an offset range |
| `requiresRepaint` | `LayoutManager.ts` | Flag consumed by `RecyclerViewManager.modifyChildrenLayout` |
| `MultiTypeAverageWindow`, `AverageWindow` | `src/utils/AverageWindow.ts` | Per-type running averages |
| `getLayoutManagerClass`, `updateLayoutParams`, `modifyChildrenLayout`, `markLayoutManagerDirty` | `RecyclerViewManager.ts` | Selection, construction, the measure → repaint loop, cache reset |
| `measureItemLayout`, `measureFirstChildLayout`, `areDimensionsEqual` | `utils/measureLayout.ts` | Synchronous measurement and the 1 px tolerance |

## Invariants and gotchas
- **Layouts are keyed by index, not by data key.** `processDataUpdate` only truncates or extends. After a prepend or reorder, `layouts[i]` still holds the old item *i*'s size, marked measured. Rendered items are corrected on the next measure pass, but off-screen items stay wrong until they render. `deleteLayout(indices)` exists but **nothing calls it** at this commit. Content-position stability across prepends belongs to [scrolling](scrolling.md) (MVCP).
- **`layouts` must stay monotonic along the scroll axis** for `getVisibleLayouts`, because the binary search assumes it. Linear and Grid satisfy this; Masonry does not strictly (column `y` values interleave). For masonry, `computeItemViewability` uses the engaged (buffered) indices instead of visible ones, with the comment "Using higher buffer for masonry to avoid missing items".
- **`isWidthMeasured` means "width known", not "measured".** Vertical Linear, Grid and Masonry set it to `true` in `estimateLayout` because width is enforced. Only `isHeightMeasured` tells you a real measurement happened. `renderProgressively` waits until both flags are true for every visible item.
- **`getLayout` throws** past `getLayoutCount()` (`indexOutOfBounds`). Use `RecyclerViewManager.tryGetLayout` when an index may be stale. `getLayout` also mutates unmeasured layouts (re-estimation).
- **Bounded relayout:** one pass recomputes at most `maxItemsToProcess` items (starts at 250, grows to `10 × measured batch size`, never shrinks) beyond the furthest changed index. The first index not recomputed is stored in `lastSkippedLayoutIndex` and picked up by the next `modifyLayout`. If the stale tail would end up above `endIndex` (`layouts[last].y < layouts[end].y`), the whole tail is recomputed right away, because content size is derived from the last items (750c5328). Data growth (`modifyLayout` growth branch) and `updateLayoutParams` bypass the cap and recompute everything.
- **Span changes are detected only for indices in the current `layoutInfo`**, i.e. rendered items. Changing `overrideItemLayout` output (e.g. via `extraData`) for off-screen measured items has no effect until one of them renders, and at that point `updateAllWidths()` fixes every width. `span` is clamped to `maxColumns` but not from below. Only `span` is read: `SpanSizeInfo.size` is ignored in v2 (`documentation/docs/v2-migration.md`). For Linear, `maxColumns` is 1, so span is always 1.
- `overrideItemLayout` receives the **same mutable `spanSizeInfo` object** on every call (`span` is reset to `undefined` first). It runs very often (`estimateLayout`, span checks for every measured item on every pass, every item in a Masonry recompute), so keep it cheap and pure.
- **The base constructor calls the virtual `updateLayoutParams`** when `previousLayoutManager` is passed, before the subclass's field initializers and constructor body run. The subclass sees `boundedSize === undefined`, recomputes everything and sets `requiresRepaint`. Any subclass field with an initializer is then reset after `super()`. Keep this in mind when adding state.
- `hasSize` (Linear) is never updated after construction. **Inferred:** this is deliberate, because in content-sized mode the measured height follows the content and would feed back into itself. A horizontal list that later receives a fixed height keeps normalizing by content until the manager is recreated.
- `enforcedHeight`, `minWidth`, `maxWidth` and `maxHeight` are honored by `ViewHolder` / `validateItemSize`, but no built-in manager sets them.
- `computeEstimatesAndMinMaxChangedLayout` is called **twice** per `modifyLayout`, so every measurement enters the 5-slot average window twice. **Inferred:** a leftover from the 2973c183 refactor; it biases estimates toward the most recent batch.
- `markLayoutManagerDirty()` (ref method `clearLayoutCacheOnUpdate`) drops the instance. The next `updateLayoutParams` builds a fresh manager with no previous layouts and no averages.
- If `yarn test` fails with "package doesn't seem to be present in your lockfile", corepack picked Yarn 4 while the repo has a Yarn v1 lockfile. Run `node_modules/.bin/jest LayoutManager --forceExit` instead.

## Design decisions
- **No size estimates from the user. A per-type rolling average, then synchronous measurement** (`documentation/docs/v2-changes.md`, `v2-migration.md`). This relies on the new architecture's synchronous `measureLayout` in `useLayoutEffect`, so corrections happen before paint, which is also why v2 is new-arch only. `getItemType` doubles as the estimate bucket, so a good `getItemType` improves estimates as well as recycling.
- **Equalize rows with `minHeight`, not a fixed `height`.** Items can still grow (a new natural tallest), and a shrink is detected because the shrunken item's height falls back to its `minHeight` (be88e939). The cost is an extra repaint whenever a row's tallest item shrinks.
- **Bounded relayout window + `lastSkippedLayoutIndex`** keeps per-measure work O(rendered × 10) instead of O(n) on long lists, while keeping the content size roughly right (750c5328 "Fix incorrect total height due to staggered layout compute", which replaced the earlier `repositionPending` flag).
- **Full relayout on span change** (2973c183): when one width changes, every later row can re-wrap, so Grid and Masonry recompute all widths from index 0 instead of tracking the change incrementally (`// TODO: Can be optimized`).
- **Measurement tolerance of 1 physical pixel** (`areDimensionsEqual`) and **0.9 px row-fit slack** (`checkBounds`) prevent float-rounding relayout loops and spurious row wraps when `boundedSize / maxColumns` is fractional.
- **Grid separators**: the last row is decided by `y`, not by `index >= n - numColumns`, so it stays correct with variable spans (#2114).

## Where to change things
| To… | Start at |
|---|---|
| Change how an existing list type positions items | `recomputeLayouts` in the subclass; for Grid, also `processAndReturnTallestItemInRow` |
| Change sizes for unmeasured items | `estimateLayout` (subclass), `getEstimatedWidth/Height`, `MultiTypeAverageWindow(5, 200)` in the `RVLayoutManager` constructor |
| Change what counts as a "changed" measurement | `computeEstimatesAndMinMaxChangedLayout` (base) and `areDimensionsEqual` in `utils/measureLayout.ts` |
| Add a layout type (e.g. horizontal grid) | Subclass `RVLayoutManager` (implement `processLayoutInfo`, `estimateLayout`, `recomputeLayouts`, `getLayoutSize`; optionally `updateLayoutParams`, `handleSpanChange`, `isInLastRow`). Register it in `RecyclerViewManager.getLayoutManagerClass` and add a prop in `FlashListProps.ts`. Not public: no layout manager is exported from `src/index.ts`, so users cannot plug in their own |
| Support another `overrideItemLayout` field | `SpanSizeInfo` + `getSpan` in `LayoutManager.ts`, the `overrideItemLayout` wrapper in `RecyclerViewManager.ts`, and the prop type in `FlashListProps.ts` |
| Fix a wrong content size or blank tail | Subclass `getLayoutSize`, `_recomputeLayouts` / `lastSkippedLayoutIndex` |
| Fix items missing from the viewport | `getVisibleLayouts`, `utils/findVisibleIndex.ts`, monotonicity of the subclass output |
| Change separator suppression | `isInLastRow` (Grid; Masonry has a TODO) and `ViewHolderCollection.tsx` |
| Change how window or bounded size is measured | The first `useLayoutEffect` and `viewToMeasureBoundedSize` in `RecyclerView.tsx` |

## Tests
- `src/__tests__/LinearLayoutManager.test.ts`, `GridLayoutManager.test.ts`, `MasonryLayoutManager.test.ts` (26 tests, all passing at 527d7678) cover basic placement, add, remove, resize window, empty lists, and stale `layoutInfo` after truncation (#2088). They drive managers directly through `src/__tests__/helpers/createLayoutManager.ts` (`createLayoutManager`, `createPopulatedLayoutManager`, `populateLayouts`, `getAllLayouts`; default window 400×900).
- Related: `AverageWindow.test.ts`, `findVisibleIndex.test.ts`, `EngagedIndicesTracker.test.ts`. `RecyclerView.test.tsx` renders the Grid and Masonry paths and covers the grid-separator fix.
- **Not covered:** grid row equalization and its `requiresRepaint` shrink path, horizontal height normalization (`hasSize` false), bounded relayout and `lastSkippedLayoutIndex`, span-change relayout, `optimizeItemArrangement: false`, switching manager classes with `previousLayoutManager`, `deleteLayout`.
- Run: `yarn test LayoutManager --forceExit` (or `node_modules/.bin/jest LayoutManager --forceExit`; see the gotcha above). Behavior that depends on real measurement (repaints, nested horizontal lists) needs the fixture app. See the `review-and-test` skill.
