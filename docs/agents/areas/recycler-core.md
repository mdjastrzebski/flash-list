<!-- index-codebase: generated from 527d7678 on 2026-10-02. Code wins when it disagrees with this doc. -->
# Recycler core (render engine)

**Purpose:** Turns `data` + scroll position into a set of mounted, absolutely positioned cells. It decides which indices to render (the "engaged indices"), gives each one a recyclable React key, renders the cells, measures them synchronously before paint, feeds the measurements to the layout manager, and re-renders until the layout settles. It also owns first-load progressive rendering, `onLoad`, `onEndReached`/`onStartReached`, nested-list coordination and the public state hooks (`useLayoutState`, `useRecyclingState`, `useMappingHelper`, `LayoutCommitObserver`).
It does **not** compute positions (see [layout-managers](layout-managers.md)), and it does not handle scroll-to/offset correction/MVCP math ([scrolling](scrolling.md)), viewability filtering ([viewability](viewability.md)), sticky headers ([sticky-headers](sticky-headers.md)) or the prop/ref surface ([public-api](public-api.md)).

**Paths:** `src/recyclerview/RecyclerView.tsx`, `RecyclerViewManager.ts`, `RenderStackManager.ts`, `ViewHolder.tsx`, `ViewHolderCollection.tsx`, `RecyclerViewContextProvider.ts`, `LayoutCommitObserver.tsx`, `RecyclerViewProps.ts` (a re-export of `FlashListProps`), `src/recyclerview/helpers/` (`EngagedIndicesTracker.ts`, `ConsecutiveNumbers.ts`, `VelocityTracker.ts`, `RenderTimeTracker.ts`), `src/recyclerview/hooks/` (`useRecyclerViewManager`, `useSecondaryProps`, `useOnLoad`, `useBoundDetection`, `useLayoutState`, `useRecyclingState`, `useMappingHelper`, `useUnmountAwareCallbacks`, `useUnmountFlag`), `src/recyclerview/utils/measureLayout.ts` + `measureLayout.web.ts`, `utils/componentUtils.ts`, `components/CompatView.ts`, `components/CompatScroller.ts`.

**Depends on:** [layout-managers](layout-managers.md), [scrolling](scrolling.md) (`useRecyclerViewController`, `ScrollAnchor`), [viewability](viewability.md), [sticky-headers](sticky-headers.md), platform config in `src/native/config/PlatformHelper*.ts` ([public-api](public-api.md)) · **Used by:** [public-api](public-api.md) (`src/FlashList.ts` is literally `export { RecyclerView as FlashList }`), [sticky-headers](sticky-headers.md) (reuses `ViewHolder` with `target="StickyHeader"`), [benchmark](benchmark.md).

## How it works

### The four objects

| Object | Lifetime | What it owns |
|---|---|---|
| `RecyclerView` (component, `React.memo(forwardRef(...))`, `displayName = "FlashList"`) | the list | refs, scroll handler, the two measurement `useLayoutEffect`s, context for children |
| `RecyclerViewManager` | one per mount (`useState` in `useRecyclerViewManager`) | the current `RVLayoutManager`, an `RVEngagedIndicesTrackerImpl`, a `RenderStackManager`, a `ViewabilityManager`; progressive-render state; `firstItemOffset` |
| `RenderStackManager` | owned by the manager | the **render stack**: `Map<recycleKey, {index, itemType, stableId}>`, plus per-type recycle pools |
| `ViewHolderCollection` / `ViewHolder` | rendered by `RecyclerView` | one absolutely positioned cell per render-stack entry, `key = recycleKey` |

The manager is a mutable, non-React object. React state in `RecyclerView` (`setRenderId`, `setLayoutTreeId`) is used only as a "please re-render" trigger. All real state lives in the manager.

### Engaged indices (what to render)

