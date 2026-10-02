<!-- index-codebase: generated from 527d7678 on 2026-10-02. Code wins when it disagrees with this doc. -->
# Sticky headers

**Purpose:** Pins the most recent `stickyHeaderIndices` item to the top of a vertical list and animates the push-out when the next header arrives. Supports a top `offset`, a backdrop layer, a configurable `zIndex`, optional hiding of the source cell, and `onChangeStickyIndex`. It does **not** use RN `ScrollView`'s native `stickyHeaderIndices`. FlashList renders a separate copy of the item outside the scroll view. Horizontal lists are not supported.

**Paths:** `src/recyclerview/components/StickyHeaders.tsx`. Wiring is in `src/recyclerview/RecyclerView.tsx`, `src/recyclerview/hooks/useSecondaryProps.tsx` (`renderStickyHeaderBackdrop`), `src/recyclerview/ViewHolderCollection.tsx` and `src/recyclerview/ViewHolder.tsx` (`hidden`, `target="StickyHeader"`). Props are declared in `src/FlashListProps.ts`.

**Depends on:** [recycler-core](recycler-core.md) (`RecyclerViewManager` offsets, engaged indices, `ViewHolder`), [layout-managers](layout-managers.md) (`getLayout(i).y`/`height`), [scrolling](scrolling.md) (scroll handler, `inverted`) · **Used by:** [public-api](public-api.md)

## Public props (verified names)

| Prop | Default | Consumed in |
|---|---|---|
| `stickyHeaderIndices` | none | `RecyclerView.tsx` → `StickyHeaders` (sorted copy in `sortedIndices`) |
| `stickyHeaderConfig.offset` | `0` | `stickyHeaderOffset`: header `top`, push math, spacer `marginTop` in `viewToMeasureBoundedSize` |
| `stickyHeaderConfig.useNativeDriver` | `true` | the `Animated.event` that drives `scrollY` |
| `stickyHeaderConfig.backdropComponent` | none | `useSecondaryProps` `renderStickyHeaderBackdrop` |
| `stickyHeaderConfig.zIndex` | `2` | `StickyHeaders` container style (`stickyHeaderZIndex`) |
| `stickyHeaderConfig.hideRelatedCell` | `false` | `RecyclerView` `currentStickyIndex` state → `ViewHolderCollection` → `ViewHolder hidden` (opacity 0) |
| `onChangeStickyIndex(current, previous)` | none | wrapped in `RecyclerView.tsx` |

There is no `stickyHeaderConfig.background` prop; the backdrop is `backdropComponent`. The `StickyHeaderBackgroundComponent` name in commit 0cf8775c's title is stale.

## How it works

There are two separate channels:

- **JS state.** On each scroll event, `compute()` picks *which* header is current and where the push starts, storing them in `stickyHeaderState`.
- **`Animated` value.** `scrollY` is driven natively when `useNativeDriver` is on. It moves the header with an interpolation, so the push animation runs without a JS round-trip per frame.

### Flow: scroll → current sticky index computed → header rendered/translated

1. Render setup in `RecyclerView.tsx`:
   - A `stickyHeaders` element is memoized only when `data` and `stickyHeaderIndices` are both non-empty. It throws `ErrorMessages.stickyHeadersNotSupportedForHorizontal` if `horizontal` is set.
   - When `stickyHeaders` exists, `animatedEvent` becomes `Animated.event([{ nativeEvent: { contentOffset: { y: scrollY } } }], { useNativeDriver, listener: onScrollHandler })`. Otherwise it is the plain `onScrollHandler`.
2. `onScrollHandler` first updates the manager's scroll offset synchronously (`velocityTracker.computeVelocity` → `updateScrollOffset`). Then it calls `stickyHeaderRef.current?.reportScrollEvent(...)`.
3. `reportScrollEvent` (exposed through `useImperativeHandle`) ignores its event argument and calls `compute()`. `compute()` works in item-relative space:
   - Reads `recyclerViewManager.getLastScrollOffset()`, which is already relative to the first item.
   - `findCurrentStickyIndex` binary-searches `sortedIndices` for the last header with `getLayout(i).y <= offset + stickyHeaderOffset`.
   - The next header counts only if it is `<= getEngagedIndices().endIndex`. Otherwise it is set to `-1` and there is no push.
   - The push start is `pushStartsAt = (nextY + firstItemOffset) - currentHeight - stickyHeaderOffset`. This value is in absolute `contentOffset` space, the same space as `scrollY`.
   - If anything changed, it calls `setStickyHeaderState` and, on an index change, `onChangeStickyIndex(newIndex)`.
