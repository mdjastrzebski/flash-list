<!-- index-codebase: generated from 527d7678 on 2026-10-02. Code wins when it disagrees with this doc. -->
# Scrolling

**Purpose:** Imperative scrolling (`scrollToIndex`, `scrollToOffset`, `scrollToEnd`, `scrollToItem`, `scrollToTop`), the initial scroll position (`initialScrollIndex`, `initialScrollIndexParams`, `maintainVisibleContentPosition.startRenderingFromBottom`), and keeping the scroll position steady when content above the viewport changes (`maintainVisibleContentPosition`, or "MVCP", plus `autoscrollToBottomThreshold`). It also covers how `horizontal`, RTL and `inverted` affect offsets. It does **not** decide which items render for a given offset. The render window ("engaged indices"), offset projection and the velocity buffer belong to [recycler-core](recycler-core.md). Item positions come from [layout-managers](layout-managers.md). Prop and ref docs for users are in [public-api](public-api.md).

**Paths:** `src/recyclerview/hooks/useRecyclerViewController.tsx` (the whole ref API plus offset correction), `src/recyclerview/components/ScrollAnchor.tsx`, `src/recyclerview/utils/findVisibleIndex.ts`, `src/recyclerview/utils/adjustOffsetForRTL.ts`, `src/recyclerview/utils/getInvertedTransformStyle.ts`, `src/FlashListRef.ts` (the ref type). Collaborators: `src/recyclerview/hooks/useBoundDetection.ts` (autoscroll to bottom), `src/recyclerview/RecyclerView.tsx` (`onScrollHandler` and the layout effects that call the controller), `src/recyclerview/RecyclerViewManager.ts` (`applyInitialScrollAdjustment`, `getInitialScrollIndex`, `ignoreScrollEvents`), `src/native/config/PlatformHelper*.ts`.

**Depends on:** [recycler-core](recycler-core.md), [layout-managers](layout-managers.md) · **Used by:** [public-api](public-api.md), [viewability](viewability.md) (`recomputeViewableItems`, the `isInitialScrollComplete` gate), [sticky-headers](sticky-headers.md) (`firstItemOffset`)

## How it works

### Two different scroll offsets

Most scroll bugs here come from mixing up these two numbers:

| Name | What it is | Read with |
|---|---|---|
| **Native / absolute offset** | What the `ScrollView` reports and accepts. Includes the header and padding above the first item (`firstItemOffset`). | `recyclerViewManager.getAbsoluteLastScrollOffset()` (= tracker offset + `firstItemOffset`) |
| **Tracker / relative offset** | `RVEngagedIndicesTrackerImpl.scrollOffset`, measured from the first item. It drives the render window, `computeVisibleIndices()` and `getFirstVisibleIndex()`. JS can move it **without moving the native view**. | `recyclerViewManager.getLastScrollOffset()` |

`RecyclerViewManager.updateScrollOffset(offset)` takes an **absolute** offset and subtracts `firstItemOffset` before passing it to the tracker. The comments on `getLastScrollOffset` and `getAbsoluteLastScrollOffset` in `RecyclerViewManager.ts` are the wrong way round. Trust the code, not the comments.

`useRecyclerViewController.updateScrollOffsetWithCallback(offset, cb)` moves only the tracker offset. If the engaged indices change, it queues `cb` in `pendingScrollCallbacks` and calls `setRenderId` to force a re-render. Otherwise it runs `cb` straight away, synchronously. The queue is drained at the start of `applyOffsetCorrection`. That function runs only from `RecyclerView.tsx`'s measurement `useLayoutEffect`, and only once `modifyChildrenLayout` reports that layout is stable. So **each queued step waits until the newly rendered items have been measured.**

### Flow: `ref.scrollToIndex({ index, animated, viewPosition, viewOffset })`

All of this is in `handlerMethods.scrollToIndex` in `useRecyclerViewController.tsx`.

