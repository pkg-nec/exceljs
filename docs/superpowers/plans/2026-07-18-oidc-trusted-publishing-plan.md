# OIDC Trusted Publishing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire OIDC-based trusted publishing between `pkg-nec/exceljs` (GitHub) and `@pkg-nec/exceljs` (npm) so future releases publish from GitHub Actions with no long-lived `NPM_TOKEN` secret, gated on GitHub Release publication plus a manual environment approval, and publish `4.4.1` as the first OIDC-sourced release.

**Architecture:** Two in-tree PRs (PR 1: `publish.yml` workflow + `publishConfig` in `package.json`; PR 2: version bump to `4.4.1`) plus three out-of-tree one-time configurations (npm Trusted Publisher, GitHub Environment `npm-publish`, branch protection on `main`). Live end-to-end verification is the `4.4.1` publish itself.

**Tech Stack:** GitHub Actions, `actions/checkout@v7`, `actions/setup-node@v7`, npm 11.10.0 (pinned, client-side OIDC publish), Node.js 22.x.

## Global Constraints

- **Working branch for PR 1:** `chore/oidc-trusted-publishing`. Create from `main` before starting Task 1.
- **Working branch for PR 2:** `chore/bump-4.4.1`. Create from `main` after PR 1 merges.
- **Merge target for both PRs:** `main`. Branch protection (Task 3) requires PR + all five CI status checks.
- **Trusted Publisher fields (exact spelling — must match across npm UI, GitHub Environment name, and workflow YAML):**
  - Organization or user: `pkg-nec`
  - Repository: `exceljs`
  - Workflow filename: `publish.yml`
  - Environment name: `npm-publish`
- **Environment deployment policy:** MUST include BOTH a `Branch` rule pattern `main` AND a `Tag` rule pattern `v*.*.*`. Release-triggered workflows run against the tag ref; the tag rule is what actually admits publishes.
- **No `NPM_TOKEN` secret** anywhere — not in repo secrets, not in the `npm-publish` environment.
- **No changes** to `.github/workflows/tests.yml`, `lib/`, `dist/`, `dependencies`, `devDependencies`, `package-lock.json`, or `engines.node`.
- **`files` field** in `package.json` stays as-is.
- **Node version in the publish workflow:** `22.x`. npm pin: `11.10.0` (matching `tests.yml`).
- **Actions versions:** `actions/checkout@v7` and `actions/setup-node@v7` (matching `tests.yml`).

---

### Task 1: Add `publishConfig` to `package.json` (PR 1)

**Files:**
- Modify: `package.json`

**Interfaces:**
- Consumes: nothing.
- Produces: a `package.json` whose `publishConfig` block reads `{ "access": "public", "provenance": true }` and is otherwise byte-identical to base. This is what tells npm (a) to publish the scoped package as public and (b) to attach provenance attestations by default.

- [ ] **Step 1: Check out the working branch**

  ```bash
  git checkout main && git pull
  git checkout -b chore/oidc-trusted-publishing
  ```

  Expected: you are on `chore/oidc-trusted-publishing`, clean working tree.

- [ ] **Step 2: Make the edit**

  `package.json` currently has no `publishConfig` block. Add one after the `postversion` line in the `scripts` block — specifically, after the closing `}` of `scripts` and before `"husky"`. The result should be:

  ```json
    "publishConfig": {
      "access": "public",
      "provenance": true
    },
  ```

  Placed between `scripts` and `husky` blocks. Do not reorder or reformat anything else in the file.

- [ ] **Step 3: Verify both fields parse correctly**

  ```bash
  node -e "const p = require('./package.json'); console.log(p.publishConfig.provenance)"
  # → true

  node -e "const p = require('./package.json'); console.log(p.publishConfig.access)"
  # → public
  ```

  Both must print exactly as shown.

- [ ] **Step 4: Verify JSON validity end-to-end**

  ```bash
  node -e "JSON.parse(require('fs').readFileSync('./package.json', 'utf8')); console.log('ok')"
  # → ok
  ```

- [ ] **Step 5: Verify no unrelated fields drifted**

  ```bash
  git diff main -- package.json | grep -E '^\+\s*"(name|version|description|private|license|author|repository|engines|main|browser|types|files|scripts|husky|lint-staged|keywords|dependencies|devDependencies)"'
  ```

  Expected: no output. Only the `publishConfig` block should be in the diff.

