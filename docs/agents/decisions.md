<!-- index-codebase: generated from 527d7678 on 2026-10-02. Code wins when it disagrees with this doc. -->
# FlashList: Design Decisions

These are cross-cutting decisions only. Area-specific ones are in each area doc's "Design decisions" section. There are no ADRs or RFCs in the repo. The evidence below is commits, PRs, code comments and `documentation/docs/v2-changes.md`.

## v2 is a JS-only rewrite for the new architecture

| Decision | Reason | Evidence |
|---|---|---|
| Drop the v1 native cell container and `recyclerlistview`, and rewrite the engine in TypeScript (`src/recyclerview/`) | Easier to maintain. One code path for iOS, Android and web | 5e9a6e01 (#1617), f9c635a9, d0aea594 "Remove native iOS code", README "JS-only solution" |
| New architecture only: `src/index.ts` throws on import otherwise | Fabric's `measureLayout` returns synchronously inside `useLayoutEffect`. That is what lets v2 measure and reposition before paint without estimates. **Inferred** from `measureLayoutRelative` in `src/recyclerview/utils/measureLayout.ts` reading values right after the call | README, `src/isNewArch.ts` |
| No size estimates from users (`estimatedItemSize` and friends removed, with no warning) | Measurement before paint makes them unnecessary. Unmeasured items use a per-`getItemType` running average (`MultiTypeAverageWindow(5, 200)`) | `documentation/docs/v2-changes.md`, `src/utils/AverageWindow.ts`, `CLAUDE.md` |
| `FlashList` is just an alias of `RecyclerView`. Masonry is a prop, not a separate component | One pipeline for every layout type | `src/FlashList.ts`, [public-api](areas/public-api.md) |
| v1 maintained separately (docs `versioned_docs/version-1.x`) | Users on the old architecture stay on 1.x | `documentation/versions.json`, README |

## Engine shape

| Decision | Reason | Evidence |
|---|---|---|
| A mutable `RecyclerViewManager` holds all state. React state (`setRenderId`, `setLayoutTreeId`) is only a re-render trigger | Layout and recycling math has to run synchronously inside layout effects and scroll callbacks. **Inferred** from structure | `useRecyclerViewManager.ts`, `RecyclerView.tsx` |
| Render → measure → re-render loop in a dependency-less `useLayoutEffect`, capped at 40 renders without commit (warning, not a crash) | Settles layout before paint. The cap turned a React "maximum update depth" crash into a recoverable warning | a102767a (#1826), `RenderTimeTracker` |
| Layouts keyed by **index**. Anchoring for prepends uses `keyExtractor` | Simple array math. MVCP and recycling identity depend on `keyExtractor` instead | [layout-managers](areas/layout-managers.md), [scrolling](areas/scrolling.md) |
| Recycle only within an item type. Off-screen recycled cells stay mounted | Reuse is cheaper than mounting. `maxItemsInRecyclePool` caps memory | `RenderStackManager.ts` |
| Progressive first render in geometric batches, with the buffer rendered only after the first paint | Fast first paint, and fewer setState cascades when lists are nested | edfa37d3, 40fb21db (#1899) |
| Velocity-projected render window, clamped to always cover the real viewport | Projection reduces blanks during flings. The clamp fixes blank bands when `drawDistance` is small | 9b5e7f8b (#2510) |
| MVCP on by default, implemented with an invisible `ScrollAnchor` and native `maintainVisibleContentPosition` on iOS/Android | The shift lands in the same frame as the layout change, so there is no flash | `ScrollAnchor.tsx`, 422dcda9 (#2145), 43b0f317 |
| `removeClippedSubviews` forced off (still overridable via `overrideProps`) | Native clipping detached buffered cells and crashed Fabric on Android | 4fa1ccdc (#2202, fixes #1914) |
| Per-platform constants via bundler file extensions (`PlatformHelper.{ios,android,web}.ts`), not runtime `Platform.OS` branches | No dead code per platform. Web gets a larger draw distance and no anchor-based correction | `src/native/config/` |
| Android inverts with `rotate(180deg)` instead of `scaleY:-1` | Performance (#751). The cost is the scrollbar on the wrong side (#2131) | `PlatformHelper.android.ts` |

## Process decisions

| Decision | Reason | Evidence |
|---|---|---|
| Agent-driven triage/fix/review via GitHub Actions + `.claude/skills/`, now disabled | Workflows were added, hardened against prompt injection, then switched off pending a security review | 0b207595 (#2157) … 435b5141 (#2454) |
| Agent PRs must be created with `SHOPIFY_GH_ACCESS_TOKEN` and carry no AI attribution | PRs created with `GITHUB_TOKEN` can't be merged (`CLAUDE.md`). Attribution policy is the maintainers' rule | ce349cfb (#2192), `CLAUDE.md`, `.claude/skills/raise-pr` |
| npm publish via GitHub Release → OIDC trusted publishing with provenance | No long-lived npm token | `.github/workflows/release.yml` |