`RVEngagedIndicesTrackerImpl.updateScrollOffset(offset, velocity, layoutManager)` in `helpers/EngagedIndicesTracker.ts`:
- **Buffer:** the total buffer is `2 * drawDistance`. 70% goes in the scroll direction and 30% behind (`largeMultiplier` / `smallMultiplier`). Buffer that can't be used at the start of the list moves to the end, and the reverse. The default `drawDistance` comes from `PlatformConfig.defaultDrawDistance`: 250 on native, 500 on web.
- **Direction:** a majority vote over a 5-slot circular `velocityHistory`.
- **Offset projection:** the window is centred on `offset + medianVelocity * averageRenderTime` (velocity is in px/ms). `averageRenderTime` comes from `RenderTimeTracker`, clamped to 16–32 ms. Real tracking is on only where `PlatformConfig.trackAverageRenderTimeForOffsetProjection` is true, which is Android only. Every other platform uses a fixed 16 ms. `useRecyclerViewController` turns projection off during `scrollToIndex` (`setOffsetProjectionEnabled(false)`).
- **Clamp:** the window always covers the real viewport `[offset, offset + viewportSize]`, even when the projection overshoots (commit 9b5e7f8b, #2510).
- The tracker returns a new `ConsecutiveNumbers` only when the range changed, otherwise `undefined`. That return value is the re-render signal.

`computeVisibleIndices` is the same lookup with no buffer and no projection. It drives progressive rendering, viewability and `getFirstVisibleIndex`.

### Render stack (which key renders which index)

`RenderStackManager.sync(getStableId, getItemType, engagedIndices, dataLength)`:
1. Clear every recycle pool (pools only live for one sync).
2. For each existing key: if its index is out of range, no longer engaged, or its `stableId`/`itemType` changed, put the key in the pool for its **old** type.
3. Assign keys to engaged indices in two passes. First the indices whose `stableId` already has a key (they keep it). Then the rest, which take a key from the pool of the **same type**, or else a new key from the `keyCounter` ("0", "1", …).
4. Re-sync the keys that are still mapped to valid but non-engaged indices. They stay in the map, so **they stay mounted** at their old position.
5. `cleanup`: keys whose item vanished get handed to an unprocessed index of the same type if one exists (d6ef6759). Otherwise they are deleted. Then trim the off-screen surplus to `maxItemsInRecyclePool`.

`stableId` = `keyExtractor(item, index)`, or `String(index)` when there is no `keyExtractor` (`RecyclerViewManager.getDataKey`). `itemType` = `String(getItemType(item, index) ?? "default")`.

### Flow 1: first mount (two-pass render, progressive batches)

1. **Render #1.** `useRecyclerViewManager` creates the manager. It has no layout manager yet, so `ViewHolderCollection` renders no cells (`containerLayout` is undefined). If there is data and the first layout isn't complete, `RecyclerView` calls `parentRecyclerViewContext.markChildLayoutAsPending(id)`.
2. **Layout effect A** (`RecyclerView`, no deps, runs every commit): `measureParentSize(internalViewRef)` and `measureFirstChildLayout(firstChildViewRef, internalViewRef)`. The first child is a zero-size marker `CompatView` placed after `ListHeaderComponent` (before it in horizontal RTL). Its x/y becomes `firstItemOffset`. Then `recyclerViewManager.updateLayoutParams(windowSize, firstItemOffset)` creates the layout manager class picked by `getLayoutManagerClass` (masonry → grid → linear).
3. **Layout effect B** (no deps): `refHolder` is empty, so `modifyChildrenLayout([], n)` makes the layout manager create *estimated* layouts. `renderProgressively()` then applies the initial scroll position (`applyInitialScrollAdjustment`), takes the visible indices, and puts the first `numColumns * initialDrawBatchSize^ceil(count/5)` of them on the render stack (edfa37d3, geometric growth; `initialDrawBatchSize` defaults to 2, override via `overrideProps.initialDrawBatchSize`). It returns `true`, so `setRenderId` runs. React re-renders synchronously, still before paint.
4. **Render #2+.** `ViewHolder`s mount at their estimated `x/y`. Each one registers its `viewRef` in `refHolder` from its own `useLayoutEffect`. Effect B measures every ref with `measureItemLayout` and calls `modifyChildrenLayout` again. This repeats, with bigger batches each time, until every visible index has `isHeightMeasured && isWidthMeasured`. At that point `hasRenderedProgressively = isFirstLayoutComplete = true`.
5. **Commit.** `modifyChildrenLayout` returns false, so effect B calls `viewHolderCollectionRef.commitLayout()` (bumps `renderId` inside `ViewHolderCollection`) and `applyOffsetCorrection()`. `ViewHolderCollection` goes from `opacity: 0` to `1` (`renderId > 0`). Its layout effect fires `onCommitLayoutEffect` (`applyInitialScrollIndex`, unmark pending in the parent, then the user's `onCommitLayoutEffect`). Its passive effect fires `onCommitEffect` (record render time, `checkBounds`, viewability, `animationOptimizationsEnabled = false`).
6. **onLoad.** `useOnLoad` (passive effect) sees `getIsFirstLayoutComplete()`. One `requestAnimationFrame` later it sets `isFirstPaintOnUiComplete = true`, calls the user's `onLoad({elapsedTimeInMs})` and `setIsLoaded(true)`. That re-render runs effect B again, and now `recomputeEngagedIndices()` adds the `drawDistance` buffer items. Until `isFirstPaintOnUiComplete` is set, buffer items are deliberately not rendered (40fb21db, #1899).

### Flow 2: scroll → new cells → measure → commit

1. `RecyclerView` `onScrollHandler` (wrapped in `Animated.event` only when sticky headers exist). It returns early if `ignoreScrollEvents` is set, reads `contentOffset`, and runs `adjustOffsetForRTL` for horizontal RTL.
2. `VelocityTracker.computeVelocity(offset, manager.getAbsoluteLastScrollOffset(), …)` calls its callback **synchronously** with `(offset - last) / dt`. It also schedules a 100 ms timeout that calls back with zero velocity and `isMomentumEnd = true`. On momentum end the controller recomputes the first visible index for offset correction, and the velocity history is reset.
3. `recyclerViewManager.updateScrollOffset(offset, velocity)` subtracts `firstItemOffset` and calls `RVEngagedIndicesTrackerImpl.updateScrollOffset`. If the range changed, `updateRenderStack` → `RenderStackManager.sync`, and `setRenderId` triggers a render.
4. The same handler also calls `stickyHeaderRef.reportScrollEvent`, `checkBounds()` (`useBoundDetection`), `recordInteraction()` (only after the initial scroll completes, 317eaecb), `computeItemViewability()`, and finally the user's `onScroll`.
5. Render: `ViewHolderCollection` maps `renderStack.entries()` to `<ViewHolder key={recycleKey} index item layout={getLayout(index)} …>`. A recycled key keeps its component instance and gets a new `item`/`index`, so `renderItem` re-runs on the existing tree.
6. Layout effect B measures all `refHolder` views and calls `modifyChildrenLayout(layoutInfo, dataLength)` → `layoutManager.modifyLayout`. It re-renders if `layoutManager.requiresRepaint` is set, or if the corrected layouts change the engaged range. Otherwise it commits (`commitLayout` + `applyOffsetCorrection`).

### Flow 3: an item changes size after mount

- **Announced:** `useLayoutState` / `useRecyclingState` setters call `recyclerViewContext.layout()`. That runs `RecyclerView`'s `setLayoutTreeId`, which is itself a `useLayoutState`, so the request also bubbles to a parent FlashList. The re-render runs effect B, which re-measures before paint.
- **Unannounced:** `ViewHolder`'s native `onLayout` → `validateItemSize`. If the size differs from the stored layout (after min/max clamping), it calls `layout()`. This is a frame late and can flicker; the code has a TODO to warn about the missing `useLayoutState`.
- Container resize: the outer `CompatView` `onLayout` calls `layout()` when width or height changes. The `ViewHolderCollection` effect keyed on `fixedContainerSize` does the same.

### Data change

`useRecyclerViewManager` runs `updateProps(props)` in a `useMemo` on every props change, and `processDataUpdate()` in a `useMemo` keyed on `data` identity. Both run **during render**, before children render. `processDataUpdate` resizes the layouts and re-syncs the render stack even if the engaged range didn't change, so deleted items release their keys.

### Nested lists and `LayoutCommitObserver`

Child FlashLists register with the nearest `RecyclerViewContext` through `markChildLayoutAsPending(useId())`. While a parent has pending children, its effect B returns early: it measures nothing and does not commit. Each child unmarks in its own `onCommitLayoutEffect`, which calls the parent's `layout()`. A horizontal child unmarks as soon as its height is known, so the parent doesn't wait on every horizontal row to finish (40fb21db, fix for "Maximum update depth exceeded"). `LayoutCommitObserver` puts a pass-through context in the tree. It forwards mark/unmark to its parent and fires its own `onCommitLayoutEffect` once its subtree has no pending lists.

## Key types and entry points

| Symbol | File | Role |
|---|---|---|
| `RecyclerView` / `RecyclerViewComponent` | `RecyclerView.tsx` | The FlashList component. Effects A/B and `onScrollHandler` live here |
| `RecyclerViewManager` | `RecyclerViewManager.ts` | Facade over layout manager + engaged tracker + render stack + viewability. Key methods: `updateLayoutParams`, `modifyChildrenLayout`, `updateScrollOffset`, `processDataUpdate`, `renderProgressively`, `getRenderStack` |
| `RenderStackManager.sync` | `RenderStackManager.ts` | Key assignment and recycling |
| `RVEngagedIndicesTrackerImpl` | `helpers/EngagedIndicesTracker.ts` | Render window, buffer split, projection |
| `ConsecutiveNumbers` | `helpers/ConsecutiveNumbers.ts` | Index range `[startIndex, endIndex]` with array-like API. `EMPTY = (-1, -2)` |
| `VelocityTracker` | `helpers/VelocityTracker.ts` | px/ms velocity + 100 ms momentum-end timer |
| `RenderTimeTracker` | `helpers/RenderTimeTracker.ts` | Average render time (Android), and the 40-renders-without-commit guard |
| `ViewHolderCollection` / `ViewHolderCollectionRef.commitLayout` | `ViewHolderCollection.tsx` | Renders cells, gates opacity, fires commit callbacks |
| `ViewHolder` | `ViewHolder.tsx` | One cell: absolute position, separator, `CellRendererComponent`, ref registration |
| `RecyclerViewContext` / `FlashListContext` / `useFlashListContext` | `RecyclerViewContextProvider.ts` | `layout()`, refs, pending-child protocol. Public subset is `FlashListContext` |
| `useLayoutState`, `useRecyclingState`, `useMappingHelper`, `LayoutCommitObserver` | `hooks/…`, `LayoutCommitObserver.tsx` | Public hooks exported from `src/index.ts` |
| `measureParentSize`, `measureFirstChildLayout`, `measureItemLayout`, `areDimensionsEqual` | `utils/measureLayout.ts` (+ `.web.ts`) | Sync measurement and 1-physical-pixel tolerance |
| `useBoundDetection.checkBounds` | `hooks/useBoundDetection.ts` | `onEndReached` / `onStartReached` / MVCP autoscroll-to-bottom |
| `useSecondaryProps` | `hooks/useSecondaryProps.tsx` | Header/footer/empty/refresh control, and wraps the scroll component in `Animated.createAnimatedComponent` |

## Invariants and gotchas

- **Measurement is synchronous and depends on Fabric.** `measureLayoutRelative` calls `view.measureLayout(relativeTo, cb)` and reads the values right after the call returns. **Inferred:** this only works because Fabric runs the callback synchronously. That would be the real reason `src/index.ts` throws `flashListV2OnlySupportsNewArchitecture` on the old architecture. On web, `measureLayout.web.ts` reads DOM `clientWidth`/`getBoundingClientRect`, and item `x/y` are always 0.
- **Effects A and B have no dependency array on purpose.** They run after every commit of `RecyclerView`. Anything that re-renders the list re-measures every mounted cell.
- **React key ≠ `keyExtractor`.** A cell's React key is a recycle-counter string. Component state inside `renderItem` survives when the cell is reused for a different item. That's what `useRecyclingState(initial, deps)` is for: it resets inside a `useMemo` while rendering when deps change, so no extra render. Its setter is a no-op when the value is unchanged, and it also triggers `layout()`.
- **Recycling only happens within one item type.** A key never moves between types, so a new type always mounts a new cell. `getItemType` also keys the per-type size averages in the layout manager (`MultiTypeAverageWindow`), so it affects estimates too.
- **Off-screen recycled cells stay mounted.** They are in the render stack at their old positions. `maxItemsInRecyclePool` (no limit by default) caps how many; `0` unmounts cells as soon as they leave the engaged range.
- **`removeClippedSubviews={false}` is forced** on the ScrollView, placed before `{...overrideProps}` so users can still override it. Native clipping detached buffered cells and crashed Fabric on Android with "Unable to find viewState for tag" (4fa1ccdc, #2202, fixes #1914).
- **Without `keyExtractor`, stableId = index.** Inserts and deletes then look like content changes at every later index. MVCP can't anchor (warning `keyExtractorNotDefinedForMVCP` when `onStartReached` is set), and `prepareForLayoutAnimationRender` warns too.
- **`ViewHolder` does not re-run `renderItem` when only `index` changes.** `children` is `useMemo`'d on `[item, extraData, target, renderItem]`; there's a TODO in `ViewHolder.tsx`. So a `renderItem` that shows `index` can go stale when items shift but `item` stays the same object.
- **`data` must change identity.** `processDataUpdate` is keyed on `data`. Mutating the array in place does not re-layout.
- **`horizontal` cannot be toggled** after the layout manager exists (throws `horizontalPropCannotBeToggled`). `masonry` + `horizontal` and `numColumns > 1` + `horizontal` also throw. Switching `masonry`/`numColumns` swaps the layout manager class, and the new one is built from the previous one (`new LayoutManagerClass(params, this.layoutManager)`). The ref method `clearLayoutCacheOnUpdate` → `markLayoutManagerDirty` drops the old layouts on the next `updateLayoutParams`.
- **Infinite-loop guard:** after 40 renders without a commit, `RecyclerView` warns `exceededMaxRendersWithoutCommit` and forces a commit instead of re-rendering (a102767a, #1826 turned the old React depth error into this warning). Usual causes are duplicate `keyExtractor` output or a list inside a ScrollView.
- **Dimension comparisons allow ±1 physical pixel** (`areDimensionsEqual` via `PixelRatio`), and measured values snap back to the old ones within tolerance. Without this, sub-pixel jitter would trigger endless re-layout.
- **Opacity gate:** `ViewHolderCollection` renders with `opacity: 0` until its first commit. The code comment says "useLayoutEffect doesn't block paint in some cases".
- `prepareForLayoutAnimationRender` sets `animationOptimizationsEnabled`, which sets `RenderStackManager.disableRecycling`. It lasts until the next `onCommitEffect` resets it, so effectively one render.
- `onEndReached` fires once per crossing into the threshold (default 0.5 viewport; `onStartReached` default 0.2). The latch resets when `data` changes. Bound checks are skipped when the window size is 0, e.g. a list mounted in a hidden tab (2b8ec28d, #2146). Autoscroll-to-bottom is suppressed while offset projection is off, i.e. during `scrollToIndex` (8ca74776, #2290).
- `RenderTarget` `"Measurement"` is still declared in `FlashListProps.ts`, but v2 code never emits it. Only `"Cell"` and `"StickyHeader"` occur.
- `getItemType`'s prop type takes `extraData`, but `RecyclerViewManager.getItemType` calls it with only `(item, index)`.
- `CellRendererComponent` receives `ref`, `onLayout`, `style` and `index`. It must forward `ref` and `onLayout` to a host view, or measurement breaks.
- `ViewHolder`s are `position: "absolute"`, except `target="StickyHeader"`, which is `relative`. Width and height are only set when the layout manager sets `enforcedWidth` / `enforcedHeight`.

## Design decisions

| Decision | Reason / evidence |
|---|---|
| Mutable manager object + counter state for re-renders | Keeps layout and recycling math out of React state so it can run inside layout effects and scroll callbacks. **Inferred** from structure (`useRecyclerViewManager`, `setRenderId`) |
| Measure in `useLayoutEffect`, re-render before paint, no size estimate props | v2 "does not ask for any estimates" (`documentation/docs/v2-changes.md`). Estimates come from per-type running averages instead |
| Progressive first render with geometric batches | Shows the first screen fast without rendering unmeasured items far down. edfa37d3 "Use geometric progression in progressive rendering" |
| Delay buffer rendering until after the first UI paint (`isFirstPaintOnUiComplete`) | Cuts setState churn when many lists are nested (40fb21db, #1899) |
| Clamp render window to the real viewport | Projection with a small trailing buffer left blank bands during flings; `drawDistance={0}` always hit it (9b5e7f8b, #2510) |
| Reuse keys that would otherwise be deleted for the same type | Fewer mounts on data replacement (d6ef6759) |
| Dispose on unmount, `restoreIfNeeded` on mount | Setup and cleanup calls were unbalanced (951eca97, #1695). **Inferred:** React StrictMode / Fast Refresh runs effect cleanup and then setup again on the same manager instance |
| Suppress separators in a grid's last row (`isInLastRow`) | Separator height inside a measured cell skewed row heights (8fdd9e86, #2114, fixes #1868) |
| `measureParentSize` returns size only | x/y were always 0 on Paper/web but non-zero on Fabric, which shifted sticky headers (f07b9b35, #2105) |
| `getValidComponent` accepts memo/forwardRef/class/elements | All component-ish props (header/footer/empty/backdrop) must accept every form (80f5695e, #2115) |

## Where to change things

| To… | Start at |
|---|---|
| Change how far ahead items render / buffer split | `RVEngagedIndicesTrackerImpl.updateScrollOffset`, `PlatformConfig.defaultDrawDistance` |
| Change key reuse / pool rules | `RenderStackManager.sync`, `cleanup`, `syncItem` |
| Change first-load batching | `RecyclerViewManager.renderProgressively`, `initialDrawBatchSize` |
| Change when re-render vs commit happens | `RecyclerView` effect B, `RecyclerViewManager.modifyChildrenLayout` |
| Change what a cell renders / its style | `ViewHolder.tsx` (memo comparator `areLayoutsEqual` too), `ViewHolderCollection.tsx` |
| Add a prop that affects layout params | `RecyclerViewManager.updateLayoutParams` (`LayoutParams`), then [layout-managers](layout-managers.md) |
| Change `onEndReached`/`onStartReached` | `hooks/useBoundDetection.ts` |
| Change `onLoad` timing | `hooks/useOnLoad.ts` |
| Nested list / commit coordination | `RecyclerViewContextProvider.ts`, `pendingChildIds` in `RecyclerView.tsx`, `LayoutCommitObserver.tsx` |
| Header/footer/empty/refresh/scroll component | `hooks/useSecondaryProps.tsx`, `utils/componentUtils.ts` |
| Measurement on a platform | `utils/measureLayout.ts`, `utils/measureLayout.web.ts` |

## Tests

| File (`src/__tests__/`) | Covers |
|---|---|
| `RenderStackManager.test.ts` | Key reuse across type changes, `disableRecycling`, `maxItemsInRecyclePool` (incl. 0), empty-data edge cases |
| `EngagedIndicesTracker.test.ts` | Viewport-coverage clamp with `drawDistance` 0, no-op at default, boundary redistribution, negative offsets |
| `RecyclerView.test.tsx` | Full render with mocked measurement for linear/grid/masonry/horizontal; grid separators; viewability + `initialScrollIndex`; sticky headers with content above; autoscroll suppression |
| `RecyclerViewManager.test.ts` | `keyExtractor` warning |
| `ConsecutiveNumbers.test.ts`, `LayoutCommitObserver.test.tsx`, `useUnmountAwareCallbacks.test.tsx`, `componentUtils.test.tsx` | Helpers |

- Run only these: `yarn test "RecyclerView|RenderStackManager|EngagedIndicesTracker|ConsecutiveNumbers|LayoutCommitObserver|useUnmountAwareCallbacks|componentUtils" --forceExit`. All 8 suites (106 tests) passed at 527d7678. If `yarn` resolves to Yarn 4 via corepack and fails with "package doesn't seem to be present in your lockfile" (the repo uses a v1 `yarn.lock`), run `node node_modules/jest/bin/jest.js <pattern> --forceExit` instead.
- Tests that render the list **must** `jest.mock("../recyclerview/utils/measureLayout", …)`. Without a host the measure callback never fires and every size is 0. Copy the mock block at the top of `RecyclerView.test.tsx`. That file also passes `drawDistance={0}` and `overrideProps={{ initialDrawBatchSize: 1 }}` to keep the render set predictable. `src/__tests__/helpers/createLayoutManager.ts` builds layout managers for unit tests.
- `shared/testing/resolver.js` resolves `.ts` before `.ios.js`/`.android.js`, so tests get the base `PlatformHelper.ts` config (`supportsOffsetCorrection: false`, no render-time tracking), not the iOS/Android one (verified by logging `PlatformConfig` from a test).
- `jestSetup.js` at the repo root ships to consumers (it mocks `@shopify/flash-list/dist/...measureLayout`). It is not this repo's test setup.
- Anything about real pixels, flings, or Fabric timing needs a device: see the `agent-device` / `review-and-test` skills.