4. `compute` also runs in a `useEffect` whenever its deps change: layout/data via `recyclerViewManager`, `sortedIndices`, or the offset.
5. Rendering:
   - `translateY = scrollY.interpolate([pushStartsAt, pushStartsAt + h] → [0, -h], clamp)`.
   - When `offset > 0`, `opacity` also fades 1 → 0 over the same range.
   - The `CompatAnimatedView` is `position: absolute; top: offset; zIndex`. Inside it, a `ViewHolder` with `target="StickyHeader"` renders the item with `position: relative` and a zero layout. `renderItem` receives `target` and can style the stuck copy differently.
6. The header is rendered **outside** `CompatScrollView`, after the backdrop, inside the outer container. Stacking order from back to front: scroll content, then `renderStickyHeaderBackdrop` (`position: absolute; inset: 0; pointerEvents: none`), then the sticky header.

## Key types and entry points

| Symbol | File | Role |
|---|---|---|
| `StickyHeaders` | `src/recyclerview/components/StickyHeaders.tsx` | Component: state, `compute`, interpolations, render |
| `StickyHeaderRef.reportScrollEvent` | same | Imperative hook called from `onScrollHandler` |
| `findCurrentStickyIndex` | same (module-private) | Binary search over sorted header indices by `y` |
| `StickyHeaderState` | same | `{ currentStickyIndex, pushStartsAt }` |
| `scrollY` (`Animated.Value`) | `src/recyclerview/RecyclerView.tsx` | Native-driven `contentOffset.y` |
| `renderStickyHeaderBackdrop` | `src/recyclerview/hooks/useSecondaryProps.tsx` | Backdrop layer, counter-transformed when `inverted` |
| `RenderTarget` / `RenderTargetOptions.StickyHeader` | `src/FlashListProps.ts` | Tells `renderItem` it is rendering the sticky copy |
| `PlatformConfig.isRN083OrAbove` | `src/native/config/PlatformHelper*.ts`, `versionCheck.ts` | Gates unmounting when no header is active |

## Invariants and gotchas

