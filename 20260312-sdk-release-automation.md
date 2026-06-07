# Meta
[meta]: #meta
- **Name:** Release Automation - SDKs
- **Start Date:** 2026-03-12
- **Author(s):** [@SoulPancake](https://github.com/SoulPancake)
- **Status:** Draft 
- **RFC Pull Request:** https://github.com/openfga/rfcs/pull/33
- **Relevant Issues:**
  - https://github.com/openfga/sdk-generator/issues/679
- **Supersedes:** N/A

## Table of Contents
- [Summary](#summary)
- [Definitions](#definitions)
- [Motivation](#motivation)
- [What it is](#what-it-is)
- [How it Works](#how-it-works)
- [Security](#security)
- [Failure Modes and Rollback Procedures](#failure-modes-and-rollback-procedures)
- [Migration](#migration)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Prior Art](#prior-art)
- [Additional Enhancements](#additional-enhancements)
- [Reusable Workflow Across the Organisation](#reusable-workflow-across-the-organisation)

## Summary
[summary]: #summary

Releasing a new version of an OpenFGA SDK is currently a manual process that requires updating the changelog, bumping version constants in source files, creating a signed git tag, and pushing it. This overhead discourages frequent shipping and leads to large, batched releases.

This RFC proposes adopting [Release Please](https://github.com/googleapis/release-please) to automate the release preparation and execution for all OpenFGA SDKs. A maintainer triggers the workflow from the GitHub Actions UI (`workflow_dispatch`), selecting a bump type (patch, minor, major, explicit, or auto). Release Please then creates a **Release PR** containing the changelog update and all version bumps. When the Release PR is merged, Release Please finalizes the release by creating a git tag, which triggers the existing publish workflows.

## Definitions
[definitions]: #definitions

- **[Conventional Commits](https://www.conventionalcommits.org/):** A specification for commit/PR titles (e.g., `feat:`, `fix:`, `chore:`) that enables automated changelog generation and semantic version calculation.

- **[Release Please](https://github.com/googleapis/release-please):** A Google-maintained tool that automates release preparation by creating and maintaining a Release PR. It parses Conventional Commits to generate changelogs and determine the next version.

- **Release PR:** A pull request created and maintained by Release Please that contains the changelog update, version bumps, and manifest changes for a pending release. Merging this PR is the "sign-off" that triggers tag creation.

- **`.release-please-manifest.json`:** A JSON file at the repository root that tracks the current version. Release Please reads and updates this file as part of the Release PR.

- **`x-release-please-version`:** A marker comment placed inline next to version constants in source files. Release Please scans for this marker and automatically updates the adjacent version string during release preparation.

- **Default Changelog Format (`changelog-type: "default"`):** The built-in Release Please changelog format that groups entries by Conventional Commit type into named sections (e.g., `Added`, `Fixed`, `Changed`). Unlike `changelog-type: "github"` — which bypasses section grouping entirely and calls GitHub's release notes API — the `default` type respects the `changelog-sections` configuration, giving full control over which commit types appear, what they are labelled, and whether they are hidden. This is the format used across all OpenFGA SDKs.

- **GitHub App Token:** A short-lived token minted at the start of each workflow run by a dedicated GitHub App installed on the repository. The App is granted only `contents: write` and `pull-requests: write` permissions. The token expires after the workflow run completes, limiting the blast radius of any credential compromise to a single run. The App's `APP_ID` and `APP_PRIVATE_KEY` are stored as repository (or organization) secrets.

## Motivation
[motivation]: #motivation

### Why should we do this?

The current release process for OpenFGA SDKs is entirely manual. A maintainer must:

1. Update `CHANGELOG.md` by hand.
2. Bump the version constant in language-specific source files.
3. Create a signed git tag.
4. Push the tag to trigger CI/CD publishing.

This manual overhead creates several problems:

- **Low release velocity:** The effort required to ship a single change is disproportionately high, leading to batched releases and delayed delivery of features and fixes to users.
- **Maintainer burden:** Each release involves repetitive, error-prone steps across multiple repositories.
- **Inconsistency:** Manual changelog entries occasionally diverge across SDK repos, and the changelog format itself is not standardized.

### What use cases does it support?

- A maintainer wants to release a patch for a critical bug fix within minutes, not hours.
- A maintainer wants to release a new minor version containing several features with a single click and one PR review.
- A maintainer wants to perform a major version bump with explicit control over the version number.
- The team wants a consistent, auditable release process across Go, .NET, JavaScript/TypeScript, Python, and Java SDKs.

### What is the expected outcome?

A standardized, near-zero-touch release process where:

1. A maintainer triggers the release workflow from the GitHub Actions UI.
2. Release Please creates a Release PR with all necessary changes.
3. A team member reviews and merges the PR.
4. The merge automatically creates a git tag, which triggers the existing publish pipeline.

## What it is
[what-it-is]: #what-it-is

This proposal introduces a two-phase release workflow for all OpenFGA SDKs, built on top of Release Please.

**Target persona:** Project contributor / SDK maintainer.

### High-Level Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 1: Release Preparation (New)                             │
│                                                                 │
│  Triggers:                                                      │
│  • workflow_dispatch — Maintainer triggers from GitHub UI to    │
│    create or update the Release PR with a specific bump type.   │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  Inputs:                                            │     │
│     │  • bump-type: auto|patch|minor|major|explicit       │     │
│     │  • release-version: (e.g. 1.2.3 or 1.4.0-beta.1)    │     │
│     └─────────────────────────────────────────────────────┘     │
│                                                                 │
│  Release Please creates/updates a Release PR containing:        │
│  • Updated CHANGELOG.md (default changelog format)              │
│  • Bumped version in .release-please-manifest.json              │
│  • Bumped version in all x-release-please-version markers       │
│                                                                 │
│  Maintainer reviews and merges the Release PR                   │
│                                                                 │
│  • push to main — Job only runs when the Release PR merge       │
│    commit lands (chore: release ...). Release Please finalizes  │
│    the release: creates the git tag and GitHub Release.         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 2: Publish (Existing)                                    │
│                                                                 │
│  Trigger: v* tag created by Release Please                      │
│                                                                 │
│  1. Run test suite and checks                                   │
│  2. Publish to language-specific registry                       │
└─────────────────────────────────────────────────────────────────┘
```

### The Release PR Model

This proposal uses the **Release PR model** (as opposed to a direct-push model) for the standard release flow. The key properties are:

- **Branch protection–fully respected.** The GitHub App acts as a standard contributor; **no actor ever pushes directly to `main`** — not even for version overrides. All changes, including `Release-As` commits, land on a short-lived `release` staging branch and reach `main` exclusively through the reviewed Release PR. See [Workaround Flow](#workaround-flow-explicit-version-overrides-without-bot-write-access-to-main) for the full mechanics.
- **Reviewable diff.** The changelog and version bump are visible in a PR before the tag is ever created.
- **Audit trail.** The PR provides a permanent, reviewable history of who approved the release and what the changelog diff contained.
- **Consistent with team norms.** The release goes through the same PR review process as any other code change.

### Version Bumping with `x-release-please-version`

Release Please uses inline marker comments to locate and update version constants across language-specific files. By placing an `x-release-please-version` comment next to any version string, Release Please will automatically update it in the Release PR. The marker is language-agnostic — it works with any comment syntax:

```
VERSION = "0.7.2"  # x-release-please-version    (Python, properties files)
const SdkVersion = "0.7.2" // x-release-please-version  (Go, Java, C#, TypeScript)
<Version>0.7.2</Version><!-- x-release-please-version -->  (XML / .csproj)
```

Each SDK repository annotates every file that contains a version constant with this marker. When Release Please prepares a release, it scans for these markers and updates the version string inline, producing a clean diff in the Release PR. The list of files to scan is declared in `release-please-config.json` under the `extra-files` key.

### Changelog Format

The changelog format will be standardized to the **default Release Please changelog format** (`changelog-type: "default"`) across all SDKs. This format groups entries by Conventional Commit type into named sections and is configured via the `changelog-sections` key in `release-please-config.json`. It is deliberately chosen over `changelog-type: "github"`, which bypasses section grouping entirely and calls GitHub's release notes API without respecting per-type configuration.

The section mapping used across all SDKs:

| Commit type | Section | Visible |
|---|---|---|
| `feat` | Added | ✅ |
| `fix` | Fixed | ✅ |
| `perf` | Changed | ✅ |
| `refactor` | Changed | ✅ |
| `revert` | Removed | ✅ |
| `docs` | Documentation | ✅ |
| `test` | Tests | hidden |
| `ci` | CI | hidden |
| `chore` | Miscellaneous | hidden |

Example output:

```markdown
## 0.7.3 (2026-03-12)

### Added
* feat: add support for batch check ([#101](https://github.com/openfga/go-sdk/pull/101))

### Fixed
* fix: correct retry logic for transient errors ([#105](https://github.com/openfga/go-sdk/pull/105))

### Documentation
* docs: update API reference ([#108](https://github.com/openfga/go-sdk/pull/108))
```

Additional notes (contributor acknowledgments, migration tips, usage examples) are curated directly in `CHANGELOG.md` while the Release PR is under review.

> **Note — release notes come from the changelog, not the PR description.** Early in our adoption we leaned on a release-please behaviour where the GitHub Release notes were generated via GitHub's API and the **Release PR description** was captured into them — so the guidance back then was to also edit the PR description to read exactly as you wanted the release notes (ideally mirroring the changelog). With the changelog-based flow (see [Signed Tags, Draft Releases, and Undraft-on-Publish](#how-it-works)), the `post-release` job builds the notes from the matching `CHANGELOG.md` section plus GitHub's auto-generated "what's changed". The **changelog is now the single source of truth**, so we no longer need to maintain the PR description separately — curating `CHANGELOG.md` in the Release PR is sufficient.

### Explicit Version Overrides

Release Please auto-calculates the version bump from Conventional Commits (e.g., `feat:` triggers a minor bump, `fix:` triggers a patch bump). However, there are cases where the auto-calculated bump is not what we want.

The workflow supports explicit overrides via the `bump-type` input on the `workflow_dispatch` trigger:

- **`auto`** — Let Release Please determine the bump from commit history (default).
- **`patch`** — Force a patch bump.
- **`minor`** — Force a minor bump.
- **`major`** — Force a major bump.
- **`explicit`** — Supply an exact version string (e.g., `1.2.3` or `1.4.0-beta.1`) via the `release-version` input.

For non-`auto` bump types, the workflow computes the next version from the current version in `.release-please-manifest.json`, creates an empty commit with the `Release-As: X.Y.Z` trailer, and pushes it to the `release` staging branch (never directly to `main`). Release Please then picks up the trailer and uses the specified version instead of auto-calculating. The PR is subsequently retargeted to `main` so it goes through the normal review flow. See [Workaround Flow](#workaround-flow-explicit-version-overrides-without-bot-write-access-to-main) for the full mechanics.

> **Note — releasing changes that aren't user-facing.** `auto` derives the bump from commit history, and commit types that are hidden from the changelog (`chore`, `docs`, `ci`, `test`, `refactor`) are treated as *non-releasing*. If the only changes since the last release are of those types, `auto` concludes no release is warranted and will not open a Release PR. To ship such a release anyway — for example a docs-only update or a dependency bump — choose any bump rule **other than `auto`** (`patch`, `minor`, `major`, or `explicit`), which forces the version forward via the `Release-As` trailer regardless of commit types.

### Conventional Commits Validation

Release Please depends on [Conventional Commits](https://www.conventionalcommits.org/) to determine version bumps and generate changelogs. To ensure every PR merged to `main` conforms to the specification, a **PR title validation check** will be added as a **required status check** on all SDK repositories.

This pattern is already in use at [openfga/terraform-provider-openfga](https://github.com/openfga/terraform-provider-openfga) and will be adopted across all SDKs:

```yaml
name: Pull Request

on:
  pull_request:
    types:
      - opened
      - reopened
      - synchronize
      - edited
    branches:
      - main

jobs:
  validate-pr-title:
    name: Validate PR Title
    runs-on: ubuntu-latest
    permissions:
      pull-requests: read
    steps:
      - name: PR Conventional Commit Validation
        uses: ytanikin/pr-conventional-commits@fda730cb152c05a849d6d84325e50c6182d9d1e9 # v1.5.1
        with:
          task_types: '["feat","fix","docs","test","refactor","ci","perf","chore","revert"]'
          add_label: 'false'
```

The `validate-pr-title` job must be configured as a **required status check** in each repository's branch protection rules. This ensures that no PR can be merged to `main` without a properly formatted title, which in turn guarantees that Release Please can always generate an accurate changelog entry.

## How it Works
[how-it-works]: #how-it-works

### Workflow Configuration

Each SDK repository will contain a Release Please configuration that defines:

1. **`release-please-config.json`** — Specifies the release type, changelog path, version file paths with `x-release-please-version` markers, and any extra files to update.

2. **`.release-please-manifest.json`** — Tracks the current released version. Release Please reads this to determine the base version and writes the new version during release preparation.

Example `release-please-config.json`:
```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "release-type": "go",
  "packages": {
    ".": {
      "include-component-in-tag": false,
      "changelog-path": "CHANGELOG.md",
      "changelog-type": "default",
      "bump-minor-pre-major": true,
      "bump-patch-for-minor-pre-major": true,
      "changelog-sections": [
        { "type": "feat",     "section": "Added",         "hidden": false },
        { "type": "fix",      "section": "Fixed",         "hidden": false },
        { "type": "perf",     "section": "Changed",       "hidden": false },
        { "type": "refactor", "section": "Changed",       "hidden": false },
        { "type": "revert",   "section": "Removed",       "hidden": false },
        { "type": "docs",     "section": "Documentation", "hidden": false },
        { "type": "test",     "section": "Tests",         "hidden": true  },
        { "type": "ci",       "section": "CI",            "hidden": true  },
        { "type": "chore",    "section": "Miscellaneous", "hidden": true  }
      ],
      "extra-files": [
        { "type": "generic", "path": "version/version.go" },
        { "type": "json",    "path": "package.json",       "jsonpath": "$.version" },
        { "type": "toml",    "path": "pyproject.toml",     "jsonpath": "$.project.version" },
        { "type": "generic", "path": "gradle.properties" }
      ]
    }
  }
}
```

> **Note on `extra-files` types:** Release Please supports three entry types under `extra-files`. `"generic"` updates any file containing an `x-release-please-version` marker comment. `"json"` and `"toml"` use a `jsonpath` expression to locate the version field directly, without needing a marker comment — useful for `package.json` and `pyproject.toml` where adding a comment next to the version field is not idiomatic.

Example `.release-please-manifest.json`:
```json
{
  ".": "0.7.2"
}
```

### GitHub Actions Workflow

> **Note:** The section below describes the workflow design concepts. The production-ready implementation splits this into a reusable central workflow (hosted in the organisation-level [`openfga/.github`](https://github.com/openfga/.github) repository) and a thin per-repo caller. See [Reusable Workflow Across the Organisation](#reusable-workflow-across-the-organisation) for the full production workflows.

The release workflow has two triggers:

- **`push` to `main`** — the job only runs when the head commit message starts with `release` (the Release PR merge commit title set by Release Please). All other pushes to `main` are skipped entirely.
- **`workflow_dispatch`** — a maintainer triggers from the GitHub UI to create or update the Release PR with a specific bump type.

A `concurrency` group (`release`, non-cancellable) ensures only one release run is in flight at a time.

```yaml
name: release-please

on:
  push:
    branches:
      - main
  workflow_dispatch:
    inputs:
      bump-type:
        description: >
          Version bump type. Select 'explicit' to supply an exact version via
          the 'release-version' field below. Select 'auto' to let
          conventional-commits determine the bump automatically.
        required: false
        type: choice
        default: 'auto'
        options:
          - auto
          - patch
          - minor
          - major
          - explicit
      release-version:
        description: >
          Explicit version to release (e.g. 1.2.3 or 1.4.0-beta.1).
        required: false
        type: string

permissions:
  contents: write
  pull-requests: write

concurrency:
  group: release
  cancel-in-progress: false

jobs:
  release-please:
    runs-on: ubuntu-latest
    # On push: only run when the merge commit is a release PR commit
    # (title starts with "release"). This prevents the job firing on
    # every single push to main. The release PR merge commit always has this
    # title because release-please sets the PR title to "release: ...".
    # On workflow_dispatch: always run (manual trigger for creating releases).
    if: |
      github.event_name == 'workflow_dispatch' ||
      startsWith(github.event.head_commit.message, 'release')

    outputs:
      release_created: ${{ steps.release.outputs.release_created }}
      tag_name:        ${{ steps.release.outputs.tag_name }}
      pr_number:       ${{ steps.release.outputs.pr_number }}

    steps:
      - name: Generate token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id:      ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}

      - name: Checkout
        uses: actions/checkout@v4
        with:
          token:       ${{ steps.app-token.outputs.token }}
          fetch-depth: 0

      # ── workflow_dispatch only: rebuild release branch from latest main ──
      - name: Prepare release branch
        if: github.event_name == 'workflow_dispatch'
        run: |
          git fetch origin main
          git checkout -B release origin/main
          git push origin release --force

      # Close any stale open release PRs before creating a new one.
      # These accumulate when previous runs fail or PRs get manually closed
      # without merging. Finding two merged PRs for the same version causes
      # release-please to try creating duplicate releases and fail.
      - name: Close stale release PRs
        if: github.event_name == 'workflow_dispatch'
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
        run: |
          gh pr list \
            --repo ${{ github.repository }} \
            --head "release-please--branches--release" \
            --state open \
            --json number \
            --jq '.[].number' \
          | xargs -r -I{} gh pr close {} \
              --repo ${{ github.repository }} \
              --comment "Superseded by new release workflow run — closing stale release PR."
          echo "Stale PR cleanup done."

      - name: Compute release-as version
        id: compute-release-as
        if: github.event_name == 'workflow_dispatch'
        run: |
          BUMP="${{ inputs.bump-type }}"

          if [[ "$BUMP" == "patch" || "$BUMP" == "minor" || "$BUMP" == "major" ]]; then
            CURRENT=$(jq -r '.["."]' .release-please-manifest.json | cut -d'-' -f1)
            IFS='.' read -r MAJOR MINOR PATCH <<< "$CURRENT"

            if   [[ "$BUMP" == "major" ]]; then NEXT="$((MAJOR + 1)).0.0"
            elif [[ "$BUMP" == "minor" ]]; then NEXT="${MAJOR}.$((MINOR + 1)).0"
            else                               NEXT="${MAJOR}.${MINOR}.$((PATCH + 1))"
            fi

            echo "value=$NEXT" >> "$GITHUB_OUTPUT"
          else
            echo "value=" >> "$GITHUB_OUTPUT"
          fi

      - name: Push Release-As commit
        if: >-
          github.event_name == 'workflow_dispatch' && (
            (inputs.bump-type == 'explicit' && inputs.release-version != '') ||
            inputs.bump-type == 'patch' ||
            inputs.bump-type == 'minor' ||
            inputs.bump-type == 'major'
          )
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          git checkout release

          if [[ "${{ inputs.bump-type }}" == "explicit" ]]; then
            VERSION="${{ inputs.release-version }}"
          else
            VERSION="${{ steps.compute-release-as.outputs.value }}"
          fi

          git commit --allow-empty \
            -m "chore: release ${VERSION}" \
            -m "Release-As: ${VERSION}"

          git push origin release

      # ── key fix: target-branch depends on what triggered the run ────────
      #
      #   workflow_dispatch  →  target release branch (build the PR)
      #   push to main       →  target main branch    (detect the merged PR
      #                         and emit release_created=true)
      - name: Resolve target branch
        id: target
        run: |
          if [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
            echo "branch=release" >> "$GITHUB_OUTPUT"
          else
            echo "branch=main" >> "$GITHUB_OUTPUT"
          fi

      - name: Run release-please
        uses: googleapis/release-please-action@v4
        id: release
        with:
          token:         ${{ steps.app-token.outputs.token }}
          config-file:   release-please-config.json
          manifest-file: .release-please-manifest.json
          target-branch: ${{ steps.target.outputs.branch }}


      - name: Retarget release PR to main
        if: github.event_name == 'workflow_dispatch'
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
        run: |
          PR=$(gh pr list \
            --repo ${{ github.repository }} \
            --head "release-please--branches--release" \
            --json number \
            --jq '.[0].number' 2>/dev/null || true)

          if [[ -z "$PR" || "$PR" == "null" ]]; then
            echo "No release PR found — nothing to retarget."
            exit 0
          fi

          echo "Retargeting PR #$PR base → main"
          gh api \
            repos/${{ github.repository }}/pulls/$PR \
            -X PATCH \
            -f base=main

          # Rename head branch from release-please--branches--release to
          # release-please--branches--main so the post-merge push-to-main
          # run can correlate the merge commit with this release PR.
          echo "Renaming head branch to release-please--branches--main"
          SHA=$(gh api repos/${{ github.repository }}/git/refs/heads/release-please--branches--release --jq '.object.sha')
          gh api \
            repos/${{ github.repository }}/git/refs/heads/release-please--branches--release \
            -X PATCH \
            -f ref="refs/heads/release-please--branches--main" \
            -f sha="$SHA"

          echo "Done. PR #$PR now targets main with correct head branch."

  # ── post-release: fires when the release PR is merged into main ──────────
  post-release:
    needs: release-please
    if: needs.release-please.outputs.release_created == 'true'
    runs-on: ubuntu-latest

    steps:
      - name: Generate token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id:      ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}

      - name: Checkout
        uses: actions/checkout@v4
        with:
          token:       ${{ steps.app-token.outputs.token }}
          fetch-depth: 0

      # Reset release branch to match main so the next cycle starts clean
      - name: Sync release branch to main
        run: |
          git fetch origin main
          git checkout release 2>/dev/null || git checkout -b release
          git reset --hard origin/main
          git push origin release --force

      - name: Post-release summary
        run: |
          echo "### Release ${{ needs.release-please.outputs.tag_name }} published" >> "$GITHUB_STEP_SUMMARY"
          echo "" >> "$GITHUB_STEP_SUMMARY"
          echo "Tag \`${{ needs.release-please.outputs.tag_name }}\` is now available." >> "$GITHUB_STEP_SUMMARY"
          echo "\`release\` branch has been reset to \`main\` for the next cycle." >> "$GITHUB_STEP_SUMMARY"
          echo "Add downstream steps here (e.g. publish packages, deploy)." >> "$GITHUB_STEP_SUMMARY"
```

Key details:

- **`Generate token`** mints a short-lived GitHub App token at the start of each job. All git operations and GitHub API calls use this App identity — never a personal token or the generic `github-actions[bot]` for PR-facing work.
- **Job-level `if` guard:** On `push` events the job is skipped unless the head commit message starts with `release` — the title Release Please always gives Release PR merge commits. On `workflow_dispatch` the job always runs.
- **`Prepare release branch`** (dispatch only) force-resets the `release` staging branch to the tip of `main`, giving Release Please a clean base to work from each cycle.
- **`Close stale release PRs`** (dispatch only) closes any open PRs whose head is `release-please--branches--release` before creating a new one, preventing duplicate-release failures.
- **`Compute release-as version`** (dispatch only) reads the current version from `.release-please-manifest.json` and calculates the next semver for `patch`, `minor`, or `major` bumps. For `explicit`, the user-supplied version is used directly.
- **`Push Release-As commit`** (dispatch only, non-`auto`) pushes an empty commit with the `Release-As: X.Y.Z` trailer onto the `release` branch — **never onto `main`**. This is the core of the branch-protection workaround.
- **`Resolve target branch`** sets `target-branch` to `release` on dispatch (so Release Please builds the PR against the staging branch) and to `main` on push (so it detects the merged PR and emits `release_created=true`).
- **`Run release-please`** runs on every invocation. The `target-branch` output from the previous step controls whether it is creating/updating a PR or finalizing a release.
- **`Retarget release PR to main`** (dispatch only) retargets the PR base to `main` and renames the head branch from `release-please--branches--release` to `release-please--branches--main` via GitHub API. This ensures the post-merge push event can correlate the merge commit with the release PR by head-branch name.
- **`Sync release branch to main`** (`post-release` job) resets the `release` branch back to `main` after a successful release so the next dispatch cycle starts from a clean state.
- **`post-release`** runs on the Release PR merge (`push` to `main`). In the production workflow it mints its own App token, imports the bot GPG key, creates the signed `v*` tag and a draft GitHub Release, and — via the tag push — hands off to the per-repo publish workflow that undrafts the release on success. See [Signed Tags, Draft Releases, and Undraft-on-Publish](#how-it-works).

### Identity and Signing

Instead of using a Personal Access Token (PAT) tied to an individual maintainer, the workflow uses a **dedicated GitHub App** as its bot identity. This has several advantages:

- **No long-lived secrets.** The App's static credentials (`RELEASER_APP_CLIENT_ID` + `RELEASER_APP_PRIVATE_KEY`) are exchanged for a short-lived token at the start of each workflow run via [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token). The token expires after the run completes, so the blast radius of any credential compromise is limited to a single run.
- **Least privilege.** The App is granted only `contents: write` and `pull-requests: write` — the minimum permissions needed to push the `Release-As` commit, manage the Release PR, and create tags.
- **GPG-signed commits and tags.** The `Release-As` commit pushed to the `release` staging branch is GPG-signed using a dedicated bot GPG key, imported via the `crazy-max/ghaction-import-gpg` action. Git is configured with `commit.gpgSign = true` and `tag.gpgSign = true` so every commit and tag produced by the bot carries a verifiable signature traceable to the project's published GPG key. The release tag itself is created by the bot with `git tag -s` in the `post-release` job — release-please's own tag/release creation is disabled via `skip-github-release: true` — so the `v*` tag carries the bot's GPG signature rather than GitHub's API-generated signature.
- **Not tied to a person.** Unlike a PAT, the App identity is owned by the account or organization, not an individual. There is no risk of losing access when a maintainer rotates out.

**Setup steps:**

1. Create a GitHub App under **Settings → Developer settings → GitHub Apps** (or under the org's settings for org-wide use).
2. Disable webhooks (not needed).
3. Grant **Contents: Read & write** and **Pull requests: Read & write** permissions.
4. Generate a private key (`.pem` file).
5. Install the App on the target repository (or repositories).
6. Store `RELEASER_APP_CLIENT_ID` and `RELEASER_APP_PRIVATE_KEY` as repository (or organization) secrets.
7. Generate a dedicated GPG key for the releaser bot identity. Store the armored private key as `GPG_PRIVATE_KEY` and the passphrase as `GPG_PASSPHRASE`. Publish the corresponding public key so downstream users can verify signed artifacts.

For organization migration, the same App and GPG key can be configured across multiple repositories from a single place, making it easy to manage centrally.

### End-to-End Flow

**On push to `main` (Release PR merge):**

When a Release PR is merged, the merge commit message starts with `release:`. The caller's `if` guard matches this pattern, so the reusable workflow runs with `target-branch: main` (landing the changelog and manifest bump), and the `post-release` job then creates the GPG-signed `v*` tag and a **draft** GitHub Release. For all other merges to `main`, the workflow is skipped entirely.

**On `workflow_dispatch` (manual trigger):**

1. **Trigger:** A maintainer navigates to **Actions → release-please → Run workflow** in the GitHub UI and selects the desired bump type (`auto`, `patch`, `minor`, `major`, or `explicit`).

2. **Prepare staging branch:** The workflow force-resets the `release` branch to the tip of `main` and closes any stale release PRs from previous cycles.

3. **Version override (if applicable):** For non-`auto` bump types, the workflow computes the target version and pushes an empty `Release-As: X.Y.Z` commit onto the `release` branch — never directly onto `main`.

4. **Release PR created:** Release Please runs with `target-branch: release`, analyzes commits since the last release, generates the changelog, updates all version markers, and opens the Release PR.

5. **Retarget:** The workflow immediately retargets the PR base to `main` and renames the head branch to `release-please--branches--main` via GitHub API, so the post-merge push event can correctly identify it.

6. **Review:** The team reviews the Release PR diff — changelog accuracy, version correctness, and any additional notes.

7. **Merge:** A maintainer merges the Release PR into `main`. This triggers the `push` path above, which finalizes the release.

8. **Post-release:** The `post-release` job creates the GPG-signed `v*` tag and a draft GitHub Release (notes built from the `CHANGELOG.md` section plus GitHub's auto-generated notes), then resets the `release` branch back to `main` for the next cycle.

9. **Publish & undraft:** Pushing the signed tag triggers the existing tag-triggered workflow, which runs the test suite and publishes to the appropriate registry. On success it calls `undraft-release.yml` to flip the draft release to published.

### Consistency across repositories

The reusable release workflows, the Conventional Commits PR-title check, and the shared release-notes scripts are centralised in the organisation-level **[`openfga/.github`](https://github.com/openfga/.github)** repository and called by every SDK/CLI repo. The per-repo Release Please configuration (`release-please-config.json`, `.release-please-manifest.json`, and the `x-release-please-version` markers) continues to be generated and kept in sync via the [sdk-generator](https://github.com/openfga/sdk-generator). This split means any improvement to the release *logic* propagates to all repos on the next run (no per-repo PR), while the language-specific *configuration* stays owned by each repo.

### Reusable Workflow Across the Organisation

Rather than duplicating the full `release-please.yml` into every repository, the workflow logic is centralised once in **`openfga/.github`** and called from each repo using GitHub's [`workflow_call`](https://docs.github.com/en/actions/using-workflows/reusing-workflows) trigger. GitHub only resolves reusable workflows from a repository's `.github/workflows/` directory, so the org repo named `.github` hosts them there and callers reference them as `openfga/.github/.github/workflows/<file>@<ref>` — the doubled `.github` is the repository name followed by the directory. The config files (`release-please-config.json`, `.release-please-manifest.json`, version markers) remain per-repo; the workflow logic lives in one place.

The shared repo hosts:

- `.github/workflows/release-please.yml` — release preparation plus signed-tag / draft-release creation.
- `.github/workflows/undraft-release.yml` — flips a draft release to published; called by each repo's publish workflow after a successful registry publish.
- `.github/workflows/pr-title-check.yml` — the Conventional Commits PR-title validation.
- `.github/workflows/scripts/parse-release.sh` — the manifest-diff / changelog-notes helper, unit-tested by `.github/workflows/test-release-scripts.yml`.

**Central repository (`openfga/.github`)** — the reusable workflow (`.github/workflows/release-please.yml`):

```yaml
name: release-please

on:
  workflow_call:
    inputs:
      trigger-event:
        description: >
          The GitHub event that triggered the caller workflow (workflow_dispatch
          or push). Pass github.event_name from the caller. Used internally to
          distinguish manual dispatch from automated push-to-main runs.
        required: true
        type: string
      bump-type:
        description: >
          Version bump type. Select 'explicit' to supply an exact version via
          the 'release-version' field below. Select 'auto' to let
          conventional-commits determine the bump automatically.
        required: false
        type: string
        default: 'auto'
      release-version:
        description: >
          Explicit version to release (e.g. 1.2.3 or 1.4.0-beta.1).
        required: false
        type: string
        default: ''
    secrets:
      RELEASER_APP_CLIENT_ID:
        required: true
      RELEASER_APP_PRIVATE_KEY:
        required: true
      GPG_PRIVATE_KEY:
        required: true
      GPG_PASSPHRASE:
        required: false

permissions:
  contents: read

concurrency:
  group: release
  cancel-in-progress: false

jobs:
  release-please:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    if: inputs.trigger-event == 'workflow_dispatch' || inputs.trigger-event == 'push'

    outputs:
      release_created: ${{ steps.release.outputs.release_created }}
      tag_name:        ${{ steps.release.outputs.tag_name }}
      pr_number:       ${{ steps.release.outputs.pr_number }}

    steps:
      - name: Generate token
        id: app-token
        uses: actions/create-github-app-token@1b10c78c7865c340bc4f6099eb2f838309f1e8c3 # v3.1.1
        with:
          client-id:   ${{ secrets.RELEASER_APP_CLIENT_ID }}
          private-key: ${{ secrets.RELEASER_APP_PRIVATE_KEY }}

      - name: Checkout
        uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
        with:
          token:       ${{ steps.app-token.outputs.token }}
          fetch-depth: 0

      - name: Prepare release branch
        if: inputs.trigger-event == 'workflow_dispatch'
        run: |
          git fetch origin main
          git checkout -B release origin/main
          git push origin release --force

      - name: Close stale release PRs
        if: inputs.trigger-event == 'workflow_dispatch'
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
        run: |
          gh pr list \
            --repo ${{ github.repository }} \
            --head "release-please--branches--release" \
            --state open \
            --json number \
            --jq '.[].number' \
          | xargs -r -I{} gh pr close {} \
              --repo ${{ github.repository }} \
              --comment "Superseded by new release workflow run — closing stale release PR."

          # Remove 'autorelease: pending' from merged PRs that were NOT merged
          # into main to prevent duplicate-release attempts.
          gh pr list \
            --repo ${{ github.repository }} \
            --label "autorelease: pending" \
            --state merged \
            --json number,baseRefName \
            --jq '.[] | select(.baseRefName != "main") | .number' \
          | xargs -r -I{} gh pr edit {} \
              --repo ${{ github.repository }} \
              --remove-label "autorelease: pending"

      - name: Compute release-as version
        id: compute-release-as
        if: inputs.trigger-event == 'workflow_dispatch'
        run: |
          BUMP="${{ inputs.bump-type }}"
          if [[ "$BUMP" == "patch" || "$BUMP" == "minor" || "$BUMP" == "major" ]]; then
            CURRENT=$(jq -r '.["."]' .release-please-manifest.json | cut -d'-' -f1)
            IFS='.' read -r MAJOR MINOR PATCH <<< "$CURRENT"
            if   [[ "$BUMP" == "major" ]]; then NEXT="$((MAJOR + 1)).0.0"
            elif [[ "$BUMP" == "minor" ]]; then NEXT="${MAJOR}.$((MINOR + 1)).0"
            else                               NEXT="${MAJOR}.${MINOR}.$((PATCH + 1))"
            fi
            echo "value=$NEXT" >> "$GITHUB_OUTPUT"
          else
            echo "value=" >> "$GITHUB_OUTPUT"
          fi

      - name: Import GPG key
        id: import-gpg
        uses: crazy-max/ghaction-import-gpg@2dc316deee8e90f13e1a351ab510b4d5bc0c82cd # v7.0.0
        with:
          gpg_private_key:     ${{ secrets.GPG_PRIVATE_KEY }}
          passphrase:          ${{ secrets.GPG_PASSPHRASE }}
          git_user_signingkey: true
          git_commit_gpgsign:  true

      - name: Configure Git
        run: |
          git config user.name  openfga-releaser-bot
          git config user.email ${{ steps.import-gpg.outputs.email }}
          git config tag.gpgSign true

      - name: Push Release-As commit
        if: >-
          inputs.trigger-event == 'workflow_dispatch' && (
            (inputs.bump-type == 'explicit' && inputs.release-version != '') ||
            inputs.bump-type == 'patch' ||
            inputs.bump-type == 'minor' ||
            inputs.bump-type == 'major'
          )
        run: |
          git checkout release
          if [[ "${{ inputs.bump-type }}" == "explicit" ]]; then
            VERSION="${{ inputs.release-version }}"
          else
            VERSION="${{ steps.compute-release-as.outputs.value }}"
          fi
          git commit --allow-empty \
            -m "chore: release ${VERSION}" \
            -m "Release-As: ${VERSION}"
          git push origin release

      - name: Resolve target branch
        id: target
        run: |
          if [[ "${{ inputs.trigger-event }}" == "workflow_dispatch" ]]; then
            echo "branch=release" >> "$GITHUB_OUTPUT"
          else
            echo "branch=main" >> "$GITHUB_OUTPUT"
          fi

      - name: Run release-please
        uses: googleapis/release-please-action@45996ed1f6d02564a971a2fa1b5860e934307cf7 # v5.0.0
        id: release
        with:
          token:         ${{ steps.app-token.outputs.token }}
          config-file:   release-please-config.json
          manifest-file: .release-please-manifest.json
          target-branch: ${{ steps.target.outputs.branch }}

      - name: Retarget release PR to main
        if: inputs.trigger-event == 'workflow_dispatch'
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
        run: |
          PR=$(gh pr list \
            --repo ${{ github.repository }} \
            --head "release-please--branches--release" \
            --json number \
            --jq '.[0].number' 2>/dev/null || true)
          if [[ -z "$PR" || "$PR" == "null" ]]; then
            echo "No release PR found — nothing to retarget."
            exit 0
          fi
          gh api repos/${{ github.repository }}/pulls/$PR -X PATCH -f base=main
          SHA=$(gh api repos/${{ github.repository }}/git/refs/heads/release-please--branches--release --jq '.object.sha')
          gh api \
            repos/${{ github.repository }}/git/refs/heads/release-please--branches--release \
            -X PATCH \
            -f ref="refs/heads/release-please--branches--main" \
            -f sha="$SHA"

  # post-release: fires on the Release PR merge (push to main). With
  # skip-github-release enabled, release-please only lands the changelog/manifest
  # bump — this job creates the GPG-signed tag and a *draft* GitHub Release.
  post-release:
    needs: release-please
    if: inputs.trigger-event == 'push'
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Generate token
        id: app-token
        uses: actions/create-github-app-token@1b10c78c7865c340bc4f6099eb2f838309f1e8c3 # v3.1.1
        with:
          client-id:   ${{ secrets.RELEASER_APP_CLIENT_ID }}
          private-key: ${{ secrets.RELEASER_APP_PRIVATE_KEY }}

      - name: Checkout
        uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
        with:
          token:       ${{ steps.app-token.outputs.token }}
          fetch-depth: 0

      - name: Import GPG key
        id: import-gpg
        uses: crazy-max/ghaction-import-gpg@2dc316deee8e90f13e1a351ab510b4d5bc0c82cd # v7.0.0
        with:
          gpg_private_key: ${{ secrets.GPG_PRIVATE_KEY }}
          passphrase:      ${{ secrets.GPG_PASSPHRASE }}
          git_user_signingkey: true
          git_commit_gpgsign:  true

      - name: Configure Git
        run: |
          git config user.name openfga-releaser-bot
          git config user.email ${{ steps.import-gpg.outputs.email }}
          git config tag.gpgSign true

      # parse-release.sh lives in openfga/.github; actions/checkout above pulled
      # the *caller* repo, so check out the shared repo for the helper script.
      - name: Checkout shared release scripts
        uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
        with:
          repository:  openfga/.github
          ref:         main
          path:        .shared-ci

      - name: Detect released version from manifest diff
        id: parse-release
        run: |
          git show "HEAD~1:.release-please-manifest.json" > /tmp/prev.json
          RELEASES=$(bash .shared-ci/.github/workflows/scripts/parse-release.sh \
            manifest-diff .release-please-manifest.json /tmp/prev.json)
          echo "releases=$RELEASES" >> "$GITHUB_OUTPUT"

      - name: Create signed tag and draft GitHub release
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
          REPO:     ${{ github.repository }}
        run: |
          echo '${{ steps.parse-release.outputs.releases }}' | jq -c '.[]' | while read -r ENTRY; do
            VERSION=$(jq -r '.version'  <<<"$ENTRY")
            TAG=$(jq -r '.tag_name'     <<<"$ENTRY")

            # GPG-signed tag. Pushing it with the App token (not GITHUB_TOKEN)
            # is what triggers the publish workflow.
            git tag -s "$TAG" -m "Release ${TAG}"
            git push origin "$TAG"

            # Release notes = CHANGELOG section (parse-release.sh) + GitHub's
            # auto-generated notes for the same range.
            NOTES=$(bash .shared-ci/.github/workflows/scripts/parse-release.sh \
              changelog-notes CHANGELOG.md "$VERSION")

            # Draft release — undrafted later by the publish workflow on success.
            gh release create "$TAG" --repo "$REPO" --title "$TAG" --notes "$NOTES" --draft
          done

      # Reset the release branch back to main so the next cycle starts clean.
      - name: Sync release branch to main
        run: |
          git fetch origin main
          git checkout release 2>/dev/null || git checkout -b release
          git reset --hard origin/main
          git push origin release --force
```

**Each SDK repository** — a thin caller (Python SDK shown as an example):

```yaml
name: release-please

permissions:
  contents: read

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      bump-type:
        description: >
          Version bump type. Select 'explicit' to supply an exact version via
          the 'release-version' field below. Select 'auto' to let
          conventional-commits determine the bump automatically.
        required: false
        type: choice
        default: 'auto'
        options:
          - auto
          - patch
          - minor
          - major
          - explicit
      release-version:
        description: >
          Explicit version to release (e.g. 1.2.3 or 1.4.0-beta.1).
        required: false
        type: string

jobs:
  release:
    permissions:
      contents: write
      pull-requests: write
    if: |
      github.event_name == 'workflow_dispatch' ||
      startsWith(github.event.head_commit.message, 'release:')
    uses: openfga/.github/.github/workflows/release-please.yml@main
    with:
      trigger-event:   ${{ github.event_name }}
      bump-type:       ${{ inputs.bump-type || 'auto' }}
      release-version: ${{ inputs.release-version || '' }}
    secrets:
      RELEASER_APP_CLIENT_ID:   ${{ secrets.RELEASER_APP_CLIENT_ID }}
      RELEASER_APP_PRIVATE_KEY: ${{ secrets.RELEASER_APP_PRIVATE_KEY }}
      GPG_PRIVATE_KEY:          ${{ secrets.GPG_PRIVATE_KEY }}
      GPG_PASSPHRASE:           ${{ secrets.GPG_PASSPHRASE }}
```

Key points:

- The central workflow **must** have `workflow_call` as a trigger — this is what makes it callable from other repositories.
- The caller passes `github.event_name` as `trigger-event` so the reusable workflow can distinguish a manual dispatch from a push-to-main run. This is necessary because `github.event_name` is not available in a `workflow_call` context.
- Reference the central workflow as `org/repo/.github/workflows/file.yml@ref`. In production, callers should pin to a commit SHA or tag (e.g. `@v1`) rather than `@main` for stability.
- **Secrets must be explicitly forwarded** via the `secrets:` block — they are not inherited automatically across repositories.
- The `release` staging branch, `release-please-config.json`, `.release-please-manifest.json`, and all `x-release-please-version` markers still live in **each individual SDK repository** — only the workflow logic is centralised.
- Any update to the shared workflow (e.g. a new step, a security fix, a new bump-type option) propagates to all callers on the next run, without a PR to every SDK repo.

### Signed Tags, Draft Releases, and Undraft-on-Publish

The release finalisation is intentionally **not** delegated to release-please's own tag/release creation. Each repo's `release-please-config.json` sets `"skip-github-release": true`, so release-please only lands the changelog and manifest bump in the Release PR. The signed tag and the GitHub Release are produced by the `post-release` job, which gives the project full ownership of every release artifact and a clean publish handshake:

1. **Detect what changed.** On the Release PR merge (`push` to `main`), `post-release` diffs `.release-please-manifest.json` against its previous revision using the shared `parse-release.sh manifest-diff` helper. For a single-package repo the manifest is `{ ".": "X.Y.Z" }`, so the helper yields one release with tag `vX.Y.Z`.
2. **Create the GPG-signed tag.** The bot's GPG key is imported and the tag is created with `git tag -s` and pushed. Because the push is authenticated with the **GitHub App installation token** (not the default `GITHUB_TOKEN`, whose pushes are deliberately prevented from triggering further workflows), the tag push **does** start the existing publish workflow.
3. **Build the release notes.** Notes are assembled from the matching `CHANGELOG.md` section (`parse-release.sh changelog-notes`) and GitHub's auto-generated "what's changed" notes for the same commit range.
4. **Create a *draft* GitHub Release.** The release is created as a draft against the signed tag (marked pre-release when the version carries a suffix such as `-beta.1`).
5. **Publish, then undraft.** The tag push triggers the per-repo publish workflow (test → publish to the language registry). Only after those jobs succeed does a final job call the shared `undraft-release.yml`, which flips the release to published (preserving the pre-release flag). If publishing fails, the release stays a draft — nothing half-published is ever surfaced as a real release.

The publish workflow undraft step is a thin caller:

```yaml
  undraft-release:
    needs: [publish]            # whatever the repo's publish jobs are
    if: startsWith(github.ref, 'refs/tags/v')
    permissions:
      contents: write           # job-level grant; top-level default stays read
    uses: openfga/.github/.github/workflows/undraft-release.yml@main
```

**Where the logic lives.** The `parse-release.sh` helper (and its unit tests, run by `test-release-scripts.yml` on every change) lives in `openfga/.github` at `.github/workflows/scripts/`. Since a reusable workflow's `actions/checkout` checks out the *caller* repo, `post-release` additionally checks out `openfga/.github` into a sub-path so the helper is available at runtime.

> **Why a draft first?** Creating the release as a draft and only undrafting after a successful publish makes the GitHub Release the source of truth for "this version is actually out". It also closes the small window where a release could be visible while its package is still missing from the registry.

## Security
[security]: #security

This section documents the security controls that MUST be enforced for the release pipeline.

### GPG-Signed Bot Commits

All commits pushed by the releaser bot MUST be GPG-signed using the project's dedicated bot GPG key. In the reusable workflow, the key is imported via the `crazy-max/ghaction-import-gpg` action and git is configured with `commit.gpgSign = true` and `tag.gpgSign = true` before any commits are created:

```yaml
- name: Import GPG key
  uses: crazy-max/ghaction-import-gpg@...
  with:
    gpg_private_key:     ${{ secrets.GPG_PRIVATE_KEY }}
    passphrase:          ${{ secrets.GPG_PASSPHRASE }}
    git_user_signingkey: true
    git_commit_gpgsign:  true

- name: Configure Git
  run: |
    git config user.name  openfga-releaser-bot
    git config user.email ${{ steps.import-gpg.outputs.email }}
    git config tag.gpgSign true
```

The GPG private key and passphrase are stored as repository/organization secrets (`GPG_PRIVATE_KEY`, `GPG_PASSPHRASE`) scoped to the releaser bot identity — not tied to any individual maintainer. The corresponding GPG public key must be published (e.g. in a project `SECURITY.md` or on a public keyserver) so that downstream users and auditors can verify signed artifacts.

> Release tags are created by the releaser bot with `git tag -s` in the `post-release` job (`skip-github-release: true` disables release-please's own tag/release creation), so each `v*` tag carries the bot's GPG signature traceable to the project's published key — no reliance on GitHub's API-generated signature. See [Signed Tags, Draft Releases, and Undraft-on-Publish](#how-it-works).

### Tag Push Restrictions

Maintainers MUST NOT be able to push tags directly. Tag protection rules must ensure that only the releaser GitHub App (bot) may create tags matching the `v*` pattern. No human actor — including repository admins — should be able to push a `v*` tag outside of the release workflow.

**Configuration:**

Under **Settings → Rules → Rulesets** (or **Settings → Tags → Protected tags**), create a tag ruleset matching `v*` that:

- Restricts tag creation to the releaser GitHub App identity only.
- Prevents deletion of `v*` tags by any actor.

This ensures every published release tag is traceable to a workflow run and cannot be created or overwritten by an individual maintainer acting outside the automation.

### Branch Protection for `main`

Both the releaser bot and human maintainers MUST NOT be able to push directly to `main` without going through a pull request with a required CODEOWNER review.

The following branch protection rules MUST be enabled on `main`:

| Rule | Setting |
|---|---|
| Require a pull request before merging | ✅ Enabled |
| Required approvals | ≥ 1 |
| Require review from Code Owners | ✅ Enabled |
| Restrict who can push to matching branches | ✅ Enabled (no direct push for any actor) |
| Do not allow bypassing the above settings | ✅ Enabled (applies to admins and bots) |

The releaser bot satisfies this constraint by design: it never pushes directly to `main`. The `Release-As` commit lands on the `release` staging branch, and all changes reach `main` exclusively through the reviewed Release PR. See [Workaround Flow](#workaround-flow-explicit-version-overrides-without-bot-write-access-to-main) for the full mechanics.

A `CODEOWNERS` file must be present in each repository. At minimum it should designate the core SDK maintainer team as owners of all files (e.g. `* @openfga/sdk-maintainers`), ensuring that the required CODEOWNER review is enforced on all PRs — including the Release PR.

---

## Failure Modes and Rollback Procedures
[failure-modes-and-rollback-procedures]: #failure-modes-and-rollback-procedures

Release Please splits the release into two clearly separated phases — Release PR merge (version bookkeeping) and tag creation / publish (artifact delivery). Because the phases are decoupled, each failure mode has a targeted recovery path rather than a blanket "revert everything" approach.

### Release PR merged but tag creation fails

**What happened:** The Release PR was merged and `main` now carries the correct version bump and changelog entry, but the `post-release` job (or Release Please's own tag-creation step) failed before the git tag was pushed.

**Recovery:**
1. Identify the merge commit SHA for the Release PR: `git log --oneline -5`.
2. Manually push a signed tag pointing to that commit:
   ```bash
   git tag -s v1.x.y <sha>
   git push origin v1.x.y
   ```
3. A manually pushed `v*` tag **always** triggers the publish workflow — no need to revert or re-run the Release PR.

> **Note:** Do not revert the Release PR merge. The changelog and version bump are correct; only the tag is missing.

---

### Tag created but a publish step fails

**What happened:** The git tag exists and the GitHub Release may or may not have been created, but one or more publish targets (a package registry, GitHub Releases, etc.) reported a failure.

**Guiding principle:** Treat each publish target independently. A blanket retry of all targets risks partial duplicates (e.g., pushing to a registry twice).

| Scenario | Recovery |
|---|---|
| Only **GitHub Releases** failed | Create it manually: `gh release create v1.x.y --title "v1.x.y" --notes-file RELEASE_NOTES.md`. No registry re-push needed. |
| Only a **registry push** failed (npm, PyPI, Maven, NuGet, pkg.go.dev) | Re-run only the failing publish job or step in the Actions UI. Do not re-push to targets that already succeeded. |
| **Multiple targets** failed | Inspect the logs before acting. Identify exactly which targets received the artifact and which did not, then fix only the failed ones. |
| Tag itself is **wrong** (wrong commit, wrong name) | Requires elevated access to delete a protected tag — there is an established process for this (previously followed by the team). Once deleted, push the corrected tag. Only pursue this path if the tag points to the wrong commit; a publish failure alone does not require deleting the tag. |

---

### Release created with wrong version

**What happened:** A release was tagged and published (or partially published) with an incorrect version number (e.g., a minor bump when a patch was intended).

The recovery path depends on whether the artifact has already reached a registry.

**Artifact already in a registry:**
Registries are generally immutable — a published artifact cannot be deleted or overwritten. The standard recovery is to **bump forward** using the `explicit` override on the next `workflow_dispatch` run:

1. Trigger the workflow with `bump-type: explicit` and set `release-version` to the correct next version.
2. Release Please creates a new Release PR targeting the correct version.
3. Optionally annotate the wrong version in `CHANGELOG.md` as a known bad release so downstream users are aware.

**Artifact not yet in any registry (caught immediately):**
If the wrong-version GitHub Release was created but nothing reached a registry:

1. Delete the GitHub Release via the UI or `gh release delete v1.x.y`.
2. Delete the tag (requires elevated access): `git push --delete origin v1.x.y`.
3. Reset the version in `.release-please-manifest.json` to the previous correct version and push the change to `main`.
4. Re-trigger the workflow with the correct bump type.

---

### Quick-reference summary

| Failure | Fix |
|---|---|
| Tag missing after PR merge | `git tag -s vX.Y.Z <sha> && git push origin vX.Y.Z` |
| Partial publish failure | Fix only the failed target; don't redo successful ones |
| Wrong version — already in registries | Bump forward with `explicit` override; can't go back |
| Wrong version — not yet in registries | Delete release + tag, reset manifest, re-trigger |

---

### Workaround Flow: Explicit Version Overrides Without Bot Write Access to `main`

The standard Release Please override mechanism pushes an empty `Release-As` commit directly to `main`. This conflicts with branch protection rules that require all changes to go through a PR. The diagram below illustrates the workaround: instead of pushing directly to `main`, the workflow pushes the `Release-As` commit onto a **dedicated release branch**, Release Please opens a PR from that branch targeting `main`, and then the workflow retargets the PR base and renames the head branch so that Release Please can find the merged PR by head-branch name on the subsequent push event. The bot never needs write access to `main`.

```
  workflow_dispatch (bump-type: patch | minor | major | explicit)
           │
           ▼
  ┌─────────────────────────────────────────┐
  │  Reset release branch to main           │  (Actions bot — release branch only,
  │  Push Release-As: vX.Y.Z commit         │   never touches main directly)
  └─────────────────────────────────────────┘
           │
           ▼
  ┌─────────────────────────────────────────┐
  │  release-please creates Release PR      │  (releaser GitHub App token)
  │  using the Release-As commit as seed    │
  └─────────────────────────────────────────┘
           │
           ▼
  ┌─────────────────────────────────────────┐
  │  Retarget PR base → main                │
  │  Rename head branch to canonical name   │  (so release-please can find it later
  │  (release-please--branches--main)       │   by head-branch name on push event)
  └─────────────────────────────────────────┘
           │
           ▼
  ┌─────────────────────────────────────────┐
  │        Human reviews + merges           │  ← normal PR review, no bypass needed
  └─────────────────────────────────────────┘
           │
           ├─── merge commit starts with "chore: release" ──────────────────────┐
           │                                                                     │
           │  push to main trigger fires                                         │  any other merge
           ▼                                                                     ▼
  ┌──────────────────────────────────────┐                             ┌─────────────────────┐
  │  release-please (target: main)       │                             │   workflow skipped   │
  │  Finds merged PR by head-branch name │                             └─────────────────────┘
  └──────────────────────────────────────┘
           │
           ▼
  ┌──────────────────────────────────────┐
  │   Tag + GitHub Release created       │
  └──────────────────────────────────────┘
           │
           ▼
  ┌──────────────────────────────────────┐
  │  post-release: sync release → main   │
  └──────────────────────────────────────┘
```

**Why this works without granting the bot write access to `main`:**

- The `Release-As` commit lands on the *release branch*, not on `main`. Branch protection for `main` is never touched.
- The GitHub App token (not `github-actions[bot]`) opens the PR, so it is attributable to the App identity and picks up the App's bypass rules (if any) — not the generic bot's.
- Retargeting the PR base to `main` and renaming the head branch are pure GitHub API calls. No code is pushed to `main` at this point.
- On merge, the push event fires normally. Release Please locates the merged PR by its head-branch name (the canonical `release-please--branches--main` pattern) and finalises the release.

**Key constraint:** the head branch must be renamed to the exact pattern Release Please expects (`release-please--branches--main`) *before* the PR is merged, otherwise the post-merge push event cannot correlate the merge commit with the release.

---

Keeping the human in the loop on version decisions via `explicit` is the **right call pre-1.0.0**. Before a project reaches a stable public API, the semantics of "major", "minor", and "patch" are still being calibrated. Letting a maintainer explicitly confirm the next version on each release — rather than delegating that decision entirely to commit-message parsing — prevents accidental signals (e.g., a `feat:` commit unintentionally triggering a minor bump at a sensitive milestone) and preserves intentionality around version choices. Once the SDK APIs stabilize post-1.0.0, `auto` becomes the sensible default.

> **Note — release-please only understands `auto` or an explicit version string.** The `patch`, `minor`, and `major` options in the workflow dropdown are conveniences implemented on top of release-please. The workflow reads the current manifest version, computes the next semver arithmetically (e.g. `1.3.11` + patch → `1.3.12`), and passes that computed string as an explicit `Release-As:` commit — functionally identical to choosing `explicit` and typing the version yourself. There is no native patch/minor/major mode in release-please. This is why `explicit` is always the safest option when in doubt: you are simply skipping the arithmetic step.

## Migration
[migration]: #migration

### Changelog Format Migration

The OpenFGA SDKs currently follow a variant of Keep a Changelog with `[Unreleased]` sections and `Added` / `Fixed` / `Changed` groupings. While the structure is broadly consistent, the formatting is not strict enough to be reliably parsed by automated tooling.

A **one-time manual migration** of every SDK repository's `CHANGELOG.md` to the default Release Please changelog format is required. This migration must be done before enabling Release Please on each repository so that the tool can correctly parse existing entries and append new ones. The steps are:

1. Remove the `[Unreleased]` section header.
2. Normalize existing version headings to the `## X.Y.Z (YYYY-MM-DD)` format.
3. Run a changelog linter to verify the migrated file parses cleanly.
4. Add the `.release-please-manifest.json` with the current version.
5. Add the `release-please-config.json` with the appropriate configuration.

Once migrated, all future changelog entries are auto-generated by Release Please from Conventional Commits, ensuring a consistent format going forward across every SDK.

### Version Marker Annotation

Each repository must add the `x-release-please-version` marker comment to all files that contain a version constant. This is a one-time, non-breaking change. The specific files vary per repository — refer to the initial release PRs for the exact set of annotated files:

- Go SDK: [openfga/go-sdk#278](https://github.com/openfga/go-sdk/pull/278)
- Java SDK: [openfga/java-sdk#295](https://github.com/openfga/java-sdk/pull/295)
- JavaScript SDK: [openfga/js-sdk#342](https://github.com/openfga/js-sdk/pull/342)
- Python SDK: [openfga/python-sdk#247](https://github.com/openfga/python-sdk/pull/247)
- .NET SDK: [openfga/dotnet-sdk#175](https://github.com/openfga/dotnet-sdk/pull/175)
- CLI: [openfga/cli#635](https://github.com/openfga/cli/pull/635)

The same approach will be applied to the VS Code extension, IntelliJ extension, OpenFGA language repository, and any other repositories that follow a versioned release process.

> **Note on multi-package (mono-repo) repositories — e.g. [openfga/language](https://github.com/openfga/language):** Some repositories ship multiple packages (one per language) from a single repo, each released independently with its own tag prefix (`pkg/js/v0.2.1`, `pkg/go/v0.2.0`, …). Release Please handles this with a multi-package `release-please-config.json`, and the same reusable workflow and scripts serve it unchanged. The relevant config keys are:
>
> - `"separate-pull-requests": true` — one Release PR per package that has unreleased changes, instead of a single combined PR.
> - `"include-component-in-tag": true` with `"tag-separator": "/"` — tags are prefixed with the component path (`pkg/go/v0.2.0`).
> - One entry per package under `packages`, each with its own `component`, `package-name`, `changelog-path`, and `x-release-please-version` markers. The manifest then carries one version per package, e.g. `{ "pkg/go": "0.2.0", "pkg/js": "0.2.1" }`.
> - `"skip-github-release": true` — the same signed-tag / draft-release / undraft flow as the single-package repos.
>
> The shared `parse-release.sh manifest-diff` helper is component-aware: it diffs the per-package manifest entries and emits one release per *changed* package, mapping the root component `.` to a plain `vX.Y.Z` tag and any other component to `<component>/vX.Y.Z`. The `post-release` job loops over those entries, signing and drafting each independently and resolving each package's `CHANGELOG.md` and previous-tag glob from its component. The `workflow_dispatch` inputs additionally accept a `target-package` selector so a maintainer can release one package (or `all`), and the per-component Release PRs are retargeted from the `release` staging branch to `main` exactly as in the single-package flow.

### GPG Key Migration

With this flow, the `Release-As` commit is signed by a **dedicated bot GPG key** rather than an individual maintainer's personal key. As part of the migration:

1. Generate a new GPG key for the releaser bot identity (e.g. `openfga-releaser-bot <releaser@openfga.dev>`).
2. Store the armored private key and passphrase as repository/organization secrets (`GPG_PRIVATE_KEY`, `GPG_PASSPHRASE`).
3. Publish the bot's GPG public key (e.g. in a `SECURITY.md` or on a public keyserver) so downstream users can verify signed artifacts.
4. Remove any personal maintainer GPG keys that were previously stored as CI secrets for signing release tags. Maintainers no longer sign release artifacts individually — the bot handles all signing.

With `skip-github-release: true`, the git tag and GitHub Release are created by the releaser bot in the `post-release` job — the tag via `git tag -s` (carrying the bot's GPG signature) and the release via `gh release create`. Release-please no longer creates these via the GitHub API. See [Signed Tags, Draft Releases, and Undraft-on-Publish](#how-it-works).

### Conventional Commits Enforcement

The PR title validation workflow described in [Conventional Commits Validation](#conventional-commits-validation) will be added to all SDK repositories and configured as a **required status check** in each repository's branch protection rules. Existing commit history does not need to be rewritten — the validator applies only to new PRs going forward.

### Rollout Plan

1. **Pilot:** Deploy the workflow to a single SDK repository (e.g., the Go SDK) to validate the end-to-end flow.
2. **Iterate:** Incorporate feedback, adjust configuration, and refine the changelog format.
3. **Expand:** Roll out to all remaining SDK/CLI/extension repositories.

## Drawbacks
[drawbacks]: #drawbacks

- **Changelog format change.** Moving from Keep a Changelog to the default Release Please changelog format is a visible change for contributors and users who follow the changelog. However, the formats are structurally similar, and the default format has the benefit of automatic generation with configurable section grouping via `changelog-sections`.

- **Dependency on Release Please.** The release process becomes dependent on Google's Release Please tool. If the tool is deprecated or introduces breaking changes, we would need to adapt. This risk is mitigated by the tool's active maintenance and wide adoption across the ecosystem.

- **Two-step release.** The Release PR model requires two steps (trigger → review and merge) rather than a single click. This is an intentional trade-off for auditability and consistency with team norms.

## Alternatives
[alternatives]: #alternatives

### Custom Lean Workflow (Keep a Changelog + shell scripts)

Build a fully custom workflow using `keep-a-changelog-action` and per-repo `bump-version.sh` scripts.

- **Pros:** Total control over every step; no external tool dependency; preserves Keep a Changelog format.
- **Cons:** Higher maintenance burden; custom scripts for each language; no automatic changelog generation from commits. Maintenance of the custom scripts with the SDKs is a manual process.

### Release-it

[Release-it](https://github.com/release-it/release-it) is a powerful Node-based release tool that supports a direct-push model.

- **Pros:** Highly configurable; supports plugins for changelog generation; single-step release.
- **Cons:** Requires Node.js as a dependency in all SDK CI environments (including Go and Python repos); aligns with the direct-push model, which requires branch protection bypass; less alignment with the team's PR-based review norms.

### Direct Push Model

The workflow pushes the release commit and tag directly to `main` without a PR.

- **Pros:** True one-click experience; simpler workflow.
- **Cons:** Requires granting the automation identity branch protection bypass ("Admin-like" write access to `main`); changelog diff lands on `main` before human review; weaker audit trail.

This model was not chosen because it compromises branch protection integrity and does not provide the reviewable PR-based audit trail that the team requires.

### Continuous Deployment (release on every merge)

Automatically release a new version on every merge to `main`.

- **Why not:** The team explicitly wants a manual gate before releases. Not every merge warrants a release, and batching changes into deliberate releases is sometimes desirable. That said, **nightly builds** could be considered as a middle ground — building `main` on a schedule (or on every merge) so the latest successful build is always available for users to test, without cutting an official release. This is tracked as an [Additional Enhancement](#additional-enhancements).

## Prior Art
[prior-art]: #prior-art

- **Release Please** is used by Google Cloud client libraries across Go, Java, Python, Node.js, Ruby, PHP, and .NET — a very similar multi-language SDK ecosystem to OpenFGA. See: [googleapis/google-cloud-go](https://github.com/googleapis/google-cloud-go), [googleapis/google-cloud-python](https://github.com/googleapis/google-cloud-python).

- **Conventional Commits** is an industry-standard specification adopted by Angular, Electron, and many other large open-source projects to enable automated changelog generation and semantic versioning.

- **PR title validation** using [`ytanikin/pr-conventional-commits`](https://github.com/ytanikin/PRConventionalCommits) is already in use at [openfga/terraform-provider-openfga](https://github.com/openfga/terraform-provider-openfga), serving as prior art for enforcing Conventional Commits across the OpenFGA ecosystem.

- The proposed flow was validated end-to-end in a private test repository containing Go, Java, JavaScript, Python, and .NET SDK stubs. The test confirmed Release Please's ability to update `x-release-please-version` markers across all languages, generate accurate changelogs, and support explicit version overrides via the `Release-As` commit trailer.

## Additional Enhancements
[additional-enhancements]: #additional-enhancements

The following are valuable improvements that can be pursued independently after this RFC is implemented:

- **GPG-signed tags and GitHub Releases (implemented):** Rather than letting release-please create tags and releases via the GitHub API (which carry GitHub's signature, not the project's key), the pipeline sets `"skip-github-release": true` and creates the GPG-signed tag (`git tag -s`) and the GitHub Release (`gh release create`) directly in the `post-release` job. This gives the project full ownership of every signed release artifact. See [Signed Tags, Draft Releases, and Undraft-on-Publish](#how-it-works) for the full handshake (including the draft-then-undraft-on-publish-success behaviour).

- **Nightly builds:** Build `main` on every merge (or nightly) to provide users with a "latest" build for testing prior to an official release.

- **Publishing to GitHub Packages Registry (GPR):** Publish SDK artifacts to GPR in addition to the primary language registries (npm, PyPI, Maven Central, NuGet, pkg.go.dev).

- **Pre-release versioning:** The workflow supports pre-release increments in principle via the `explicit` bump type, but the exact UX for triggering pre-release versions (e.g., `1.0.0-beta.1`) might differ across SDKs and their individual registries, so there might need to be constraints or checks on allowed values for these.

- **`RELEASE.md` per-repository guide:** Each SDK repository should ship a `RELEASE.md` (or equivalent) that documents the day-to-day release process for contributors. This covers the versioning conventions in force, when to use `explicit`, how betas work, changelog authoring rules, and a troubleshooting section. A canonical template, based on real operational experience, is documented below and will be templated via sdk-generator alongside the workflow files.

<details>
<summary>Canonical <code>RELEASE.md</code> template</summary>

````markdown
# Release guide

This project uses [release-please](https://github.com/googleapis/release-please) via a
`workflow_dispatch`-triggered GitHub Actions workflow. This document explains how to cut
a release and what to watch out for.

---

## Versioning rules for this project

We are pre-1.0.0. Semver conventions are relaxed:

| Change type      | Bump                  | Example             |
|---               |---                    |---                  |
| Breaking change  | **Minor** (`0.x.0`)   | `1.3.0` → `1.4.0`  |
| Everything else  | **Patch** (`0.0.x`)   | `1.3.0` → `1.3.1`  |

Major bumps (`2.0.0`) are reserved for a deliberate 1.0.0 graduation decision — not for
routine breaking changes.

---

## Cutting a release

1. Go to **Actions → release-please** and click **Run workflow**.
2. Choose a bump type:
   - `patch` — bugfixes, docs, small changes
   - `minor` — breaking changes (see above)
   - `explicit` — you specify the exact version string (e.g. `1.4.0` or `1.4.0-beta.1`)
3. The workflow creates a release PR. Review it, then merge.
4. The GitHub Release and tag are created automatically on merge.

> **Note — release-please only understands `auto` or an explicit version string.**
> The `patch`, `minor`, and `major` options in the workflow dropdown are conveniences
> implemented in the workflow. The workflow reads the current manifest version, computes
> the next version (e.g. `1.3.11` + patch = `1.3.12`), and passes that computed string
> to release-please as an explicit `Release-As:` commit — exactly the same as choosing
> `explicit` and typing it yourself. There is no native patch/minor/major mode in
> release-please. This is why `explicit` is always the safest option when in doubt —
> you are just skipping the arithmetic step.

---

## When to use `explicit`

Use `explicit` and type the version yourself in any of these situations:

**After a beta or non-conventional tag.**
If the previous release was something like `1.3.12-beta.1`, release-please tracks the
base semver (`1.3.12`) but cannot reliably decide whether the next release should be
`1.3.12`, `1.3.13`, or `1.4.0`. It will often guess wrong, especially pre-1.0.0 where
commit-based bump rules don't map cleanly to our conventions.

The rule of thumb: **if the last tag had a pre-release suffix, always use `explicit` for
the next release.**

**After a manually created tag.**
Any tag created outside of the release-please workflow (e.g. hotfixes, manual git tags)
is invisible to release-please's version logic. Use `explicit` to anchor the next version
correctly.

**When you want a beta.**
Release-please does not increment pre-release suffixes automatically (`beta.1` → `beta.2`
does not happen on its own). If you want a beta series, use `explicit` for every beta,
incrementing the suffix manually:
```
1.4.0-beta.1  →  explicit: 1.4.0-beta.2  →  explicit: 1.4.0
```

---

## What goes in the changelog

Commit messages must follow [Conventional Commits](https://www.conventionalcommits.org/)
for release-please to group them correctly:

```
feat: add dark mode support          → Added
fix: correct null pointer in parser  → Fixed
docs: update API reference           → Documentation
perf: cache DNS lookups              → Changed
refactor: extract auth helper        → (hidden)
chore: bump dependencies             → (hidden)
```

Commits without a conventional prefix (e.g. `"Update README"`) are parsed but may appear
ungrouped. Always use a prefix.

---

## Troubleshooting

**"Invalid previous_tag parameter" error.**
The manifest version does not have a corresponding GitHub Release object. Reset the
manifest to the last valid tag:
```bash
echo '{ ".": "1.x.y" }' > .release-please-manifest.json
git commit -am "chore: reset manifest to v1.x.y"
git push origin main
```

**Duplicate release PRs.**
Close all stale ones and label them `autorelease: tagged` so release-please ignores them
on the next run. The workflow auto-closes stale open PRs on each dispatch, but merged
duplicates need manual labelling.

**Changelog shows everything ungrouped.**
Make sure `changelog-type` in `release-please-config.json` is set to `"default"`, not
`"github"`. The `"github"` type bypasses changelog sections entirely and calls GitHub's
release notes API directly.
````

</details>