- [ ] **Step 6: Commit**

  ```bash
  git add package.json
  git commit -m "chore: add publishConfig with access:public and provenance:true"
  ```

---

### Task 2: Create the `publish.yml` workflow (PR 1)

**Files:**
- Create: `.github/workflows/publish.yml`

**Interfaces:**
- Consumes: Task 1's `publishConfig.provenance: true` (means `--provenance` on the CLI is redundant but the workflow keeps it for defense-in-depth).
- Produces: a workflow that fires on `release: published`, is gated on the `npm-publish` GitHub Environment (manual approval), checks out the exact tagged commit, verifies the tag matches `package.json`'s version, and runs `npm publish --provenance` using an OIDC token. Requires the npm-side Trusted Publisher (Task 4) and the GitHub Environment (Task 5) to exist before it can actually publish — but the file lands safely without them because no release is published during this task.

- [ ] **Step 1: Create the workflow file**

  Create `.github/workflows/publish.yml` with EXACTLY this content:

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

  Implementation notes:
  - Do NOT add `--access public` to the publish step — covered by `publishConfig.access`.
  - Do NOT add `--tag latest` — stable versions use `latest` by default.
  - Do NOT add `env: NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}` — there is no token; OIDC replaces it.
  - Do NOT delete or reformat the inline comments on `id-token: write` and `environment: npm-publish` — they document non-obvious requirements.

- [ ] **Step 2: Verify YAML parses**

  ```bash
  python3 -c "import yaml; yaml.safe_load(open('.github/workflows/publish.yml')); print('ok')"
  # → ok
  ```

- [ ] **Step 3: Verify critical fields are present**

  ```bash
  grep -E "^\s*id-token: write(\s|$|#)" .github/workflows/publish.yml
  grep -E "^\s*environment: npm-publish(\s|$|#)" .github/workflows/publish.yml
  grep -F 'ref: ${{ github.event.release.tag_name }}' .github/workflows/publish.yml
  grep -F 'npm publish --provenance' .github/workflows/publish.yml
  ```

  Each grep must return at least one line. If any returns zero lines, the workflow is missing a critical setting — do not proceed.

- [ ] **Step 4: Verify no secret is referenced**

  ```bash
  grep -E "NPM_TOKEN|secrets\." .github/workflows/publish.yml
  ```

  Expected: no output. Any hit means a secret leaked in — remove it before committing.

- [ ] **Step 5: Verify the existing test workflow is untouched**

  ```bash
  git diff main -- .github/workflows/tests.yml .github/workflows/asset-size.yml
  ```

  Expected: no output.

- [ ] **Step 6: Commit**

  ```bash
  git add .github/workflows/publish.yml
  git commit -m "ci: add OIDC-based publish workflow gated on npm-publish environment"
  ```

---

### Task 3: Push PR 1 and wait for CI

**Files:** none modified.

**Interfaces:**
- Consumes: Tasks 1 and 2 committed on `chore/oidc-trusted-publishing`.
- Produces: a merged PR on `main` with green CI, making the in-tree wiring live.

Note: branch protection (Task 6) may not be in place yet when this PR is opened; that is fine. The PR validates CI correctness regardless. Branch protection should be configured (Task 6) before merging if possible, but the CI green check is the critical gate either way.

- [ ] **Step 1: Confirm the branch diff vs. `main`**

  ```bash
  git log --oneline main..HEAD
  git diff --stat main..HEAD
  ```

  Expected commits (2):
  - `chore: add publishConfig with access:public and provenance:true`
  - `ci: add OIDC-based publish workflow gated on npm-publish environment`

  Expected files changed:
  - `package.json` (publishConfig added)
  - `.github/workflows/publish.yml` (created)

  If any file OUTSIDE this list appears, investigate before pushing.

- [ ] **Step 2: Push the branch**

  ```bash
  git push -u origin chore/oidc-trusted-publishing
  ```

