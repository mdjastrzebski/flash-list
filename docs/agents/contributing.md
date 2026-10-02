<!-- index-codebase: generated from 527d7678 on 2026-10-02. Code wins when it disagrees with this doc. -->
# FlashList: Contributing

The rules for humans are in `CONTRIBUTING.md`, and agent rules are in `CLAUDE.md`. This page records what the tooling actually does, and corrects the stale commands in `CONTRIBUTING.md`.

## Setup

- **Node 22.18.0.** CI pins this in every workflow (`env.NODE_VERSION`). There is no `.nvmrc`. `dev.yml` (20.11.0) and `documentation/dev.yml` (20.4.0) are stale Shopify-internal `dev up` configs.
- **Yarn classic 1.x.** `yarn.lock` is v1 and there is no `packageManager` field. A Yarn 2+ or corepack binary fails with "package doesn't seem to be present in your lockfile".
- **Ruby 3.3.1** (`.ruby-version`), for CocoaPods and the Jekyll site.
- **Library only:** run `yarn`. Unit tests need nothing else.
- **Full setup (macOS):** `yarn up`. It installs the fixture, runs `pod install`, `brew`-installs `applesimutils` for Detox, and builds. It changes the system.
- **The fixture app consumes `dist/`.** `fixture/react-native/package.json` has `"@shopify/flash-list": "link:../../"`, and the root `main` is `dist/index.js`. Run `yarn build`, or `yarn build --watch`, after every `src/` change and after every branch switch.

## Feedback loop

| To check… | Run | Notes |
|---|---|---|
| Unit tests | `yarn test --forceExit` | Jest runs on `src/` directly, so no build is needed. 199 tests in 15 suites in `src/__tests__/`, all passing at 527d7678 |
| One file or pattern | `yarn test GridLayoutManager --forceExit` · `yarn test -t "<name>"` | Fallback: `node_modules/.bin/jest <pattern> --forceExit` |
| Lint and format | `yarn lint` / `yarn lint:fix` | ESLint 8 with `@shopify` + Prettier 2, via `.eslintrc.js`. Also lints `fixture/**` |
| Types | `yarn type-check` | `tsc --noEmit`. **Test files are excluded** (`tsconfig.json`), so they are never type-checked. CI relies on `tsc -b` in the build job |
| Build | `yarn build` | `tsc -b` → `dist/` (gitignored) |
| Fixture type-check | `cd fixture/react-native && yarn build` | Required CI check |
| Run the fixture | `yarn start` (Metro), then `yarn ri` / `yarn ra` (`fixture:rn:ios` / `fixture:rn:android`) | iOS app `FlatListPro`. Screens are registered in `fixture/react-native/src/NavigationTree.tsx` and `ExamplesScreen.tsx` |
| Web fixture | `cd fixture/web && yarn && yarn web`, plus `yarn watch-web` at the root | `watch-web` rsyncs `dist/` into `fixture/web/node_modules` and needs `fswatch`. Not in CI |
| iOS e2e (Detox screenshots) | `yarn e2e:ios` | Tests are in `fixture/react-native/e2e/tests/`. Baselines are in `e2e/artifacts/ios/<Name>/<Name>.png`; diffs go to `e2e/artifacts/ios/diff/`. To regenerate a baseline, delete its directory and run twice |
| Android e2e | `yarn e2e:android` | AVD `React-Native-Phone`. No Android baselines exist, and there is no CI job |
| Docs site | `cd documentation && yarn && yarn start` | Docusaurus, served at localhost:3000 |
| On-device checks by an agent | `.claude/skills/review-and-test`, `agent-device` | Uses simulator snapshots. Kill Metro (`lsof -ti:8081 \| xargs kill -9`) when done |

Writing tests: any test that renders a FlashList must `jest.mock("../recyclerview/utils/measureLayout", …)`. Copy the block at the top of `src/__tests__/RecyclerView.test.tsx`. Layout managers can be unit-tested directly with `src/__tests__/helpers/createLayoutManager.ts`. Repo Jest resolves the generic `PlatformHelper.ts`, not the `.ios`/`.android` files.

### Corrections to `CONTRIBUTING.md`

| `CONTRIBUTING.md` says | Actually |
|---|---|
| `yarn run-android` / `yarn run-ios` / `yarn run-web` | `yarn fixture:rn:android` (`ra`) / `yarn fixture:rn:ios` (`ri`) / the web flow above. `rw` and `fixture:expo:*` point at the removed `fixture/expo` |
| `run-e2e-ios` / `run-e2e-android` | `yarn e2e:ios` (`rei`) / `yarn e2e:android`. The `rea` alias is broken |
| e2e tests in `fixture/e2e`, screenshots in `fixture/e2e/artifacts/ios` | `fixture/react-native/e2e/tests/`, `fixture/react-native/e2e/artifacts/ios/` |
| PR template `.github/PULL_REQUEST_TEMPLATE.md` | `.github/pull_request_template.md` (the casing breaks the link) |

## CI (`.github/workflows/flash-list.yml`, `website.yml`)

