<!-- index-codebase: generated from 527d7678 on 2026-10-02. Code wins when it disagrees with this doc. -->
# FlashList: Agent Index

## What this is

`@shopify/flash-list` is a FlatList-compatible list component for React Native. It recycles cells instead of mounting one per item. v2 (current, `package.json` 2.3.3) is a JS-only rewrite for React Native's **new architecture** (Fabric) only, plus react-native-web. It needs no size estimates: items are measured synchronously in `useLayoutEffect` and repositioned before paint. It is a single TypeScript library package (`src/` compiled to `dist/` with `tsc`). Around it sit a React Native fixture app, a web fixture, a Docusaurus documentation site and Detox e2e screenshot tests. The README and the [docs site](https://shopify.github.io/flash-list/docs/) cover usage; this file covers internals.

## Architecture in brief

`FlashList` **is** `RecyclerView` (`src/FlashList.ts` is one re-export line). One mutable `RecyclerViewManager` per mounted list holds every piece of engine state. React state is only used as a "re-render now" trigger.

| Concept | Owner | One line |
|---|---|---|
| Layout manager | `src/recyclerview/layout-managers/` | One `RVLayout` (x, y, w, h) per **data index**. Unmeasured items get per-`getItemType` running-average estimates (default 200). Picks Linear, Grid or Masonry from props |
| Engaged indices | `helpers/EngagedIndicesTracker.ts` | The render window: viewport + `drawDistance` buffer, skewed in the scroll direction and projected by velocity |
| Render stack | `RenderStackManager.ts` | Maps **recycle keys** (React keys, not `keyExtractor` output) to indices. A key is only reused for an item of the same type |
| ViewHolder | `ViewHolder.tsx` / `ViewHolderCollection.tsx` | One absolutely positioned cell per render-stack entry |
| Controller | `hooks/useRecyclerViewController.tsx` | The `FlashListRef` API: scrollTo*, initial scroll, offset correction (MVCP) |

### Primary flows

1. **Mount.**
   1. First render: no cells.
   2. Layout effect A in `RecyclerView.tsx` measures the container and the first-child marker, giving the window size and `firstItemOffset` (header height). It then creates the layout manager.
   3. Layout effect B creates estimated layouts. `renderProgressively` mounts the visible items in geometric batches, each measured with `measureItemLayout` and fed back through `modifyChildrenLayout`.
   4. Once every visible item is measured, `commitLayout` lifts the `opacity: 0` gate.
   5. `onLoad` fires one animation frame later, and only then are the buffer items rendered. Details: [recycler-core](areas/recycler-core.md).
2. **Scroll.**
   1. `onScrollHandler` → `VelocityTracker` → `RecyclerViewManager.updateScrollOffset` → `EngagedIndicesTracker`. If the range changed, `RenderStackManager.sync` reassigns keys.
   2. `setRenderId` re-renders. Effect B re-measures all mounted cells, and the layout manager repositions them. If anything moved, the list re-renders again before paint (up to 40 times); otherwise it commits.
   3. The same handler also drives sticky headers, `onEndReached`/`onStartReached` and viewability.
3. **Data change / prepend.**
   1. `processDataUpdate` runs during render in a `useMemo([data])`. It resizes the index-keyed layouts and re-syncs keys.
   2. After commit, `applyOffsetCorrection` (iOS/Android only) shifts the scroll offset so the first visible item (anchored by `keyExtractor`) stays put. This is `maintainVisibleContentPosition`, on by default. Details: [scrolling](areas/scrolling.md).

## Map