- [ ] **Step 3: Open PR against `main`**

  ```bash
  gh pr create --base main --head chore/oidc-trusted-publishing \
    --title "ci: wire OIDC trusted publishing to npm" \
    --body "$(cat <<'EOF'
  Implements docs/superpowers/specs/2026-07-18-oidc-trusted-publishing-design.md.

  ## What changed
  - `.github/workflows/publish.yml`: new workflow, fires on `release: published`, runs against the `npm-publish` GitHub Environment, checks out the tagged commit, sanity-checks tag ↔ `package.json` version, and runs `npm publish --provenance` using an OIDC token (no `NPM_TOKEN` secret).
  - `package.json`: `publishConfig` block added with `"access": "public"` (required for scoped packages) and `"provenance": true`.

  ## What did NOT change
  - `lib/`, `dist/`, `dependencies`, `devDependencies`, `package-lock.json`, `engines.node`.
  - The existing test CI workflow (`tests.yml`) — same trigger, same matrix.
  - No `NPM_TOKEN` secret is added or referenced anywhere.

  ## Out-of-tree setup required BEFORE the workflow can actually publish
  See the spec (Sections 1–3) and plan (Tasks 4–6). This PR is safe to merge without them — the workflow only fires on `release: published`, which is a deliberate later act.
  1. Register a Trusted Publisher on npmjs.com for `@pkg-nec/exceljs` with Organization=pkg-nec, Repository=exceljs, Workflow=publish.yml, Environment=npm-publish.
  2. Create a GitHub Actions Environment `npm-publish` with reviewer `maw629` and deployment rules `Branch: main` + `Tag: v*.*.*`.
  3. Enable branch protection on `main` (PR required + all five CI checks).

  ## Live verification
  The `4.4.1` publish (Tasks 7–8 in the plan) is the end-to-end proof of the trust chain.
  EOF
  )"
  ```

- [ ] **Step 4: Wait for CI to go green**

  ```bash
  gh pr checks --watch
  ```

  Expected: all five checks (`Node v20.x on ubuntu-latest`, `Node v22.x on ubuntu-latest`, `Node v24.x on ubuntu-latest`, `Measure performance impact of changes`, `Ensure typescript compatibility`) go green. This PR introduces no code changes to `lib/`, so a CI failure would indicate an infrastructure issue — investigate before merging.

- [ ] **Step 5: Merge PR 1**

  ```bash
  gh pr merge --squash --delete-branch
  ```

  Or merge via the GitHub UI. Pull `main` afterward:

  ```bash
  git checkout main && git pull
  ```

---

### Task 4: Register Trusted Publisher on npmjs.com (out-of-tree, user action)

**Files:** none. Manual configuration on `npmjs.com`.

**Interfaces:**
- Consumes: nothing in-repo.
- Produces: a Trusted Publisher record on npm's side that admits OIDC handshakes from `publish.yml` on `pkg-nec/exceljs` against the `npm-publish` environment.

This task can run at any point after Task 3's PR is merged (or in parallel). It does not need to precede Tasks 5–6.

- [ ] **Step 1: Log in to npmjs.com**

  Log in as an account with admin rights on the `@pkg-nec` org — the same account used to publish `4.4.0`.

- [ ] **Step 2: Navigate to the package's Trusted Publisher settings**

  Open `https://www.npmjs.com/package/@pkg-nec/exceljs`, click the **Settings** tab, then click **Trusted Publisher** in the left-hand nav.

- [ ] **Step 3: Add a new publisher — GitHub Actions**

  Click **Add publisher**, choose **GitHub Actions**, and fill in exactly:

  - **Organization or user:** `pkg-nec`
  - **Repository:** `exceljs`
  - **Workflow filename:** `publish.yml`
  - **Environment name:** `npm-publish`

  Every field is a case-sensitive string match against what the workflow's OIDC JWT will present. A single character wrong here will cause `npm ERR! code E401` at publish time.

- [ ] **Step 4: Save**

  Click **Save** (or equivalent primary button). npm displays the publisher in the list.

- [ ] **Step 5: Record the four saved values**

  Note the four values or take a screenshot. Task 8's verification will confirm these against what the workflow file declares.

  There is no `npm` CLI command that reads back Trusted Publisher configuration — Task 8's verification is UI-only for the npm side.

---

