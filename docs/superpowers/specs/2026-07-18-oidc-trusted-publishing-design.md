# Design: OIDC trusted publishing for `@pkg-nec/exceljs`

**Status:** approved
**Date:** 2026-07-18
**Author:** Hardy Nguyen (maw.signup@gmail.com)

## Context

`4.4.0` was published by hand from a laptop logged into an npm account with rights on the `@pkg-nec` org. This works but carries the same costs it did for http-server: the publish credential lives on developer machines (rotation, offboarding, laptop loss all become supply-chain events), and every future release requires a human at that specific machine.

This spec wires OIDC-based trusted publishing between `pkg-nec/exceljs` (GitHub) and `@pkg-nec/exceljs` (npm). Once it lands, every future release is published from GitHub Actions using short-lived OIDC tokens — no long-lived npm credentials anywhere — with signed provenance attestations on every published tarball.

Unlike the http-server handshake, this spec includes the first live publish (`4.4.1`) as its end-to-end verification step. `4.4.1` is functionally identical to `4.4.0`; its sole purpose is to prove the trust chain end-to-end before it is depended on for a real feature release.

## Goal

Establish OIDC-based trusted publishing between `pkg-nec/exceljs` (GitHub) and `@pkg-nec/exceljs` (npm), gated on GitHub Release publication and a manual environment approval, and publish `4.4.1` as the first OIDC-sourced release. After this spec lands:

1. Push a `v*` tag from `main`.
2. Create a GitHub Release from that tag.
3. Approve the paused `npm-publish` job in the Actions UI.
4. Workflow runs `npm publish --provenance` using an OIDC token; tarball ships with a signed provenance attestation viewable on the package's npm page.

No `NPM_TOKEN` secret exists anywhere in the pipeline.

## Definition of done

1. On `npmjs.com`, `@pkg-nec/exceljs` has a Trusted Publisher configured with:
   - Organization: `pkg-nec`, Repository: `exceljs`, Workflow: `publish.yml`, Environment: `npm-publish`.
2. On GitHub, `pkg-nec/exceljs` has:
   - An Environment `npm-publish` with required reviewer `maw629` and deployment rules `Branch: main` + `Tag: v*.*.*`.
   - Branch protection on `main` requiring PR + all five CI status checks to pass.
3. `.github/workflows/publish.yml` exists on `main`, matches the shape in Section 4 of this spec, passes YAML parse, and its `id-token: write` permission and `environment: npm-publish` binding are in place.
4. `package.json` `publishConfig` on `main` contains `"access": "public"` and `"provenance": true`.
5. `@pkg-nec/exceljs@4.4.1` is published on npm with a provenance attestation visible on the package page.

## Non-goals

- Any change to `.github/workflows/tests.yml` (test CI). That workflow keeps its own trigger and matrix.
- Any change to `lib/`, `dist/`, `dependencies`, `devDependencies`, `package-lock.json`, or `engines.node`.
- `files` in `package.json` — stays as-is.
- Any `NPM_TOKEN` secret configuration. Trusted publishing replaces it; the goal is that no such secret exists.
- Alternative trigger designs (tag-push, workflow-dispatch). Trigger is GitHub Release publication.

## Work items

### 1. Register Trusted Publisher on npm (out-of-tree, one-time)

On `npmjs.com`, logged in as an account with admin rights on the `@pkg-nec` org:

1. Navigate to `@pkg-nec/exceljs` → **Settings → Trusted Publisher**.
2. **Add publisher → GitHub Actions.**
3. Fill in exactly:
   - Organization or user: `pkg-nec`
   - Repository: `exceljs`
   - Workflow filename: `publish.yml`
   - Environment name: `npm-publish`
4. Save.

All four fields must match the workflow's context at publish time or the OIDC handshake fails with `npm ERR! code E401`. Every character is a case-sensitive match against what the workflow's OIDC JWT presents.

### 2. Create GitHub Environment (out-of-tree, `gh api`)

