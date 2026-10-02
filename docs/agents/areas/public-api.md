<!-- index-codebase: generated from 527d7678 on 2026-10-02. Code wins when it disagrees with this doc. -->
# Public API and platform shims

**Purpose:** Defines what `@shopify/flash-list` exports, the props and ref types users code against, the new-architecture import guard, user-facing error and warning strings, per-platform constants (`PlatformConfig`), the Jest mock shipped to users, and packaging (`package.json`, `tsconfig.json`). It does **not** cover how the list renders, recycles, lays out or scrolls. Those live in the sibling areas linked below.

**Paths:** `src/index.ts`, `src/FlashList.ts`, `src/AnimatedFlashList.ts`, `src/FlashListProps.ts`, `src/FlashListRef.ts`, `src/isNewArch.ts`, `src/errors/`, `src/native/config/`, `src/recyclerview/RecyclerViewProps.ts`, `src/recyclerview/hooks/useSecondaryProps.tsx`, `src/recyclerview/components/Compat*.ts`, `package.json`, `jestSetup.js`, `tsconfig.json`, `shared/tsconfig.base.json`, `documentation/docs/`

**Depends on:** [recycler-core](recycler-core.md) (the actual component) · **Used by:** app developers, `fixture/react-native`, `fixture/web`, [benchmark](benchmark.md), and every sibling area (they read `PlatformConfig`, `ErrorMessages` and `RecyclerViewProps`)

## How it works