| Area | Responsibility | Paths | Doc |
|---|---|---|---|
| public-api | Exports, prop and ref types, new-arch guard, errors/warnings, `PlatformConfig`, shipped `jestSetup.js`, packaging | `src/index.ts`, `src/FlashList*.ts`, `src/errors/`, `src/native/config/`, `src/isNewArch.ts`, `package.json` | [areas/public-api.md](areas/public-api.md) |
| recycler-core | Render engine: engaged indices, keys/recycling, measure → re-render loop, `onLoad`, bounds, nested lists, public state hooks | `src/recyclerview/*.tsx?`, `helpers/`, most of `hooks/`, `utils/measureLayout*` | [areas/recycler-core.md](areas/recycler-core.md) |
| layout-managers | Positions, estimates, grid row equalization, masonry columns, visible-range binary search | `src/recyclerview/layout-managers/`, `src/utils/AverageWindow.ts`, `utils/findVisibleIndex.ts` | [areas/layout-managers.md](areas/layout-managers.md) |
| scrolling | Ref API, `scrollToIndex`, initial scroll, MVCP / offset correction, inverted, RTL | `hooks/useRecyclerViewController.tsx`, `components/ScrollAnchor.tsx`, `src/FlashListRef.ts`, `utils/adjustOffsetForRTL.ts`, `utils/getInvertedTransformStyle.ts` | [areas/scrolling.md](areas/scrolling.md) |
| viewability | `onViewableItemsChanged`, thresholds, `minimumViewTime` timers | `src/recyclerview/viewability/` | [areas/viewability.md](areas/viewability.md) |
| sticky-headers | Sticky header overlay (a second render of the item), push animation, `stickyHeaderConfig` | `components/StickyHeaders.tsx` | [areas/sticky-headers.md](areas/sticky-headers.md) |
| benchmark | Public JS-FPS benchmark hooks and auto-scroll | `src/benchmark/` | [areas/benchmark.md](areas/benchmark.md) |
| Tooling (no area doc) | Fixture apps, e2e, docs site, CI, release, agent workflows | `fixture/`, `documentation/`, `website/`, `.github/`, `.claude/` | [contributing.md](contributing.md) |

## Where to look