### Task 5: Create the `npm-publish` GitHub Environment (out-of-tree, `gh api`)

**Files:** none. Configuration on `github.com/pkg-nec/exceljs` via `gh api`.

**Interfaces:**
- Consumes: nothing in-repo.
- Produces: a GitHub Actions Environment named `npm-publish` with (a) required reviewer `maw629`, (b) deployment-branch policy allowing `main` as a branch and `v*.*.*` as a tag pattern, (c) NO environment secrets.

- [ ] **Step 1: Create the environment (empty configuration)**

  ```bash
  gh api --method PUT \
    -H "Accept: application/vnd.github+json" \
    /repos/pkg-nec/exceljs/environments/npm-publish \
    --input - <<'EOF'
  {
    "deployment_branch_policy": {
      "protected_branches": false,
      "custom_branch_policies": true
    }
  }
  EOF
  ```

  Expected: a JSON blob describing the created environment. `deployment_branch_policy.custom_branch_policies: true` switches the environment into "selected branches and tags" mode.

- [ ] **Step 2: Add required reviewer**

  Look up `maw629`'s numeric user ID (needed for the API — reviewer targets are ID-based):

  ```bash
  gh api /users/maw629 --jq .id
  # → prints a numeric ID, e.g. 12345678
  ```

  Then set the reviewer:

  ```bash
  MAW_ID=$(gh api /users/maw629 --jq .id)
  gh api --method PUT \
    -H "Accept: application/vnd.github+json" \
    /repos/pkg-nec/exceljs/environments/npm-publish \
    --input - <<EOF
  {
    "wait_timer": 0,
    "reviewers": [
      { "type": "User", "id": ${MAW_ID} }
    ],
    "deployment_branch_policy": {
      "protected_branches": false,
      "custom_branch_policies": true
    }
  }
  EOF
  ```

  Expected: JSON response includes a `reviewers` array with your user entry.

- [ ] **Step 3: Add the `main` branch policy**

  ```bash
  gh api --method POST \
    -H "Accept: application/vnd.github+json" \
    /repos/pkg-nec/exceljs/environments/npm-publish/deployment-branch-policies \
    --input - <<'EOF'
  {
    "name": "main",
    "type": "branch"
  }
  EOF
  ```

  Expected: JSON response with the created policy including an `id` field.

- [ ] **Step 4: Add the `v*.*.*` tag policy**

  ```bash
  gh api --method POST \
    -H "Accept: application/vnd.github+json" \
    /repos/pkg-nec/exceljs/environments/npm-publish/deployment-branch-policies \
    --input - <<'EOF'
  {
    "name": "v*.*.*",
    "type": "tag"
  }
  EOF
  ```

  Expected: JSON response with the created policy.

  This tag rule is what actually admits release-triggered workflow runs — release workflows execute against the tag ref, not the branch. Without this rule, every publish attempt fails at the environment gate.

- [ ] **Step 5: Confirm no environment secrets exist**

  ```bash
  gh api /repos/pkg-nec/exceljs/environments/npm-publish/secrets --jq '.total_count, .secrets'
  ```

  Expected: `0` followed by `[]`. If anything else appears, an old `NPM_TOKEN` (or similar) leaked in — delete it:

  ```bash
  # only if a secret is present:
  gh api --method DELETE /repos/pkg-nec/exceljs/environments/npm-publish/secrets/<SECRET_NAME>
  ```

---

### Task 6: Configure branch protection on `main` (out-of-tree, `gh api`)

**Files:** none.

**Interfaces:**
- Consumes: nothing in-repo.
- Produces: branch protection on `main` requiring PR + all five CI status checks, blocking force pushes, restricting direct pushes to `maw629`.

Note: the exact status check names must match what GitHub records after at least one CI run. If PR 1 (Task 3) has already run, these names are known. If configuring before any CI run, add the checks by name anyway — GitHub will accept them and they'll activate once the first matching run completes.

- [ ] **Step 1: Look up `maw629`'s user ID (if not already known from Task 5)**

  ```bash
  gh api /users/maw629 --jq .id
  ```