These are **required on `main`**:
- Build Typescript
- Build fixture Typescript
- Lint Typescript
- Test Typescript (`yarn test --forceExit`)
- Metro bundling for iOS and for Android
- Build iOS (`xcodebuild`)
- Build Android (`./gradlew assembleDebug`)
- Build documentation
- `cla`

`Run iOS e2e tests` runs on every PR but is **not** required. Its diff PNGs are uploaded as artifacts when it fails. Every action is pinned by SHA.

## Rules

- **Conventional commits:** `fix(scope): …`, `feat(scope): …`. Scopes include `layout`, `hooks`, `scroll`, `sticky-headers`, `recycling`, `viewability`. Link issues with `Fixes #N`. This is written policy, not enforced by tooling.
- **CLA:** contributors must sign Shopify's CLA. This is enforced (`cla.yml`, required check).
- **PR template:** `.github/pull_request_template.md`.
- **Documentation:** update `documentation/docs/` for API changes (`CONTRIBUTING.md`).
- **CHANGELOG.md:** written by hand under `## [Unreleased]`. It has no v2 version sections; v2 release notes appear to live in GitHub Releases. **Inferred.**
- **Agent-raised PRs** (`CLAUDE.md`, `.claude/skills/raise-pr`):
  - Create the PR with `GH_TOKEN="$AGENT_PR_TOKEN" gh pr create`.
  - No AI attribution.
  - Kill Metro before finishing on CI.
- **Self-learning:** `CLAUDE.md` asks agents to record new pitfalls in the matching skill file, in the same PR as the fix.

## Release process

1. A maintainer on an up-to-date `main` runs `yarn version`. Yarn 1 commits the version and tags it (e.g. 527d7678 "v2.3.3").
2. `git push origin main --follow-tags`. This is a direct push to `main`, so it needs admin rights.
3. Publish a GitHub Release for the tag. That triggers `.github/workflows/release.yml`:
   - Checks out the tag, `yarn install --frozen-lockfile`.
   - `npm publish --provenance` via OIDC trusted publishing. `prepublishOnly` builds `dist/`.
   - The dist-tag is the prerelease identifier (e.g. `alpha`), or `latest`.
   - It can also be started by hand with `workflow_dispatch` and a `tag` input.

Docs deploy separately. Every push to `main` runs `deploy-website-on-main.yml` → `build_website.sh`, which builds Docusaurus (`documentation/`) and the Jekyll landing page (`website/`), merges them, and pushes to `gh-pages`. `shipit.production.yml` is a Shopify-internal Shipit config; GitHub Release + OIDC is the path that publishes.

## Agent automation

`.github/workflows/agent-{bot,android-bot,fix,triage}.yml` run `anthropics/claude-code-action` through Shopify's AI proxy, following the `.claude/skills/` workflows.

**All four are disabled.** Each job's `if:` starts with `false &&` (435b5141, #2454, "pending security review").

| Workflow | Triggered by | Runner and model | What it can do |
|---|---|---|---|
| `agent-bot.yml` | `@agent` comment from an owner, member or collaborator | macOS, Opus | Contents, PR and issue write. `AGENT_PR_TOKEN` (= `secrets.SHOPIFY_GH_ACCESS_TOKEN`). Allowed tools include `rm`, `kill`, `git push -u`, `gh pr *`, WebFetch |
| `agent-android-bot.yml` | `@android-agent` comment, same gate | Ubuntu with a KVM emulator | Same as `agent-bot.yml` |
| `agent-fix.yml` | `agent-fix` label or `/fix` comment, plus a write-permission check | macOS | Same as `agent-bot.yml`. Follows `fix-github-issue` + `raise-pr` |
| `agent-triage.yml` | `agent-triage` label, plus a write-permission check | Ubuntu, Sonnet, 10 turns | Issue write only (labels). No PR token |

Hardening history:
- Proxy prompt-scan headers in block mode (20e29c20, 4c6abc3b).
- Issue text is no longer interpolated into prompts (fb051a78).
- Label triggers are authorized by the actor's repo permission (ef42058f).

Known loose ends:
- The workflow prompts say to use `gh api`, but `gh api` is not in the allowlists, and `triage-issue` forbids it.
- `raise-pr`'s `SHOPIFY_GH_ACCESS_TOKEN` env fallback is never exported by any workflow.
- `npm install -g agent-device` is unpinned.

Feedback artifacts (`/tmp/agent-feedback.md`) are mined by the `analyze-feedback` skill. Its cursor is `.claude/feedback-scan-cursor.json`.

## Generated and do-not-edit files

| Path | Origin |
|---|---|
| `dist/` | `yarn build` (gitignored) |
| `website/_site/`, `documentation/build/` | `build_website.sh` |
| `fixture/react-native/e2e/artifacts/ios/**` | Detox baselines. Regenerate them; don't hand-edit |
| `documentation/versioned_docs/version-1.x/` | Frozen v1 docs |
| `yarn.lock` files | Yarn 1 |
