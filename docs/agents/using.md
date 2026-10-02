<!-- index-codebase: generated from 527d7678 on 2026-10-02. Code wins when it disagrees with this doc. -->
# FlashList: Using and Extending

The user-facing prop and ref reference is `documentation/docs/fundamentals/usage.md` (live at https://shopify.github.io/flash-list/docs/usage). This page covers what that reference doesn't say, or gets wrong.

## Platforms and integrations

| Platform / integration | Supported versions | Where it lives |
|---|---|---|
| React Native, new architecture (Fabric) | Required. No declared range (peer deps `react-native: *`, `react: *`). Exercised with RN 0.84.1 / React 19.2.3 (`fixture/react-native`) and RN 0.76.5 / React 18.3.1 (root dev deps, used for tests) | `src/index.ts` guard, `src/isNewArch.ts` |
| Old architecture | Not supported. Import throws `flashListV2OnlySupportsNewArchitecture`. Use FlashList 1.x | `src/errors/ErrorMessages.ts` |
| iOS / Android | Supported. Per-platform constants differ (see below) | `src/native/config/PlatformHelper.{ios,android}.ts` |
| Web (react-native-web) | Supported, allowed through `isNewArch`. DOM-based measuring, draw distance 500, MVCP via `scrollTo` | `PlatformHelper.web.ts`, `src/recyclerview/utils/measureLayout.web.ts`, `fixture/web` (not in CI) |
| RN ≥ 0.83 | Only version-gated behavior: sticky headers unmount when no header is active | `src/native/config/versionCheck.ts` (#2076) |
| Expo | Works as a plain JS dependency, no config plugin. The docs claim "Expo Go SDK 46+" predates v2. **Inferred** stale | `documentation/docs/index.mdx` |
| Reanimated / Animated | `AnimatedFlashList` = `Animated.createAnimatedComponent(FlashList)`. Reanimated guide in docs | `src/AnimatedFlashList.ts`, `documentation/docs/guides/reanimated.md` |
| Jest (users' tests) | Ship `jestSetup.js`, which mocks `dist/recyclerview/utils/measureLayout` (400×900 window, 100×100 items) | `jestSetup.js`, `documentation/docs/guides/testing.md` |

## Configuration

Everything is a prop. There are no env vars, globals or config files. These are the props whose behavior isn't obvious from the reference:

| Setting | Where it's read | Default | Notes |
|---|---|---|---|
| `drawDistance` | `RecyclerViewManager.updateProps` → `EngagedIndicesTracker` | 250 native, 500 web (`PlatformConfig.defaultDrawDistance`) | Sticky: unsetting it keeps the last value |
| `getItemType` | `RecyclerViewManager.getItemType` | `"default"` | Controls both recycle pools and size-estimate buckets |
| `keyExtractor` | `RecyclerViewManager.getDataKey` | `String(index)` | Required for MVCP anchoring and layout animations (warnings otherwise) |
| `maintainVisibleContentPosition` | `RecyclerViewManager.shouldMaintainVisibleContentPosition`, `useRecyclerViewController`, `useBoundDetection` | Enabled. `animateAutoScrollToBottom` true | `{disabled: true}` turns it off. Sub-options: `autoscrollToTopThreshold`, `autoscrollToBottomThreshold`, `startRenderingFromBottom` |
| `maxItemsInRecyclePool` | `RenderStackManager` constructor (read once) | Unlimited | `0` unmounts cells when they leave the window |
| `numColumns`, `masonry`, `optimizeItemArrangement`, `overrideItemLayout` | `RecyclerViewManager.getLayoutManagerClass` / `updateLayoutParams` | 1, false, true, none | Only `span` is read from `overrideItemLayout` |
| `stickyHeaderConfig` (`offset`, `zIndex`, `useNativeDriver`, `hideRelatedCell`, `backdropComponent`) | `RecyclerView.tsx`, `useSecondaryProps` | 0, 2, true, false, none | See [sticky-headers](areas/sticky-headers.md) |
| `viewabilityConfig` / `viewabilityConfigCallbackPairs` | `ViewabilityManager` (constructed once) | `minimumViewTime` 250 ms (FlatList uses 0) | Frozen at mount |
| `onEndReachedThreshold` / `onStartReachedThreshold` | `useBoundDetection` | 0.5 / 0.2 | In viewport lengths |
| `initialScrollIndexParams.viewOffset` | `useRecyclerViewController` | 0 | |
| `overrideProps` | Spread last onto the ScrollView; `initialDrawBatchSize` read by `RecyclerViewManager` | `initialDrawBatchSize` 2 | Escape hatch. Can break the list |

Not user-settable: `PlatformConfig.supportsOffsetCorrection` (iOS/Android), `trackAverageRenderTimeForOffsetProjection` (Android), `RenderTimeTracker` limits (40 renders, 16–32 ms).

## Extension points

- **Public hooks** (`src/index.ts`):
  - `useRecyclingState`: per-cell state that resets when the cell is recycled.
  - `useLayoutState`: state whose changes trigger a synchronous relayout.
  - `useMappingHelper`: stable keys for `.map()` inside recycled cells.
  - `useFlashListContext`: lets nested children reach the list's ref and scroll view.
  - `LayoutCommitObserver`: callback once every FlashList in its subtree has committed.
  - Semantics are in [recycler-core](areas/recycler-core.md).
- **Render overrides:** `CellRendererComponent` must forward `ref` and `onLayout` to a host view. Also available: `renderScrollComponent`, `ListHeaderComponent`, `ListFooterComponent`, `ListEmptyComponent`, `ItemSeparatorComponent`, `stickyHeaderConfig.backdropComponent`.
- **Custom layouts:** not pluggable. No layout manager is exported, so a new layout type needs a fork or an upstream change ([layout-managers](areas/layout-managers.md#where-to-change-things)).
- **Benchmarking:** `useBenchmark`, `useFlatListBenchmark`, `useDataMultiplier`, `JSFPSMonitor`, `autoScroll` ([benchmark](areas/benchmark.md)).

## Limitations and gotchas

- `horizontal` cannot be toggled after mount; remount with a new `key`. `horizontal` throws when combined with `masonry`, `numColumns > 1` or `stickyHeaderIndices`.
- Mutating `data` in place does nothing. Data changes are detected by identity.
- Without `keyExtractor`, prepends can't be anchored and recycled state can leak. Duplicate keys can produce the "exceeded max renders without commit" warning.
- Items that change size without `useLayoutState` / `useRecyclingState` are only corrected through native `onLayout`, a frame late (possible flicker).
- Unknown and removed v1 props (`estimatedItemSize`, `onBlankArea`, `disableAutoLayout`, …) are silently forwarded to the ScrollView.
- `viewabilityConfig` and the presence of viewability callbacks are frozen at mount.
- Doc/code mismatches to keep in mind:
  - Docs say `getVisibleIndices()`; the code has `computeVisibleIndices()`, which returns `{startIndex, endIndex}`.
  - `usage.md` still lists `onBlankArea`.
  - `scrollToEnd` is typed `void` but is async.
- Known issues from users' point of view: `documentation/docs/known-issues.md` (RTL horizontal padding, horizontal lists with headers need a fixed height, MVCP-caused movement on reorder).

## Security and trust

- **Runtime:** the library makes no network calls and sends no telemetry. `src/` has no `fetch`, `XMLHttpRequest` or `WebSocket` and no `process.env` reads. It writes no files, and its only side effects are `console.warn`/`console.error`.
- **Install:** no `postinstall`, `install` or `prepare` scripts run on users' machines. `prepublishOnly` (`yarn run build`) runs only for maintainers. The package ships `dist`, `src` and `jestSetup.js`.
- **Supply chain:** published from GitHub Actions with npm trusted publishing (OIDC) and `--provenance` (`.github/workflows/release.yml`).
- **Reporting vulnerabilities:** the repo has no `SECURITY.md`. Shopify's org-level policy applies: HackerOne, https://hackerone.com/shopify.
- **For contributors and agents working in this repo:**
  - `.claude/settings.json` pre-allows several Bash prefixes for anyone opening the repo in Claude Code, including `rm`, `kill`, `nohup`, `git push -u` and `gh pr`. Most entries use a `Bash(cmd:sub:*)` colon form that may not match as intended. **Unverified.**
  - The skills that act outward or destructively:
    - `raise-pr` pushes branches and opens PRs.
    - `triage-issue` labels and comments on issues.
    - `upgrade-react-native` runs `rm -rf node_modules yarn.lock` and `rm -rf Pods Podfile.lock`.
    - `fix-github-issue` kills port 8081 and opens PRs.
    - `review-and-test` temporarily edits `fixture/react-native/index.js`, and may inject a debug `fetch("http://localhost:9876")` into `src` that has to be removed.
  - The four `agent-*.yml` GitHub workflows are disabled (`false &&` in the job `if:`, 435b5141) pending a security review. Details are in [contributing.md](contributing.md#agent-automation).