Create the `npm-publish` environment on `pkg-nec/exceljs` with:

1. **Name:** `npm-publish`.
2. **Required reviewers:** `maw629`.
3. **Deployment branches and tags:** two rules:
   - Ref type `Branch`, pattern `main` — guards future `workflow_dispatch`-style paths.
   - Ref type `Tag`, pattern `v*.*.*` — **required** because release-triggered workflow runs execute against the tag ref (`refs/tags/v4.4.1`), not the branch. Without this rule, every publish fails with a "ref not allowed to deploy to this environment" error.
4. **Environment secrets:** none. Do NOT add `NPM_TOKEN`.

The manual-approval gate is automatic once "Required reviewers" is set — any job declaring `environment: npm-publish` pauses on entry until `maw629` approves.

### 3. Configure branch protection on `main` (out-of-tree, `gh api`)

Enable branch protection on `main` with the following settings:

- **Require a pull request before merging.** Required approvals: 1.
- **Require status checks to pass before merging.** Required checks (exact names from `tests.yml`):
  - `Node v20.x on ubuntu-latest`
  - `Node v22.x on ubuntu-latest`
  - `Node v24.x on ubuntu-latest`
  - `Measure performance impact of changes`
  - `Ensure typescript compatibility`
- **Block force pushes.**
- **Restrict pushes to `maw629`.**

This is a first-time setup for this repo; no existing protection rules are overwritten.

### 4. Workflow file — `.github/workflows/publish.yml` (in-tree, PR 1)

Add a new workflow with this exact shape:

```yaml
name: Publish to npm

on:
  release:
    types: [published]

permissions:
  contents: read
  id-token: write   # required to mint an OIDC token for npm

jobs:
  publish:
    name: npm publish
    runs-on: ubuntu-latest
    environment: npm-publish   # binds this job to the approval-gated env
    steps:
      - name: Checkout the tagged commit
        uses: actions/checkout@v7
        with:
          ref: ${{ github.event.release.tag_name }}

      - name: Set up Node
        uses: actions/setup-node@v7
        with:
          node-version: 22.x
          registry-url: 'https://registry.npmjs.org'

      - name: Pin npm version
        run: npm install -g npm@11.10.0

      - name: Install dependencies (honor lockfile)
        run: npm ci

      - name: Sanity-check tag matches package.json version
        run: |
          TAG="${{ github.event.release.tag_name }}"
          PKG_VERSION="v$(node -p "require('./package.json').version")"
          if [ "$TAG" != "$PKG_VERSION" ]; then
            echo "Tag ${TAG} does not match package.json version ${PKG_VERSION}" >&2
            exit 1
          fi

      - name: Publish
        run: npm publish --provenance
```

Notes on each choice:

- `on: release: types: [published]` fires only when a Release is *published* — a draft Release does not fire this.
- `permissions.id-token: write` lets GitHub Actions mint the OIDC JWT for this job. Without it, `npm publish` falls back to token auth and fails (no token is configured).
- `environment: npm-publish` is the manual approval gate. Removing this line silently removes the approval requirement.
- `actions/checkout@v7` and `actions/setup-node@v7` match the versions already in `tests.yml`.
- Node `22.x` matches the CI mid-range LTS.
- `npm install -g npm@11.10.0` matches the npm pin in `tests.yml` for consistency.
- `ref: ${{ github.event.release.tag_name }}` builds the tarball from the exact tagged commit.
- `npm ci` honors the lockfile, fails on drift.
- Tag-vs-`package.json` sanity check catches a forgotten version bump before shipping an inconsistent artifact.
- `npm publish --provenance` kept explicit even though `publishConfig.provenance: true` (Section 5) also enables it — belt-and-suspenders so removing the config flag does not silently disable provenance.
- No `--access public` on the CLI — covered by `publishConfig.access`.
- No `--tag latest` — `4.4.1` is a stable version; `latest` is npm's default.
- No `env: NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}` — OIDC replaces tokens entirely.