- [ ] **Step 2: Apply branch protection**

  ```bash
  MAW_ID=$(gh api /users/maw629 --jq .id)
  gh api --method PUT \
    -H "Accept: application/vnd.github+json" \
    /repos/pkg-nec/exceljs/branches/main/protection \
    --input - <<EOF
  {
    "required_status_checks": {
      "strict": true,
      "contexts": [
        "Node v20.x on ubuntu-latest",
        "Node v22.x on ubuntu-latest",
        "Node v24.x on ubuntu-latest",
        "Measure performance impact of changes",
        "Ensure typescript compatibility"
      ]
    },
    "enforce_admins": false,
    "required_pull_request_reviews": {
      "required_approving_review_count": 1,
      "dismiss_stale_reviews": false
    },
    "restrictions": {
      "users": ["maw629"],
      "teams": [],
      "apps": []
    },
    "allow_force_pushes": false,
    "allow_deletions": false
  }
  EOF
  ```

  Expected: a JSON blob describing the updated branch protection. Check that `required_status_checks.contexts` lists all five check names.

- [ ] **Step 3: Verify protection is active**

  ```bash
  gh api /repos/pkg-nec/exceljs/branches/main/protection \
    --jq '{required_pr: .required_pull_request_reviews.required_approving_review_count, force_push_allowed: .allow_force_pushes.enabled, checks: [.required_status_checks.contexts[]]}'
  ```

  Expected:
  ```json
  {
    "required_pr": 1,
    "force_push_allowed": false,
    "checks": [
      "Node v20.x on ubuntu-latest",
      "Node v22.x on ubuntu-latest",
      "Node v24.x on ubuntu-latest",
      "Measure performance impact of changes",
      "Ensure typescript compatibility"
    ]
  }
  ```

---

### Task 7: Version bump to `4.4.1` (PR 2)

**Files:**
- Modify: `package.json` (version field only)

