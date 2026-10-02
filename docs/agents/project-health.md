<!-- index-github: snapshot of Shopify/flash-list on 2026-10-02, last 365 days. Numbers go stale; rerun index-github to refresh. -->
# FlashList: Project Health

**Verdict:** Maintained, slow, and trending toward minimal maintenance. Releases still ship (v2.3.3 came out 2026-10-01) and the lead maintainer's own fixes land within hours. But since spring 2026 almost no community PR has been reviewed: 65 open external PRs have had no maintainer response for more than 14 days, and commits fell from 38 in March 2026 to 1–3 a month.
**Risks:** *Concentration*: one person (naqvitalha) did 92% of merges and about 79% of human commits in the last year. *License*: none. MIT, with a Shopify CLA required for contributions. *Stability*: low. Post-1.0, one major release in three years, but v2 was a full rewrite that supports only the New Architecture.
**Snapshot:** 2026-10-02, covering 2025-10-02 to 2026-10-02. Not truncated: 481 PRs, 90 issues, and all 73 open issues and 131 open PRs were fetched.

## What it is

A drop-in replacement for React Native's `FlatList` for long or complex lists. v1 (2022) recycled native views through `recyclerlistview` and needed size estimates. v2 (July 2025) is a JS-only rewrite for the New Architecture that measures items itself, so it needs no estimates. It also adds masonry, sticky-header config, and `maintainVisibleContentPosition` for chat UIs (per its README). It's for app teams that hit FlatList's blank-cell and memory problems on large datasets. See `docs/agents/areas/` for how the code works.

## Alternatives