1. Out-of-range `index`, or no `scrollViewRef`, resolves at once and does nothing.
2. Two things are switched off for the duration: `pauseOffsetCorrection.current = true` (stops MVCP corrections) and `recyclerViewManager.setOffsetProjectionEnabled(false)` (stops velocity look-ahead). Other code treats **"offset projection disabled"** as the signal that a scrollToIndex is in flight (see `useBoundDetection`).
3. `getFinalOffset()` reads `getLayout(index)`. That layout is **estimated** if the item has never been measured. It then applies `viewPosition` (`offset - (window - itemSize) * viewPosition`), adds `viewOffset`, and adds `firstItemOffset`, giving an absolute offset.
4. `getStartScrollOffset()` clamps the start point to at most `2 × window` (`bufferForCompute`) from the target. It also calls `setScrollDirection("forward" | "backward")` so the render buffer leans toward the target.
5. `performScrollStep` runs 5 steps (`steps = 5`). Each step calls `updateScrollOffsetWithCallback` with an interpolated offset. These steps are **virtual**: they move the render window, render and measure the items there, and the next step runs only after layout settles. The native view does not move.
   - Non-animated: the steps go start → target, so the target area is measured last.
   - Animated: the steps go target → start, so the target area gets measured, and the window ends at the start point where the native animation begins.
6. After each step:
   - If `index >= getDataLength()` (the data shrank), it falls back to `scrollToEnd` and resolves.
   - If the recomputed target moved past both the original target and the original start (estimates turned into real measurements), it recomputes target and start and **restarts at step 0**.
7. `finishScrollToIndex` recomputes the target and clamps it to `getMaxScrollOffset()`. If `animated`, it first jumps without animation to `startScrollOffset`. Then it calls `scrollToOffset({ offset, animated, skipFirstItemOffset: true })`. This is the only time the native view moves.
8. After `setTimeout(animated ? 300 : 200)`, it clears `pauseOffsetCorrection`, re-enables projection, and **then** resolves the promise. Nothing corrects the position after the native scroll. Once the target is measured, the step-0 restart is the only re-correction.

`scrollToOffset` itself: if `I18nManager.isRTL && horizontal`, it mirrors the offset with `adjustOffsetForRTL` and folds in `firstItemOffset`. It then adds `firstItemOffset` only when `skipFirstItemOffset === false` and calls `scrollViewRef.current.scrollTo({x|y, animated})`. `skipFirstItemOffset` **defaults to `true`**, so the public `scrollToOffset({offset})` scrolls to a raw native offset with the header included.