**Interfaces:**
- Consumes: PR 1 merged on `main`, Tasks 4–6 complete.
- Produces: `package.json` with `"version": "4.4.1"`, a `v4.4.1` tag on `main`, and an open PR (or direct merge if branch protection isn't yet fully enforced).

`4.4.1` is functionally identical to `4.4.0`. The only change is the version number.

- [ ] **Step 1: Create the working branch**

  ```bash
  git checkout main && git pull
  git checkout -b chore/bump-4.4.1
  ```

- [ ] **Step 2: Bump the version**

  Option A (preferred — runs preversion checks automatically):
  ```bash
  npm version patch --no-git-tag-version
  ```
  The `--no-git-tag-version` flag prevents `npm version` from creating the commit and tag itself — we'll do that via PR. Then manually inspect that `package.json` now shows `"version": "4.4.1"`.

  Option B (manual edit): open `package.json`, change line 3 from `"version": "4.4.0"` to `"version": "4.4.1"`. Run `node -e "JSON.parse(require('fs').readFileSync('./package.json','utf8')); console.log('ok')"` to confirm valid JSON.

  Note: Option A's `preversion` script runs `clean + build + test:version`. This may take several minutes. If it fails, fix the underlying issue before proceeding — do not skip the preversion script.

- [ ] **Step 3: Verify only the version field changed**

  ```bash
  git diff -- package.json
  ```

  Expected: exactly one line changed — `"version": "4.4.0"` → `"version": "4.4.1"`. If any other field appears in the diff, investigate.

- [ ] **Step 4: Commit**

  ```bash
  git add package.json
  git commit -m "chore: bump version to 4.4.1"
  ```

- [ ] **Step 5: Push and open PR**

  ```bash
  git push -u origin chore/bump-4.4.1
  gh pr create --base main --head chore/bump-4.4.1 \
    --title "chore: bump version to 4.4.1" \
    --body "Version bump only. 4.4.1 is functionally identical to 4.4.0. First release published via OIDC trusted publishing."
  ```

- [ ] **Step 6: Wait for CI and merge**

  ```bash
  gh pr checks --watch
  ```

  Once green, merge:

  ```bash
  gh pr merge --squash --delete-branch
  git checkout main && git pull
  ```

- [ ] **Step 7: Create and push the `v4.4.1` tag**

  The `postversion` script would normally handle this, but since we used `--no-git-tag-version` in Step 2, tag manually after the PR merges:

  ```bash
  git tag v4.4.1
  git push origin v4.4.1
  ```

  Expected: `git log --oneline -1` shows the merge commit; `git tag -l "v4.4.1"` returns `v4.4.1`.

---

### Task 8: Publish `4.4.1` and verify the trust chain

**Files:** none modified.

**Interfaces:**
- Consumes: Tasks 3–7 all complete — PR 1 merged, Trusted Publisher configured, Environment configured, branch protection active, `v4.4.1` tag on `main`.
- Produces: `@pkg-nec/exceljs@4.4.1` published on npm with a provenance attestation, proving the OIDC trust chain end-to-end.

- [ ] **Step 1: Create the GitHub Release**

  ```bash
  gh release create v4.4.1 \
    --title "4.4.1" \
    --notes "Identical to 4.4.0. First release published via OIDC trusted publishing (no NPM_TOKEN)."
  ```

  Expected: release created at `https://github.com/pkg-nec/exceljs/releases/tag/v4.4.1`. The `publish.yml` workflow fires immediately.

- [ ] **Step 2: Approve the environment gate**

  Navigate to `https://github.com/pkg-nec/exceljs/actions`, find the running `Publish to npm` workflow for `v4.4.1`, and approve the paused `npm-publish` job as `maw629`.

  The job will proceed through: checkout → setup Node → pin npm → `npm ci` → tag sanity check → `npm publish --provenance`.

- [ ] **Step 3: Confirm the workflow completed successfully**

  ```bash
  gh run list --workflow=publish.yml --limit=1
  ```

  Expected: status `completed`, conclusion `success`.

  If the workflow failed, check the run logs:
  ```bash
  gh run view --log --workflow=publish.yml
  ```

  Common failure causes:
  - `npm ERR! code E401` → Trusted Publisher field mismatch (Task 4). Verify all four fields against `.github/workflows/publish.yml`.
  - "ref not allowed to deploy to this environment" → Tag policy missing (Task 5, Step 4). Add the `v*.*.*` tag rule.
  - Tag/version mismatch → `package.json` version doesn't match `v4.4.1`. Verify Task 7.

- [ ] **Step 4: Verify tarball exists on npm**

  ```bash
  npm view @pkg-nec/exceljs@4.4.1 dist.integrity
  ```

  Expected: a `sha512-...` hash. If this returns nothing, the publish did not complete.

- [ ] **Step 5: Verify provenance attestation**

  Open in a browser:
  ```
  https://www.npmjs.com/package/@pkg-nec/exceljs?activeTab=code
  ```

  Expected: a provenance badge is visible on the `4.4.1` version page, linking back to the `pkg-nec/exceljs` repository and the specific workflow run.

- [ ] **Step 6: Static verification — GitHub Environment**

  ```bash
  gh api /repos/pkg-nec/exceljs/environments/npm-publish \
    --jq '{name, reviewers: [.protection_rules[]? | select(.type=="required_reviewers") | .reviewers[]? | .reviewer.login]}'
  ```

  Expected: `{"name":"npm-publish","reviewers":["maw629"]}`.

- [ ] **Step 7: Static verification — deployment branch/tag policies**

  ```bash
  gh api /repos/pkg-nec/exceljs/environments/npm-publish/deployment-branch-policies \
    --jq '.branch_policies[] | {name, type}'
  ```

  Expected: exactly TWO entries:
  - `{"name":"main","type":"branch"}`
  - `{"name":"v*.*.*","type":"tag"}`

---

## Definition of Done (from the spec's Verification section)

1. ✅ npm Trusted Publisher shows `pkg-nec / exceljs / publish.yml / npm-publish` — Task 4.
2. ✅ GitHub Environment `npm-publish` exists with reviewer `maw629` and both deployment policies — Task 5, verified in Task 8 Steps 6–7.
3. ✅ Branch protection on `main` active with PR + all five CI checks — Task 6.
4. ✅ `publish.yml` YAML parses and contains `id-token: write` + `environment: npm-publish` — Task 2 Step 2–3.
5. ✅ `package.json` `publishConfig` has `access: public` and `provenance: true` — Task 1 Step 3.
6. ✅ `@pkg-nec/exceljs@4.4.1` published with provenance attestation — Task 8 Steps 4–5.