### What does `FlashList` actually point to?
`src/FlashList.ts` is one line: `export { RecyclerView as FlashList } from "./recyclerview/RecyclerView"`. There is no wrapper class. `FlashList` **is** `RecyclerView`: a `React.memo(forwardRef(RecyclerViewComponent))` whose `displayName` is `"FlashList"`. The v1 class component was deleted in `5e9a6e01` (FlashList v2, #1617) and `9d1aef13` ("Remove most of the JS code and tests"). The ref type is the `FlashListRef<T>` interface. It is not a class instance, so `useRef<FlashList<T>>` from v1 no longer type-checks.

`RecyclerViewProps` is only a type alias: `export type { FlashListProps as RecyclerViewProps }`. The props are defined once, in `src/FlashListProps.ts`, and internal code imports them under the `RecyclerViewProps` name.

### What happens at import time?
`src/index.ts` re-exports everything, then runs `if (!isNewArch()) throw new Error(ErrorMessages.flashListV2OnlySupportsNewArchitecture)`. `isNewArch()` (cached in a module variable) returns true when `global.nativeFabricUIManager` exists, when `global.__turboModuleProxy` exists, or when `Platform.OS === "web"`. If the probe itself throws, it also returns true. On the old architecture the import throws, so nothing renders.

### How are props routed?
In `RecyclerViewComponent` (`src/recyclerview/RecyclerView.tsx`), a fixed list of props is destructured. Everything else goes into `...rest`, which is spread onto the internal ScrollView, followed by `{...overrideProps}`, which wins. Props that configure the list engine (`drawDistance`, `numColumns`, `masonry`, `getItemType`, `keyExtractor`, `maxItemsInRecyclePool`, etc.) are **not** destructured. They reach the engine through `useRecyclerViewManager` → `RecyclerViewManager.updateProps(props)` (`propsRef`) and also leak to the ScrollView as unknown props, which do nothing there.

`useSecondaryProps` builds the parts that are not the list itself:
- `refreshControl`: the user's `refreshControl` if given, else a `<RefreshControl>` when `onRefresh` is set (uses `refreshing` and `progressViewOffset`).
- Header and footer: wrapped in `CompatView` with `List*ComponentStyle` and the inverted counter-transform.
- Empty state: rendered only when `data` is empty. It is rendered *in addition to* the footer, not instead of it.
- Sticky-header backdrop.
- `CompatScrollView`: `Animated.createAnimatedComponent(renderScrollComponent ?? Animated.ScrollView)`. A plain function component passed as `renderScrollComponent` is wrapped in `forwardRef`.

### How is masonry exposed?
As a prop: `masonry?: boolean` on `FlashList`, plus `optimizeItemArrangement` (defaults to `true` in `RecyclerViewManager`). `MasonryFlashList` no longer exists in v2 (`src/index.ts` does not export it). `getColumnFlex` was dropped. `RecyclerViewManager.getLayoutManagerClass` picks the layout manager in this order: masonry → `RVMasonryLayoutManagerImpl`; `numColumns > 1` → grid; otherwise linear. See [layout-managers](layout-managers.md).

## Key types and entry points

| Symbol | File | Role |
|---|---|---|
| `FlashList` | `src/FlashList.ts` | Alias of `RecyclerView` (memo + forwardRef) |
| `AnimatedFlashList` | `src/AnimatedFlashList.ts` | `Animated.createAnimatedComponent(FlashList)`, typed as `ComponentType<FlashListProps<any>>`, so it loses the item generic |
| `FlashListProps<T>` | `src/FlashListProps.ts` | The full prop surface. Extends `Omit<ScrollViewProps, "maintainVisibleContentPosition">` |
| `RecyclerViewProps<T>` | `src/recyclerview/RecyclerViewProps.ts` | Type alias of `FlashListProps`, used internally |
| `FlashListRef<T>` | `src/FlashListRef.ts` | Imperative API. Implemented by `handlerMethods` in `useRecyclerViewController` ([scrolling](scrolling.md)) |
| `ListRenderItemInfo`, `RenderTarget`, `RenderTargetOptions` | `src/FlashListProps.ts` | `renderItem` args. `target` is `"Cell" \| "StickyHeader" \| "Measurement"` |
| `OverrideProps` | `src/FlashListProps.ts` | `overrideProps` type. Carries the hidden `initialDrawBatchSize` |
| `isNewArch()` | `src/isNewArch.ts` | Fabric/TurboModule/web probe used by the import guard |
| `ErrorMessages`, `WarningMessages` | `src/errors/` | Every user-facing throw/`console.warn` string |
| `PlatformConfig` | `src/native/config/PlatformHelper{,.ios,.android,.web}.ts` | Per-platform constants, selected by bundler extension resolution |
| `isRN083OrAbove` | `src/native/config/versionCheck.ts` | Reads `Platform.constants.reactNativeVersion` |
| `useSecondaryProps` | `src/recyclerview/hooks/useSecondaryProps.tsx` | Header, footer, empty state, refresh control, scroll component |
| Hooks and helpers re-exported from `index.ts` | `useLayoutState`, `useRecyclingState`, `useMappingHelper`, `useFlashListContext`, `LayoutCommitObserver`, `ViewToken` | See [recycler-core](recycler-core.md) and [viewability](viewability.md) |
| Benchmark exports | `useBenchmark`, `useFlatListBenchmark`, `useDataMultiplier`, `JSFPSMonitor`, `autoScroll`, `Cancellable` | See [benchmark](benchmark.md) |

### `PlatformConfig` values

| Field | default (`.ts`) | `.ios` | `.android` | `.web` | Read by |
|---|---|---|---|---|---|
| `defaultDrawDistance` | 250 | 250 | 250 | 500 | `EngagedIndicesTracker.drawDistance` |
| `supportsOffsetCorrection` | false | true | true | false | `useRecyclerViewController` |
| `trackAverageRenderTimeForOffsetProjection` | false | false | true | false | `RenderTimeTracker` |
| `isRN083OrAbove` | true | runtime check | runtime check | true | `StickyHeaders` (returns null when no header is active, #2076) |
| `invertedTransformStyle` | `scaleY:-1` | `scaleY:-1` | `rotate:180deg` | `scaleY:-1` | `getInvertedTransformStyle` (#2131, #751) |

## Invariants and gotchas

- **`estimatedItemSize` and the other v1 estimate props have no runtime handling at all.** They are not in `FlashListProps` (TypeScript rejects them as excess properties), nothing in `src` reads or warns about them, and at runtime they go into `...rest` → ScrollView and are ignored. The same goes for `estimatedListSize`, `estimatedFirstItemOffset`, `onBlankArea`, `disableAutoLayout` and `disableHorizontalListHeightMeasurement`. The docs call them "Deprecated", but the code has removed them. Do not add them back (see `CLAUDE.md`).
- **Only three runtime warnings exist** (`WarningMessages`):
  - `keyExtractorNotDefinedForMVCP`: `onStartReached` set without `keyExtractor`, from `RecyclerViewManager.checkPropsAndWarn`, in the constructor only.
  - `keyExtractorNotDefinedForAnimation`: from `prepareForLayoutAnimationRender`.
  - `exceededMaxRendersWithoutCommit`: 40 renders without a commit (`RenderTimeTracker`).
- **Prop combinations that throw:**
  - `masonry` + `horizontal`
  - `numColumns > 1` + `horizontal`
  - `stickyHeaderIndices` + `horizontal`
  - toggling `horizontal` after mount: remount with a new `key` instead
  - setting both viewability threshold types
- **Some props stick after you unset them.** `updateProps` does `props.drawDistance ?? previous` (same for `overrideProps.initialDrawBatchSize`), so changing `drawDistance` back to `undefined` keeps the last value instead of restoring the platform default.
- **A new prop that is not destructured in `RecyclerViewComponent` is forwarded to the ScrollView.** Destructure it if passing it down would cause trouble.
- **`overrideProps` is spread last onto the ScrollView**, so it can override `onScroll`, `ref` or `maintainVisibleContentPosition` and break the list. That is intentional as an escape hatch (see its JSDoc).
- **`jestSetup.js` is shipped to users** (`package.json` `files`) and mocks `@shopify/flash-list/dist/recyclerview/utils/measureLayout`. It forces `measureParentSize` and `measureFirstChildLayout` to 400×900 and every `measureItemLayout` to 100×100, so tests render a bounded window instead of mounting everything. **Moving or renaming `src/recyclerview/utils/measureLayout.ts` or those three exports breaks every user's Jest setup.** The `x`/`y` in its `measureParentSize` mock are leftovers: the real function dropped them in #2105.
- **Platform files are resolved by the bundler, not at runtime.** `tsc` emits `PlatformHelper.ios.js` and the other variants into `dist/`, and Metro or react-native-web picks one. In repo Jest, `shared/testing/resolver.js` lists `.ts` before any `.ios.*`, so tests always get the generic `PlatformHelper.ts`: no offset correction, draw distance 250.
- **`measureLayout` also has a web variant** (`src/recyclerview/utils/measureLayout.web.ts`, DOM-based). Keep its exports in sync with the native one.
- **The fixture app consumes `dist/`**: `fixture/react-native/package.json` has `"@shopify/flash-list": "link:../../"`, and `main` is `dist/index.js`. Run `yarn build` after editing `src/`.
- **`FlashList` is `React.memo`'d.** Unmemoized inline props re-render everything (`documentation/docs/fundamentals/usage.md` point 6).

## Design decisions

- **One component, no wrapper.** The v2 rewrite (`5e9a6e01`, #1617) moved to a JS-only `RecyclerView` and aliased it as `FlashList`. The v1 `recyclerlistview` dependency was removed in `f9c635a9`. Evidence: `src/FlashList.ts`, `README.md` ("moving to a JS-only solution").
- **Hard fail on the old architecture.** v2 relies on synchronous layout measurement in `useLayoutEffect`, which Fabric provides (README: "designed to be new architecture only and will not run on old architecture"). Web is allowed through explicitly in `isNewArch`.
- **Masonry became a prop** so it can share the layout-manager pipeline and `overrideItemLayout` (`documentation/docs/v2-changes.md`, "New features").
- **Android inverts with `rotate(180deg)` instead of `scaleY:-1`**, for performance (#751), at the cost of the scrollbar appearing on the left (#2131, JSDoc on `inverted`).
- **Errors and warnings were centralized** into `src/errors/` in `1d665c01` (it replaced `CustomError`/`ExceptionList`).
- **Peer deps are `*`** (`react`, `react-native`, `@babel/runtime`). **Inferred:** the real compatibility gate is the runtime `isNewArch()` check, not semver.

## Breaking changes and limitations from v1 (verified against code)

| Doc claim (`documentation/docs/`) | Code verdict |
|---|---|
| New architecture required (`v2-migration.md`) | True: `src/index.ts` throws |
| Estimate props removed (`v2-changes.md` "Deprecated") | Removed, with no warning (see gotchas) |
| `MasonryFlashList` → `masonry` prop; no `getColumnFlex` | True: no such export |
| `overrideItemLayout` reads only `span` | True: `LayoutManager` calls it with a `SpanSizeInfo` |
| Ref type is `FlashListRef`; `CellContainer` no longer exported | True: `src/index.ts` |
| `maintainVisibleContentPosition` on by default | True: `shouldMaintainVisibleContentPosition()` returns `!disabled` |
| Ref has `getVisibleIndices` (`v2-changes.md`, `usage.md` "returns `number[]`") | **Wrong:** the method is `computeVisibleIndices()` and returns `{startIndex, endIndex}` (the `FlashListRef.ts` JSDoc example also uses the old name) |
| `onBlankArea` documented as a prop (`usage.md`) | **Stale:** not in `FlashListProps`, and `v2-changes.md` says it is unsupported |
| `onCommitLayoutEffect` "called before layout is committed" (`v2-changes.md`) | The JSDoc says "when the layout is committed". It fires from `ViewHolderCollection`'s layout effect |
| `numColumns` JSDoc "masonry layouts are not supported" | Stale v1 text |
| Known issues: RTL horizontal padding, fixed height for horizontal lists with headers, reorder movement caused by MVCP (`known-issues.md`) | Consistent with the code. Workaround for the last one: `maintainVisibleContentPosition={{disabled:true}}` |
| Expo Go "from SDK 46 upwards" (`index.mdx`) | **Inferred stale:** it predates v2's new-architecture requirement |

Undocumented ref methods: `clearLayoutCacheOnUpdate`, `getAbsoluteLastScrollOffset`, `getChildContainerDimensions`, `computeVisibleIndices`.

## Where to change things

| To… | Start at |
|---|---|
| Add a prop end to end | 1. Add the type with JSDoc to `FlashListProps` in `src/FlashListProps.ts`. 2. Read it where it belongs: list-level engine config via `this.propsRef` in `RecyclerViewManager` (re-read every render through `updateProps`); render or ScrollView concerns by destructuring in `RecyclerViewComponent` or `useSecondaryProps`. 3. Document it in `documentation/docs/fundamentals/usage.md` under "# Props" (plus `v2-changes.md` if notable). 4. Add a test in `src/__tests__/RecyclerView.test.tsx` (see its "Sticky header config" block for the pattern). 5. Optionally add a fixture screen in `fixture/react-native/src` (register it in `NavigationTree.tsx` and `ExamplesScreen.tsx`) |
| Add a ref method | `FlashListRef` in `src/FlashListRef.ts` and `handlerMethods` in `src/recyclerview/hooks/useRecyclerViewController.tsx` |
| Add a public export | `src/index.ts`. Keep the `isNewArch` guard at the bottom |
| Add a user-facing error or warning | `src/errors/ErrorMessages.ts` or `WarningMessages.ts`, then throw/`console.warn` at the call site |
| Change a per-platform default | All four `src/native/config/PlatformHelper*.ts` files. They must keep the same shape |
| Gate behaviour on the RN version | `src/native/config/versionCheck.ts`, exposed via `PlatformConfig` |
| Change header, footer, empty or refresh rendering | `src/recyclerview/hooks/useSecondaryProps.tsx` and the JSX order in `RecyclerView.tsx` |
| Change what users' Jest sees | `jestSetup.js` and `documentation/docs/guides/testing.md` |
| Change what gets published | `package.json` `files`/`main`/`types`. `tsconfig.json` emits `src` → `dist` (ES2015 modules, declarations, source maps; tests excluded) |

## Tests

- `src/__tests__/RecyclerView.test.tsx` imports `FlashList` from `..` (the real entry, so the new-arch guard runs) and mocks `../recyclerview/utils/measureLayout` the same way `jestSetup.js` does.
- `src/__tests__/LayoutCommitObserver.test.tsx` also imports from `..`.
- Nothing tests the export list, `isNewArch`, `PlatformHelper` variants or `useSecondaryProps` directly.
- Commands: `yarn test --forceExit` (as in CI `flash-list.yml`), `yarn type-check` (`tsc --noEmit`, which excludes tests), `yarn lint`, `yarn build` (`tsc -b`).