- **The sticky header is a second render of the item.** The original cell stays in the scroll view and is still recycled, so `renderItem` runs twice for that index. `hideRelatedCell` only sets the cell's `opacity` to 0; it still takes space and still receives layout.
- **`offset` also shifts content.** `viewToMeasureBoundedSize` gets `marginTop: stickyHeaderOffset`, so items start `offset` px lower and `firstItemOffset` includes it (#1953).
- **Next-header push only within engaged indices.** If the next header is beyond `getEngagedIndices().endIndex` (not laid out yet), `pushStartsAt` is `MAX_SAFE_INTEGER` and no push is animated until it is engaged.
- **No active header (`-1`).** On RN ≥ 0.83 (and web/default) the component returns `null`. On older RN it stays mounted with an empty container. The `-1` state with height 0 produces a degenerate `[p, p]` interpolation range. On RN 0.83 that range produced `translateY: -Infinity` and left the whole list invisible (#2069).
- **Invalid indices are silently ignored.** If the largest sticky index is `>= getDataLength()` (`lengthInvalid`), `compute` does nothing and the header freezes at its last state. The render also guards `currentStickyIndex < data.length` (crash on data removal, #1718).
- **`stickyHeaderIndices` is sorted on a copy.** The caller's array is never mutated (commit 7345b92e).
- **`onChangeStickyIndex`'s `previous` argument is `RecyclerView`'s `currentStickyIndex` state.** That state is only updated when `hideRelatedCell` is true. Read from code: with `hideRelatedCell: false`, `previous` stays `-1`.
- **With `offset ≠ 0`, `compute` re-sets state on every scroll event.** It compares the raw `newPushStartsAt` against the stored `pushStartsAt`, which has `stickyHeaderOffset` already subtracted, so the two never match. The memoized interpolations stay the same, but `StickyHeaders` re-renders on every scroll event (read from code).
- **`useNativeDriver: false`** makes `scrollY` a JS-driven value. The push animation then depends on JS-thread frames. The option exists for custom scroll components that can't drive native events (**Inferred**).
- **Horizontal throws at render.** The check only runs when `data` and `stickyHeaderIndices` are both non-empty.
- **Inverted:** the outer container is flipped with `getInvertedTransformStyle`. The sticky `ViewHolder` (via its `inverted` prop) and the backdrop get the counter-transform so they read upright (commit 8c008171, #2131). Because the container is flipped, the "top" of the list, and so the stuck header, appears at the visual bottom (**Inferred**, not covered by unit tests; see fixture `InvertedTest.tsx`).
- **`getLastScrollOffset` is already relative to the first item.** It excludes `firstItemOffset`, which includes `ListHeaderComponent` height and `offset`. Comparing it with `getAbsoluteLastScrollOffset` or with `scrollY` without converting is the classic bug here. #2105 (commit f07b9b35) fixed premature activation on Fabric when content sits above the list.

## Design decisions

- **JS picks the header and native moves it.** Index selection needs layouts, which live in JS. The push is an `Animated` interpolation on native-driven `scrollY`, so it stays smooth between JS updates. This has been the FlashList v2 design since 5e9a6e01 (#1617). **Inferred** rationale.
- **Fade when `offset > 0`.** #1953: "when offset > 0, header fades as it's pushed". **Inferred:** sliding up would overlap the app's fixed top bar above the offset line.
- **Configurable `zIndex` (default 2).** d6c9513a (#2311) made the previously hard-coded `2` configurable so callers can control stacking against their own overlays. The default stays `2` for backward compatibility.
- **Unmount on `-1` only on RN ≥ 0.83.** #2069 returned `null` to fix the invisible-list bug on 0.83. #2076 (99ff4d8c) limited that to RN ≥ 0.83 because unmounting and remounting caused header flashing on older RN.
- **Backdrop as a separate layer** above content and below the header, for blur or background effects (#1953).

## Where to change things

| To… | Start at |
|---|---|
| Change which header is active / boundary rules | `StickyHeaders.tsx` `compute`, `findCurrentStickyIndex` |
| Change push or fade animation | `StickyHeaders.tsx` `translateY` / `opacity` `useMemo` |
| Add a `stickyHeaderConfig` option | `FlashListProps.ts` → `RecyclerView.tsx` destructure/defaults → `StickyHeaderProps` (pattern: d6c9513a) |
| Change backdrop rendering | `useSecondaryProps.tsx` `renderStickyHeaderBackdrop` and its placement in `RecyclerView.tsx` |
| Fix `onChangeStickyIndex` arguments | the `onChangeStickyIndex` wrapper inside the `stickyHeaders` `useMemo` in `RecyclerView.tsx` |
| Hide/show the source cell | `ViewHolderCollection.tsx` (`hidden`), `ViewHolder.tsx` (`opacity`) |
| Support horizontal | `RecyclerView.tsx` throw site, plus `x`/`width` and `translateX` throughout `StickyHeaders.tsx` |

## Tests

- `src/__tests__/StickyHeaders.test.tsx` renders `StickyHeaders` against a mocked `RecyclerViewManager` (`createMockRecyclerViewManager`: `getLastScrollOffset`, `getLayout`, `tryGetLayout`, `getEngagedIndices`, `firstItemOffset`). It covers index selection with and without offset, `pushStartsAt`, callback sequencing, edge cases (before the first header, next header beyond engaged indices, variable heights, negative offset), `top`, and `zIndex`.
- `src/__tests__/RecyclerView.test.tsx`: `"Sticky header config"` (zIndex via the full `FlashList`) and `"Sticky headers with content above FlashList"` (#2105 regression).
- Not covered: `inverted`, `hideRelatedCell`, `backdropComponent`, the RN-version unmount branch, and native-driver behavior. Check those on a device with `fixture/react-native/src/StickyHeaderExample.tsx` and `InvertedTest.tsx`.

```bash
yarn test StickyHeaders --forceExit
yarn test RecyclerView.test -t "Sticky" --forceExit
# If corepack's Yarn 4 rejects the lockfile: node_modules/.bin/jest StickyHeaders --forceExit
```