| Project | Approach | Differences | Activity |
| --- | --- | --- | --- |
| React Native `FlatList` | Built-in virtualization that mounts and unmounts cells | No recycling, so more blank cells and GC churn on fast scroll. No dependency. Works on both architectures. | Ships with RN (Meta) |
| [@legendapp/list](https://github.com/LegendApp/list) | Pure-JS virtualized list with optional recycling | Same "no estimates, JS-only" pitch as FlashList v2, with a strong focus on chat and bidirectional lists. Its license wasn't detected by the GitHub API; check its package. | v3.6.0 on 2026-09-29; 3.4k stars; ~718k npm downloads/week |
| [recyclerlistview](https://github.com/Flipkart/recyclerlistview) | Native-view recycling with mandatory size estimates (the engine under FlashList v1) | Lower level and needs explicit layout providers. | Last release 4.2.3 in Mar 2025; ~786k downloads/week, probably mostly pulled in by FlashList v1 |

FlashList gets ~2.83M npm downloads a week (2026-09-24 to 09-30), about 4× Legend List. This comparison rests on GitHub and npm data plus general knowledge. No web search was done.

## Maturity

- **Version:** v2.3.3. The project reached 1.0 in June 2022 and 2.0 on 2025-07-31. The CHANGELOG says it follows SemVer.
- **Breaking releases:** four 0.x minors in 2022, then 1.0 (2022-06) and 2.0 (2025-07). Breaking changes are rare, but 2.0 dropped the old architecture and changed the API (no `estimatedItemSize`). The project has v1 docs and a migration guide (`documentation/docs/v2-migration.md`). v1 last shipped as v1.8.3 in June 2025, and nothing indicates it's still maintained ([#2256](https://github.com/Shopify/flash-list/issues/2256) asks about v1 on RN 0.84 and has no answer).
- **Production use:** the repo has no adopters list. Usage shows in 7.2k stars and 2.8M weekly npm downloads.
- `CHANGELOG.md` is stale: it has no 2.x sections. The real release notes are the auto-generated GitHub Releases.

## Activity

### Maintainers

- **Merges:** naqvitalha 47, devi-labs 3 (their own CI-hardening PRs), alexrass 1. Top merger share is 0.92.
- **Commits, last year:** 65 in total, 58 by humans. naqvitalha made 46, and no one else made more than 3. Bus factor: 1 (2 people are needed to reach 80%). All-time, earlier maintainers (Marek Fořt 118, David Cortés 83, Elvira Burchik 27) have stopped committing.
- **Reviews:** tmgsca and gantonious review naqvitalha's own PRs (CI, scroll fixes, RN upgrades), but they don't review community PRs. In the window, only naqvitalha answered issues as a maintainer.
- **No governance files:** there's no `CODEOWNERS`, `MAINTAINERS`, or `GOVERNANCE.md`. GitHub shows naqvitalha as COLLABORATOR, not MEMBER, because org membership is private.

### Responsiveness

| PRs (last 365 days) | Community* | Maintainer | Dependabot |
| --- | --- | --- | --- |
| Opened | ~82 | 40 | 347 |
| Landed | 6 | 32 (all self-merged) | 4 |
| Closed unmerged | 8 | 6 | 272 |
| Still open | 67 | 2 | 57 |
| Median first maintainer response | ~8 days, but only for the ~20 that got one | 0.7 days | n/a |
| Median days to land | 3.5 (n=12, incl. Shopify staff) | 0.1 | 0.6 |

\*External PRs minus Shopify-run accounts (`shopify-github-actions-access` agent bot, devi-labs, and three authors with Shopify email domains). The six community PRs that landed: [#1995](https://github.com/Shopify/flash-list/pull/1995) (131 days), [#2053](https://github.com/Shopify/flash-list/pull/2053), [#2095](https://github.com/Shopify/flash-list/pull/2095), [#2105](https://github.com/Shopify/flash-list/pull/2105), [#2290](https://github.com/Shopify/flash-list/pull/2290), and [#2311](https://github.com/Shopify/flash-list/pull/2311). No community PR has landed since 2026-06-09.

**Issues:** 71 were filed by users. The other 19 were maintainer or security test issues for the agent workflows, which close within minutes and pull the script's median close time down to 0.08 days. Of the 71, 21 are closed, with a median of **70 days to close**, and a maintainer commented on only 11. From March to June 2026, a Claude-based `agent-triage` workflow labeled issues P0/P1/P2. Since it was disabled ([#2454](https://github.com/Shopify/flash-list/pull/2454), 2026-08-19), new issues ([#2427](https://github.com/Shopify/flash-list/issues/2427), [#2509](https://github.com/Shopify/flash-list/issues/2509), [#2517](https://github.com/Shopify/flash-list/issues/2517)) haven't been labeled at all. Nothing was closed as not planned, and there's no stale bot.

### Backlog

- **73 open issues:** 63 older than 90 days, 22 older than a year, and 27 with no comments. 60 are labeled `bug`: 1 P0 ([#2086](https://github.com/Shopify/flash-list/issues/2086), diagnosed as an upstream RN bug), 31 P1, and 19 P2.
- **131 open PRs:** 59 are Dependabot. Of the rest, 67 have no review. 22 came from a single account on 2026-08-28 and look like a bulk automated sweep ([#2470](https://github.com/Shopify/flash-list/pull/2470)–[#2487](https://github.com/Shopify/flash-list/pull/2487)). Many others are small, focused fixes for open P1 bugs that are still waiting, such as [#2458](https://github.com/Shopify/flash-list/pull/2458), [#2459](https://github.com/Shopify/flash-list/pull/2459), [#2383](https://github.com/Shopify/flash-list/pull/2383), and [#2400](https://github.com/Shopify/flash-list/pull/2400).
- [#2072](https://github.com/Shopify/flash-list/pull/2072) (a docs fix) was approved twice in January 2026 but was never merged.

### Releases

There were 8 stable releases in the window: v2.2.0 (2025-10-27), v2.2.1–2.2.3 (Jan–Mar), v2.3.0–2.3.1 (Mar), v2.3.2 (2026-06-10), and v2.3.3 (2026-10-01). The median gap between stable releases is ~23 days over the long run, but the most recent gap was 113 days. Prereleases (`-alpha`/`-rc`) were used heavily before 2.0 and hardly at all since. Releases are clustered around periods of maintainer focus: commits peaked in Mar 2026 (38) and May–Jul 2025 (24–37 a month).

## Backing and funding

The project is owned by the Shopify organization and is part of Shopify's React Native open-source group (it has a Discord channel). It requires the Shopify CLA, and its security reports go through Shopify's program (a `Hackerone` label shows up on PRs). In the last year, three committers used Shopify email domains. The lead maintainer commits from a personal address. There are no funding links, sponsors, paid tier, or open-core split. In practice it's a single-company project that depends on how much time Shopify gives one engineer.

## License

- MIT (`LICENSE.md`, the only license file in the tree). Commercial use is fine.
- **CLA:** contributors must sign the Shopify CLA through `Shopify/shopify-cla-action` (17 PRs carried `cla-needed`). Read its terms if copyright assignment matters to you. No DCO. The only `Signed-off-by` lines come from Dependabot.
- **Runtime dependencies:** none. The peer dependencies are `react`, `react-native`, and `@babel/runtime`.

## Security

- The org-level security policy is enabled ([policy](https://github.com/Shopify/flash-list/security/policy)). No security advisories have been published.
- Dependabot is configured, but it mostly churns: 272 of 347 PRs were closed unmerged and 57 are open.
- No install-time lifecycle scripts.
- **Agent workflows:** March–August 2026 saw Claude-based triage and fix workflows. In July a security retest series (SI-2719, test issues [#2360](https://github.com/Shopify/flash-list/issues/2360)–[#2371](https://github.com/Shopify/flash-list/issues/2371)) probed them for prompt injection. They were hardened ([#2359](https://github.com/Shopify/flash-list/pull/2359), [#2368](https://github.com/Shopify/flash-list/pull/2368), [#2372](https://github.com/Shopify/flash-list/pull/2372)) and then disabled ([#2454](https://github.com/Shopify/flash-list/pull/2454)).

## Community

The issue templates send questions to the [docs](https://shopify.github.io/flash-list/docs/), GitHub Discussions (168 threads), and the Shopify React Native OSS Discord. There's a Contributor Covenant code of conduct. Decisions are made informally by the lead maintainer. There's no public roadmap, no milestones, and no pinned issues.

## Contributing in practice

- **What's welcome:** there are no `good first issue` or `help wanted` labels. The best bets are focused fixes for P1 bugs that include a repro (an Expo snack, or a fixture sample) and a unit test. That's the pattern behind the community PRs that landed.
- **What gets rejected:** heavy algorithmic changes on hot paths ([#2104](https://github.com/Shopify/flash-list/pull/2104), where an O(N log N) sort per layout was turned down), fragile docs tooling ([#2054](https://github.com/Shopify/flash-list/pull/2054)), problems that can be solved in userland ([#2221](https://github.com/Shopify/flash-list/pull/2221)/[#2218](https://github.com/Shopify/flash-list/issues/2218)), PRs superseded by a root-cause fix ([#1952](https://github.com/Shopify/flash-list/pull/1952)), and drive-by AI bot PRs ([#1987](https://github.com/Shopify/flash-list/pull/1987)–[#1990](https://github.com/Shopify/flash-list/pull/1990)).
- **How review goes:** when naqvitalha reviews, the feedback is technical and specific: they explain the layout model and often suggest a simpler path ([#2108](https://github.com/Shopify/flash-list/pull/2108)). The usual wait is the problem. Expect weeks to months, or no reply.
- **Credit:** GitHub auto-generated release notes name PR authors and first-time contributors. 21 commits in the last year kept `Co-authored-by` trailers.
- For the written rules, see `CONTRIBUTING.md` and `CLAUDE.md` (conventional commits, scopes).

## Problem areas

| Area | Theme | Evidence |
| --- | --- | --- |
| `maintainVisibleContentPosition` / chat ([scrolling](areas/scrolling.md)) | Jitter, flicker, or jumps when prepending, using `startRenderingFromBottom`, or with the keyboard | [#2052](https://github.com/Shopify/flash-list/issues/2052) (most-reacted), [#2018](https://github.com/Shopify/flash-list/issues/2018), [#2050](https://github.com/Shopify/flash-list/issues/2050), [#2070](https://github.com/Shopify/flash-list/issues/2070), [#1998](https://github.com/Shopify/flash-list/issues/1998), [#2026](https://github.com/Shopify/flash-list/issues/2026), [#2066](https://github.com/Shopify/flash-list/issues/2066) |
| Initial scroll / `scrollToIndex` (`useRecyclerViewController`, `RecyclerViewManager`) | Wrong items or offset with `initialScrollIndex`, double render | [#2307](https://github.com/Shopify/flash-list/issues/2307), [#1938](https://github.com/Shopify/flash-list/issues/1938), [#1996](https://github.com/Shopify/flash-list/issues/1996) |
| Render stack and layout table ([recycler core](areas/recycler-core.md)) | "index out of bounds" / "Exceeded max renders" crashes when data shrinks or layouts lag behind data | [#2440](https://github.com/Shopify/flash-list/issues/2440), [#2509](https://github.com/Shopify/flash-list/issues/2509), [#2291](https://github.com/Shopify/flash-list/issues/2291), [#2319](https://github.com/Shopify/flash-list/issues/2319), [#2013](https://github.com/Shopify/flash-list/issues/2013), [#2517](https://github.com/Shopify/flash-list/issues/2517) |
| Sticky headers ([sticky headers](areas/sticky-headers.md)) | Offset shifts content, the header blocks touches, duplicate or early headers | [#2419](https://github.com/Shopify/flash-list/issues/2419), [#2249](https://github.com/Shopify/flash-list/issues/2249), [#1959](https://github.com/Shopify/flash-list/issues/1959), [#1942](https://github.com/Shopify/flash-list/issues/1942) |
| Masonry ([layout managers](areas/layout-managers.md)) | Misalignment after pagination, items jumping or disappearing | [#2270](https://github.com/Shopify/flash-list/issues/2270), [#2107](https://github.com/Shopify/flash-list/issues/2107), [#2103](https://github.com/Shopify/flash-list/issues/2103) |
| Inverted / web / a11y | TalkBack order, web virtualization freezes, scrollbar layout loop, tab order | [#2381](https://github.com/Shopify/flash-list/issues/2381), [#2380](https://github.com/Shopify/flash-list/issues/2380), [#2334](https://github.com/Shopify/flash-list/issues/2334), [#1955](https://github.com/Shopify/flash-list/issues/1955) |
| Recycled cell state | A `TextInput` keeps focus or text in the wrong row; video plays in the wrong cell | [#2299](https://github.com/Shopify/flash-list/issues/2299), [#1980](https://github.com/Shopify/flash-list/issues/1980), [#1965](https://github.com/Shopify/flash-list/issues/1965) |
| Viewability ([viewability](areas/viewability.md)) | Wrong viewable items with headers or initial scroll; `onEndReached` firing twice | [#2238](https://github.com/Shopify/flash-list/issues/2238), [#1984](https://github.com/Shopify/flash-list/issues/1984) |

By git history, fix commits in the last year cluster in `src/recyclerview/hooks` (7), `RecyclerViewManager.ts` (5), `layout-managers` (4), and `RecyclerView.tsx` (4). The most-wanted feature, `contentContainerStyle` `gap` ([#2097](https://github.com/Shopify/flash-list/issues/2097)), was labeled P2 by the agent but has no maintainer reply.

## Caveats

- The script counts `shopify-github-actions-access` (Shopify's agent bot) and Shopify staff as "external". The community numbers above were recomputed by hand from `prs.json`.
- The script's issue medians are skewed by about 19 workflow and security test issues. Use the user-only figures above.
- The `P0`/`P1`/`P2` labels were mostly applied by an AI triage workflow, not by a maintainer, so they aren't proof that a maintainer looked at the issue.
- The open-PR count is inflated by one bulk sweep of 22 PRs. Even without it, 40+ focused community fixes are waiting.
- naqvitalha shows as COLLABORATOR because org membership is private, so maintainer-only response metrics depend on that one account.
- The alternatives comparison didn't use a web search. The clone was current as of 2026-10-01.