### 5. `package.json` `publishConfig` (in-tree, PR 1)

Add a new `publishConfig` block. There is currently none; both fields go in together:

```json
"publishConfig": {
  "access": "public",
  "provenance": true
}
```

`access: "public"` is required: npm defaults scoped packages to `restricted` access if this field is absent. `provenance: true` makes provenance the default behavior of `npm publish` for this package regardless of how it is invoked.

### 6. Version bump to `4.4.1` (in-tree, PR 2)

Single change: `package.json` `version` → `4.4.1`. No lib, no dist, no deps.

The `preversion` script (`clean + build + test:version`) runs locally before the bump; use `npm version patch` or a manual edit. The existing `postversion` script (`git push --no-verify && git push --tags --no-verify`) automatically pushes the commit and the `v4.4.1` tag.

`4.4.1` is functionally identical to `4.4.0`. The release notes for the GitHub Release should say so explicitly.

### 7. Publish act (after PR 2 merges)

1. Confirm tag `v4.4.1` is on `main` (the `postversion` script handles this).
2. Create the GitHub Release:
   ```bash
   gh release create v4.4.1 \
     --title "4.4.1" \
     --notes "Identical to 4.4.0. First release published via OIDC trusted publishing (no NPM_TOKEN)."
   ```
3. The `publish.yml` workflow fires and enters the `npm-publish` environment gate.
4. Approve the paused job in the Actions UI as `maw629`.
5. Workflow completes; `4.4.1` is published.

## Verification

### Static (after PR 1 merges, before PR 2)

1. `gh api /repos/pkg-nec/exceljs/environments/npm-publish` returns 200 with `reviewers` populated.
2. `gh api /repos/pkg-nec/exceljs/environments/npm-publish/deployment-branch-policies` includes both `{name: "main", type: "branch"}` and `{name: "v*.*.*", type: "tag"}`.
3. `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/publish.yml')); print('ok')"` prints `ok`.
4. `node -e "const p = require('./package.json'); console.log(p.publishConfig.provenance)"` prints `true`.
5. `node -e "const p = require('./package.json'); console.log(p.publishConfig.access)"` prints `public`.
6. Trusted Publisher on npm UI shows all four fields correctly.

### Live (after publish act)

7. `npm view @pkg-nec/exceljs@4.4.1 dist.integrity` — returns a hash (tarball exists).
8. `https://www.npmjs.com/package/@pkg-nec/exceljs?activeTab=code` — provenance badge visible.

## Risks & mitigations

- **Trusted Publisher field mismatch.** Any typo in Section 1's four fields breaks the publish with `E401`. Verify against the actual values in `.github/workflows/publish.yml` and the environment name string before saving.
- **`id-token: write` missing from workflow.** Without it, `npm publish` falls back to token auth and fails. Explicitly listed in Section 4; called out here so a future edit touching `permissions:` does not accidentally drop it.
- **Environment approval bypass.** If someone removes `environment: npm-publish` from the workflow, the approval gate silently disappears. Future PRs touching `publish.yml` should be treated as high-risk. A CODEOWNERS entry on `.github/workflows/publish.yml` would formalize this if solo-maintainer status changes.
- **Tag policy missing → every publish fails at env gate.** If only the branch rule is present, release-triggered runs (which use tag refs) cannot reach the environment. Verify both rules exist after Section 2 via `gh api`.
- **`preversion` script fails.** `npm version patch` will refuse to proceed if `test:version` fails. Fix the underlying build/test failure first; do not skip the preversion script.
- **Provenance requires public repo.** `pkg-nec/exceljs` is public. Flag in any future privacy discussion.

## Rollback

If the OIDC setup misbehaves after landing:

- **Delete the Trusted Publisher** on npm (Settings → Trusted Publisher → Remove). Publishing reverts to token-based; you can still `npm publish` by hand as before.
- **Delete the environment or the workflow file** to stop future automated publishes.
- Nothing in this spec touches already-published tarballs or existing versions.