| Task or question | Start at |
|---|---|
| Add a prop | [public-api → Where to change things](areas/public-api.md#where-to-change-things) |
| Add a ref method | `src/FlashListRef.ts` + `handlerMethods` in `useRecyclerViewController.tsx` |
| Blank cells / white bands while scrolling | `EngagedIndicesTracker.updateScrollOffset`, `PlatformConfig.defaultDrawDistance` ([recycler-core](areas/recycler-core.md)) |
| Items overlap or jump after a size change | `RVLayoutManager.modifyLayout`, subclass `recomputeLayouts`, `useLayoutState` ([layout-managers](areas/layout-managers.md)) |
| Grid row heights wrong | `RVGridLayoutManagerImpl.processAndReturnTallestItemInRow` |
| Masonry columns | `RVMasonryLayoutManagerImpl`, `optimizeItemArrangement` |
| `scrollToIndex` lands in the wrong place | `useRecyclerViewController` ([scrolling](areas/scrolling.md)) |
| Content jumps when prepending (chat lists) | `applyOffsetCorrection`, `ScrollAnchor`, `keyExtractor` ([scrolling](areas/scrolling.md)) |
| Item state leaks between rows | Recycling. `useRecyclingState`, `RenderStackManager.sync` ([recycler-core](areas/recycler-core.md)) |
| "Exceeded max renders without commit" warning | `RenderTimeTracker`, effect B in `RecyclerView.tsx`; usually duplicate keys |
| `onEndReached` misfires | `hooks/useBoundDetection.ts` |
| `onViewableItemsChanged` timing | `ViewabilityHelper` ([viewability](areas/viewability.md)) |
| Sticky header issues | `components/StickyHeaders.tsx` ([sticky-headers](areas/sticky-headers.md)) |
| Platform-specific defaults (web, Android) | `src/native/config/PlatformHelper*.ts` |
| Users' Jest setup broke | `jestSetup.js` (mocks `dist/recyclerview/utils/measureLayout`) |
| Reproduce a bug on device | Fixture screens in `fixture/react-native/src` (register in `NavigationTree.tsx` + `ExamplesScreen.tsx`); `.claude/skills/fix-github-issue`, `agent-device` |
| Why a design is the way it is | [decisions.md](decisions.md) |
| Config surfaces, platforms, limits, security | [using.md](using.md) |
| Build, test, CI, release | [contributing.md](contributing.md) |

## Dev loop

```bash
yarn                       # Yarn classic 1.x (v1 lockfile)
yarn test --forceExit      # Jest on src/ directly, no build needed
yarn test <pattern> --forceExit
yarn lint && yarn type-check
yarn build                 # tsc -b → dist/, which the fixture app consumes
```

If `yarn` is a 2+/4.x binary (global install or corepack), it rejects the v1 lockfile with "package doesn't seem to be present in your lockfile". Use Yarn 1, or run Jest directly with `node_modules/.bin/jest <pattern> --forceExit`. Full details: [contributing.md](contributing.md).

## Conventions and traps

- **`estimatedItemSize` does not exist in v2.** Nothing reads it and nothing warns; it leaks onto the ScrollView. Don't add it back (`CLAUDE.md`). Two stale uses remain: `fixture/react-native/src/twitter/Twitter.tsx` and `documentation/docs/fundamentals/performant-components.md`.
- **`dist/` is gitignored and not rebuilt on branch switch.** The fixture links the package root (`link:../../`) and `main` is `dist/index.js`, so run `yarn build` before device testing.
- **Platform files** (`*.ios.ts`, `*.android.ts`, `*.web.ts`) are chosen by the bundler. Repo Jest resolves the generic `.ts` (`shared/testing/resolver.js`), so unit tests see no offset correction.
- **`jestSetup.js` ships to users.** Renaming `src/recyclerview/utils/measureLayout.ts` or its exports breaks their tests.
- **Tests that render a list must mock `measureLayout`** (see the top of `src/__tests__/RecyclerView.test.tsx`).
- **Test files are not type-checked**: `tsconfig.json` excludes them, so `yarn type-check` skips them.
- **Unknown props are forwarded to the ScrollView.** Engine props are read from `props` by `RecyclerViewManager`, not destructured.
- Commits use conventional format (`fix(scope): …`). Agent-raised PRs must carry no AI attribution (`CLAUDE.md`, `.claude/skills/raise-pr`).

## Glossary

| Term | Meaning | Defined in |
|---|---|---|
| Engaged indices | Indices that are rendered: viewport plus buffer. Not the same as visible | `helpers/EngagedIndicesTracker.ts` |
| Visible indices | Viewport only, no buffer (`computeVisibleIndices`) | same |
| Recycle key | React key of a cell ("0", "1", …). Not the `keyExtractor` output | `RenderStackManager.ts` |
| stableId | `keyExtractor` output, or `String(index)` | `RecyclerViewManager.getDataKey` |
| ViewHolder | The cell wrapper (Android RecyclerView term), not the item | `ViewHolder.tsx` |
| Draw distance | Buffer size in px around the viewport (250 native, 500 web) | `PlatformHelper*.ts` |
| firstItemOffset | Offset of the first item in the scroll content (header height + sticky offset). Layout x/y exclude it | `RecyclerView.tsx`, `RecyclerViewManager.ts` |
| Bounded size / window size | Cross-axis size, measured from a zero-size marker view after the header, so it excludes padding | `RecyclerView.tsx` |
| Commit | The layout is final for this pass (`commitLayout`), not a React commit | `ViewHolderCollection.tsx` |
| `requiresRepaint` | Layout manager asks for one more render+measure before commit; no native repaint involved | `LayoutManager.ts` |
| Offset projection | Render window centred on `offset + velocity × avgRenderTime` | `EngagedIndicesTracker.ts` |
| Offset correction / MVCP | Compensating scroll offset so the anchored first visible item doesn't move | `useRecyclerViewController.tsx` |
| Span | Columns an item covers in grid/masonry (`overrideItemLayout`) | `LayoutManager.ts` |
| `RenderTarget` | `"Cell"` or `"StickyHeader"` (`"Measurement"` is declared but unused in v2) | `FlashListProps.ts` |
| Pending child layout | Nested FlashLists block the parent's commit until their first commit | `RecyclerViewContextProvider.ts` |

## Further reading

- [`CLAUDE.md`](../../CLAUDE.md): project rules for agents (estimatedItemSize, PR token, commit format, skills table).
- `.claude/skills/`: hand-written workflows: `fix-github-issue`, `review-and-test`, `agent-device`, `raise-pr`, `triage-issue`, `upgrade-react-native`, `analyze-feedback`.
- [decisions.md](decisions.md), [using.md](using.md), [contributing.md](contributing.md).
- [project-health.md](project-health.md): maintenance activity, backing, contribution norms (from `index-github`).
- User docs source: `documentation/docs/` (`v2-changes.md`, `v2-migration.md`, `known-issues.md`, `fundamentals/usage.md` is the prop/ref reference). Live: https://shopify.github.io/flash-list/docs/.
- `CONTRIBUTING.md`, `RELEASE.md`. Several commands in `CONTRIBUTING.md` are stale; [contributing.md](contributing.md) has the corrections.