`scrollToEnd`: if the last index is not engaged, it first `await`s `scrollToIndex(last)`. This step is skipped when the last item is already rendered (94f1d120, #1781). Then, after a `setTimeout(0)`, it calls the native `scrollToEnd`.

### Flow: initial scroll (`initialScrollIndex` / `startRenderingFromBottom`)

1. `RecyclerViewManager.getInitialScrollIndex()` returns `initialScrollIndex`, or `dataLength - 1` when `startRenderingFromBottom` is set. The constructor sets `isInitialScrollComplete = (that === undefined)`.
2. Virtual phase: `RecyclerViewManager.renderProgressively` → `applyInitialScrollAdjustment()` calls `layoutManager.recomputeLayouts(0, index)` **and only then** reads the target's x/y into the tracker offset, so the first render window is around the target. Reading before the recompute rendered the wrong window, leaving blank space or overlapping items (348c9ea5, #1797).
3. Native phase: `applyInitialScrollIndex` in the controller runs from both `onCommitLayoutEffect` and `onCommitEffect` of `ViewHolderCollection`. Once `getIsFirstLayoutComplete()` is true and `!isInitialScrollComplete`, it:
   - pauses MVCP;
   - calls `scrollToOffset({ offset: layout.y|x + initialScrollIndexParams.viewOffset, skipFirstItemOffset: false })` immediately and again in `setTimeout(0)`;
   - schedules `isInitialScrollComplete = true` plus an unpause after 100 ms.
   Every commit inside that 100 ms scrolls again, so the position keeps up as estimates turn into measurements.
4. `onScrollHandler` skips `recordInteraction()` until `isInitialScrollComplete`, so the programmatic scroll doesn't satisfy `waitForInteraction` (317eaecb, #1943).
5. `startRenderingFromBottom` with content shorter than the window: `ViewHolderCollection` receives `getAdjustmentMargin()` = `max(0, window - content - firstItemOffset)` as `marginTop`/`marginLeft`, which pushes the items down so they sit at the bottom.

### Flow: data prepended → MVCP keeps the first visible item in place

MVCP is **on by default**. `shouldMaintainVisibleContentPosition()` returns `!maintainVisibleContentPosition?.disabled`. It works only with a `keyExtractor` (`hasStableDataKeys()`).

1. **Snapshot.** `computeFirstVisibleIndexForOffsetCorrection()` stores the key of `computeVisibleIndices().startIndex` (`firstVisibleItemKey`) and a **copy** of its layout (`firstVisibleItemLayout`). It runs at the end of every `applyOffsetCorrection` and on synthetic "momentum end" in `onScrollHandler`. "Momentum end" is `VelocityTracker`'s 100 ms with no scroll events.
2. Items are prepended. The parent re-renders, the layout manager shifts the existing items down with estimated sizes for the new ones, and the measurement layout effect loops until layout is stable. Then it calls `commitLayout()` and `applyOffsetCorrection()`.
3. `applyOffsetCorrection` first drains `pendingScrollCallbacks`. Then it requires first layout complete, a `keyExtractor`, data length > 0 and MVCP enabled. `hasDataChanged` compares **lengths only**.
4. **Find the anchor's new index.** It searches `getEngagedIndices()` first. If that misses, it searches the full `data` array, but only when `hasDataChanged || ignoreScrollEvents`. The second condition keeps the anchor findable on follow-up passes, where large unmeasured items converge and push it out of the engaged window (43b0f317, #2136).
5. `diff = newLayout.y − snapshot.y` (`.x` when horizontal). The snapshot is refreshed **even if no correction is applied**.
6. If `diff !== 0 && !pauseOffsetCorrection && !animationOptimizationsEnabled`:
   - **iOS / Android** (`PlatformConfig.supportsOffsetCorrection`): `scrollAnchorRef.current.scrollBy(diff)` (see the ScrollAnchor mechanics below).
   - **Web and other platforms** (`PlatformHelper.web.ts`, default `PlatformHelper.ts`): `scrollViewRef.scrollTo({ y: absolute + diff, animated: false })`.
   - If `hasDataChanged`, it also moves the tracker offset by `diff` (`updateScrollOffsetWithCallback`), so the render window matches before any scroll event arrives. It also sets `recyclerViewManager.ignoreScrollEvents = true` for 100 ms, so scroll events caused by the correction itself are dropped.
7. It re-snapshots and stores `lastDataLengthRef`.

**ScrollAnchor mechanics** (`ScrollAnchor.tsx`, mounted in `RecyclerView.tsx` as the **first child** of the ScrollView only when MVCP is on):

- The anchor is a zero-height `position: absolute` view at `top` (or `left`) = `1000000` plus all previous diffs.
- `RecyclerView.tsx` passes the native `maintainVisibleContentPosition={{...userConfig, minIndexForVisible: 0}}`.
- `scrollBy(diff)` is a `setState` that moves the anchor by `diff`. Native MVCP then shifts the content offset by the same amount, in the same native commit as the layout change.
- **Inferred:** native MVCP picks the first child at or after `minIndexForVisible` that lies below the current offset. The anchor is child 0 and sits 1e6 px down, so it is always the view native MVCP tracks. Real items can't be anchors because they are absolutely positioned inside one `ViewHolderCollection` container.

### Autoscroll to bottom (`useBoundDetection`)

- `checkBounds()` runs on every scroll and every commit. For **vertical lists only**, it sets the `pendingAutoscrollToBottom` ref when the list is within `autoscrollToBottomThreshold × window` of the end. The default is `-1`, which means off (96146b8c, #1737).
- `runAutoScrollToBottomCheck` runs from two effects:
  - one on `[data, windowHeight, windowWidth]` changes;
  - one on content-size changes, which fires only if `checkBounds` has not run in the last 100 ms, i.e. the user isn't actively scrolling (4980b04a).
- It returns early while `!isOffsetProjectionEnabled`, i.e. while a scrollToIndex is in flight. Otherwise the leftover pending flag fired `scrollToEnd` halfway through a scrollToIndex (8ca74776, #2290).
- It scrolls with `requestAnimationFrame` → native `scrollToEnd({ animated: (animateAutoScrollToBottom ?? true) && !ignoreScrollEvents })`. Animation is forced off while an MVCP correction is settling, to avoid jitter (5a15a807, #2095).
- `autoscrollToTopThreshold` is never read in JS. It reaches the native ScrollView only through the spread of the user config.

### `horizontal`, RTL, `inverted`

| Mode | What changes for offsets |
|---|---|
| `horizontal` | Every computation switches from `y`/`height` to `x`/`width`. MVCP works horizontally since 422dcda9 (#2145). Before that, `shouldMaintainVisibleContentPosition` returned false for horizontal lists. Autoscroll-to-bottom stays vertical-only. |
| RTL + `horizontal` | Native offsets count from the right. `adjustOffsetForRTL(offset, contentSize, windowSize) = contentSize − offset − windowSize`, which is its own inverse. It is applied to incoming offsets in `onScrollHandler` (using the event's `contentSize`/`layoutMeasurement`) and to outgoing offsets in `scrollToOffset` (using `getChildContainerDimensions().width`/`getWindowSize().width`). `RecyclerView.tsx` also places the `firstChildViewRef` measuring view **before** the header and redefines `firstItemOffset` as `measured.x − childContainerWidth` once layout exists. Expected runtime values are in `.claude/skills/review-and-test/SKILL.md` ("Verify firstItemOffset Values"). RTL in vertical lists changes nothing. |
| `inverted` | **Visual only.** `getInvertedTransformStyle(horizontal)` flips the outer container, and the same transform flips each cell (`ViewHolder.tsx`), the header, footer, empty component and sticky backdrop back (`useSecondaryProps.tsx`). All offsets, `scrollToIndex`, MVCP and bound detection keep working in logical coordinates, so offset 0 is at the visual bottom and `scrollToEnd` goes to the visual top. iOS and web use `scaleY(-1)`/`scaleX(-1)`. Android uses `rotate(180deg)` for both axes for performance (#751, e2854a53/#851), which puts the scrollbar on the left (8c008171, #1844). |

## Key types and entry points

| Symbol | File | Role |
|---|---|---|
| `useRecyclerViewController` | `src/recyclerview/hooks/useRecyclerViewController.tsx` | Builds `handlerMethods` (the ref API) and returns `applyOffsetCorrection`, `computeFirstVisibleIndexForOffsetCorrection`, `applyInitialScrollIndex` to `RecyclerView.tsx` |
| `handlerMethods.scrollToIndex` | same | Virtual steps, then one native scroll. Returns `Promise<void>` |
| `updateScrollOffsetWithCallback` | same | Moves the tracker offset and continues once the new layout has committed |
| `applyOffsetCorrection` | same | MVCP correction plus draining `pendingScrollCallbacks` |
| `pauseOffsetCorrection` (ref) | same | Turns MVCP off during scrollToIndex and the initial scroll |
| `ScrollAnchor` / `ScrollAnchorRef.scrollBy` | `src/recyclerview/components/ScrollAnchor.tsx` | Moving view that native MVCP tracks |
| `adjustOffsetForRTL` | `src/recyclerview/utils/adjustOffsetForRTL.ts` | Converts offsets between LTR and RTL, in both directions |
| `getInvertedTransformStyle` | `src/recyclerview/utils/getInvertedTransformStyle.ts` | Picks `PlatformConfig.invertedTransformStyle[Horizontal]` |
| `findFirstVisibleIndex` / `findLastVisibleIndex` | `src/recyclerview/utils/findVisibleIndex.ts` | Binary search over layouts sorted by x/y. Used by `LayoutManager.getVisibleLayouts`, which backs `computeVisibleIndices` and the engaged indices |
| `FlashListRef<T>`, `ScrollToIndexParams`, `ScrollToOffsetParams` | `src/FlashListRef.ts` | Type contract for the ref |
| `ignoreScrollEvents`, `isInitialScrollComplete`, `firstItemOffset` | `src/recyclerview/RecyclerViewManager.ts` | Public mutable flags shared with `RecyclerView.tsx` and `useBoundDetection` |
| `setOffsetProjectionEnabled` / `isOffsetProjectionEnabled` | `src/recyclerview/RecyclerViewManager.ts` | Also serves as the "scrollToIndex in flight" signal |
| `PlatformConfig.supportsOffsetCorrection` | `src/native/config/PlatformHelper*.ts` | `true` on iOS/Android, `false` on web and the default (other platforms) |
| `useBoundDetection` | `src/recyclerview/hooks/useBoundDetection.ts` | `onStartReached`/`onEndReached` and autoscroll to bottom |

## Invariants and gotchas

- **MVCP does nothing without a `keyExtractor`, and gives no warning.** `WarningMessages.keyExtractorNotDefinedForMVCP` fires only when `onStartReached` is set without a `keyExtractor` (`checkPropsAndWarn`).
- **`ignoreScrollEvents` drops whole scroll events.** For 100 ms after a correction that changed data length, `onScrollHandler` returns at its first line. The user's `onScroll`, sticky-header updates, `checkBounds` and viewability are all skipped, not just the offset update.
- **Mid-flight reads are virtual.** During `scrollToIndex`, `computeVisibleIndices()`, `getFirstVisibleIndex()` and `getAbsoluteLastScrollOffset()` report the step offsets, not what is on screen.
- **The promise resolves on a timer, not on native completion.** It resolves 200 ms (instant) or 300 ms (animated) after the final `scrollTo`. A long native animation can still be running.
- **Any new programmatic scroll path must disable offset projection** (or add its own guard in `runAutoScrollToBottomCheck`). Otherwise autoscroll-to-bottom can override it.
- **The MVCP snapshot keeps refreshing while paused.** Only the correction is skipped. That way, when the pause lifts there is no stale diff to apply.
- **`hasDataChanged` is length-based.** Replacing items with the same count doesn't trigger the full-data anchor search or the tracker-offset update.
- **The clamp ignores the footer.** `getMaxScrollOffset()` = content + `firstItemOffset` − window. `ListFooterComponent` is not included, so `scrollToIndex` can't scroll into the footer.
- **The ref types don't match the code** in `FlashListRef.ts`:
  - `scrollToEnd` is typed `void` but is `async`;
  - `scrollToItem` is typed `void`, drops the promise, and matches items with `===` identity;
  - the `skipFirstItemOffset` JSDoc doesn't say that it defaults to `true`;
  - `prepareForLayoutAnimationRender`'s "remember to reset" note is stale. `onCommitEffect` resets `animationOptimizationsEnabled = false` after the next commit. While it is `true`, MVCP corrections are skipped (c9bfdb5f, #1715).
- **The ref object is a spread** of the native scroll ref and `handlerMethods`. `props` is redefined as a live getter, because spreading had frozen it (a93953dc).
- **`ScrollAnchor` position is hard-coded.** It starts at `1000000` with a `// TODO: Fix this value`. **Inferred:** content longer than about 1e6 px, or many cumulative negative diffs, could stop the anchor from being the view native MVCP tracks.
- **`findVisibleIndex` assumes layouts sorted by position.** Masonry columns break that assumption. The masonry tests only assert `!== -1`. For masonry, `RecyclerViewManager.computeItemViewability` uses engaged indices instead (see [viewability](viewability.md)).
- **Inverted lists still use logical coordinates.** `autoscrollToBottomThreshold`, `onEndReached` and `startRenderingFromBottom` refer to the **logical** end. In an inverted list that end is at the visual top. **Inferred:** the logical/visual flip is a common source of confusion in issues.
- **Repo Jest resolves the generic `PlatformHelper.ts`** (`shared/testing/resolver.js` lists `.ts` before `.ios.js`; verified by logging `PlatformConfig` in a test), so unit tests run with `supportsOffsetCorrection: false`. Tests therefore hit the web-style `scrollTo` correction branch, and the iOS/Android `ScrollAnchor.scrollBy` branch has no unit coverage. Verify MVCP changes in the fixture app.

## Design decisions

| Decision | Why / evidence |
|---|---|
| Correct MVCP by moving an invisible anchor, not with `scrollTo` | Native MVCP applies the shift in the same frame as the layout change, so there is no flash (header comment in `ScrollAnchor.tsx`). The `scrollTo` fallback exists only where `supportsOffsetCorrection` is false. Memoized in f9bf76dc. |
| scrollToIndex renders virtual steps before one native scroll | Unmeasured targets have estimated positions. Rendering the path first lets the estimates settle, and the restart rule catches large drift. The 2× window clamp bounds how far the animated scroll travels and how much must be rendered. **Inferred** rationale for the 2× and 5 values; no commit explains them. |
| Steps use callbacks, not promises | 4202c8ab replaced `updateScrollOffsetAsync` with `updateScrollOffsetWithCallback`. **Inferred:** promise microtasks ran outside React's commit ordering; callbacks run synchronously inside `applyOffsetCorrection`. |
| Anchor lookup: engaged indices first, then full `data` | Fast in the common case. The full search is needed after a prepend and while a correction is settling (43b0f317, #2136). |
| The initial scroll has a virtual and a native phase, retried for 100 ms | The first render must already be at the target (348c9ea5, #1797). The native position must follow estimate changes over the first few commits (comment in `applyInitialScrollIndex`). `viewOffset` added in 8d923d06 (#1870). |
| `isInitialScrollComplete` moved from a local ref onto the manager | So `onScrollHandler` can skip `recordInteraction` during the programmatic initial scroll (317eaecb, #1943). |
| Autoscroll is blocked while projection is disabled, and not animated while `ignoreScrollEvents` is set | 8ca74776 (#2290) and 5a15a807 (#2095). Without them, autoscroll raced with imperative scrolls and MVCP corrections. |
| The render window is clamped to always cover the real viewport | Velocity projection could leave blank bands during flings (9b5e7f8b, #2510; lives in [recycler-core](recycler-core.md)). |

## Where to change things

| To… | Start at |
|---|---|
| Change how the scrollToIndex target is computed (`viewPosition`, clamping, headers) | `getFinalOffset` / `finishScrollToIndex` in `useRecyclerViewController.tsx` |
| Fix scrollToIndex landing in the wrong place for unmeasured items | `performScrollStep` restart condition, `steps`, `bufferForCompute` (same file) |
| Fix a scroll jump on prepend or item resize | `applyOffsetCorrection` + `computeFirstVisibleIndexForOffsetCorrection` (same file); `ScrollAnchor.tsx` |
| Change platform behavior (correction method, inverted transform) | `src/native/config/PlatformHelper.{ios,android,web}.ts` and the default `PlatformHelper.ts` |
| Fix the initial position being off or blank | `RecyclerViewManager.applyInitialScrollAdjustment` (virtual) and `applyInitialScrollIndex` (native) |
| Fix chat autoscroll | `useBoundDetection.ts` (`checkBounds`, `runAutoScrollToBottomCheck`) |
| Fix horizontal RTL offset bugs | `adjustOffsetForRTL` call sites in `RecyclerView.tsx` `onScrollHandler`, `handlerMethods.scrollToOffset`, and the `firstItemOffset` computation in `RecyclerView.tsx`'s first `useLayoutEffect` |
| Add or change a ref method | `src/FlashListRef.ts` (type) plus `handlerMethods` (implementation). Update the docs per [public-api](public-api.md) |
| Fix wrong visible / first-visible indices | `findVisibleIndex.ts`, `LayoutManager.getVisibleLayouts`, `RVEngagedIndicesTrackerImpl.computeVisibleIndices` |

## Tests

```bash
yarn test "RecyclerView.test|findVisibleIndex|EngagedIndicesTracker|RecyclerViewManager" --forceExit
# If `yarn` resolves to Yarn 4 via corepack, it fails ("not present in your lockfile"); the repo uses a v1 lockfile.
# Fallback after installing deps with Yarn 1: node_modules/.bin/jest "<pattern>" --forceExit
```

At 527d7678, these 4 suites (65 tests) pass in about 2 s.

| Test | Covers |
|---|---|
| `src/__tests__/RecyclerView.test.tsx` → "autoscrolling to bottom is suppressed while offset projection is disabled" | The scrollToIndex / autoscroll race (#2290). Its comments show how to fake scroll events (`onScroll` with a `nativeEvent`) and remeasurement (`measureItemLayout.mockImplementation`). |
| `RecyclerView.test.tsx` → "Viewability with initialScrollIndex", "RecyclerView ref" | `isInitialScrollComplete` gating; the live `props` getter on the ref |
| `src/__tests__/findVisibleIndex.test.ts` | The binary search across linear, grid, masonry and edge cases |
| `src/__tests__/EngagedIndicesTracker.test.ts` → "real viewport coverage with drawDistance 0" | The projection clamp that scrollToIndex depends on |
| `src/__tests__/RecyclerViewManager.test.ts` | The keyExtractor/MVCP warning |

**No unit tests** cover `applyOffsetCorrection`, `ScrollAnchor`, the scrollToIndex steps, RTL or `inverted`. To check those, use the fixture screens in `fixture/react-native/src/` (`Chat.tsx`, `ChatUnstableItems.tsx`, `ChatInverted.tsx`, `HorizontalMVCP.tsx`, `InvertedTest.tsx`) and the e2e tests in `fixture/react-native/e2e/tests/` (`Chat`, `ChatInverted`, `InvertedTest`, `RecyclerViewHandler`). Run them on a device with the `review-and-test` / `agent-device` skills. That skill also documents how to enable RTL correctly (`I18nManager.forceRTL` in `fixture/react-native/index.js`, then a full relaunch).
