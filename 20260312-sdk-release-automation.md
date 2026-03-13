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
- [Migration](#migration)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Prior Art](#prior-art)
- [Additional Enhancements](#additional-enhancements)

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

- **GitHub Changelog Format:** The changelog format used by GitHub Releases. It lists changes grouped by type with links to PRs and contributors. This replaces the Keep a Changelog (KaC) format currently used across the SDKs.

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
│  • Updated CHANGELOG.md (GitHub changelog format)               │
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

- **Branch protection–friendly by default.** The GitHub App acts as a standard contributor; in the normal flow, no actor modifies `main` without at least one peer approval on the Release PR.
- **Reviewable diff.** The changelog and version bump are visible in a PR before the tag is ever created.
- **Audit trail.** The PR provides a permanent, reviewable history of who approved the release and what the changelog diff contained.
- **Consistent with team norms.** The release goes through the same PR review process as any other code change.

> **Note:** For explicit or non-auto version overrides, the workflow currently pushes an empty `Release-As` commit directly to `main` using the `github-actions[bot]` identity (not the GitHub App). If branch protection rules require all changes to go through a PR, you must grant an exception for `github-actions[bot]` to allow this direct push, or implement the override via a PR branch instead. See [Release Please documentation](https://github.com/googleapis/release-please?tab=readme-ov-file#how-do-i-change-the-version-number) for more details on version overrides and their limitations in manifest mode.

### Version Bumping with `x-release-please-version`

Release Please uses inline marker comments to locate and update version constants across language-specific files. By placing an `x-release-please-version` comment next to any version string, Release Please will automatically update it in the Release PR. The marker is language-agnostic — it works with any comment syntax:

```
VERSION = "0.7.2"  # x-release-please-version    (Python, properties files)
const SdkVersion = "0.7.2" // x-release-please-version  (Go, Java, C#, TypeScript)
<Version>0.7.2</Version><!-- x-release-please-version -->  (XML / .csproj)
```

Each SDK repository annotates every file that contains a version constant with this marker. When Release Please prepares a release, it scans for these markers and updates the version string inline, producing a clean diff in the Release PR. The list of files to scan is declared in `release-please-config.json` under the `extra-files` key.

### Changelog Format

The changelog format will be standardized to the **GitHub changelog format** across all SDKs. This format is very similar to Keep a Changelog but without the `[Unreleased]` section. Each release entry lists changes grouped by type with links to the relevant PRs and contributor attribution.

Example:

```markdown
## 0.7.3 (2026-03-12)

## What's Changed
* feat: add support for batch check by @contributor1 in https://github.com/openfga/go-sdk/pull/101
* fix: correct retry logic for transient errors by @contributor2 in https://github.com/openfga/go-sdk/pull/105
* chore: update dependencies by @contributor3 in https://github.com/openfga/go-sdk/pull/108

**Full Changelog**: https://github.com/openfga/go-sdk/compare/v0.7.2...v0.7.3
```

Additional notes (e.g., contributor acknowledgments, migration tips, or usage examples) can be added to the changelog when the release PR is reviewed or the release notes after creation, just as we do today.

### Explicit Version Overrides

Release Please auto-calculates the version bump from Conventional Commits (e.g., `feat:` triggers a minor bump, `fix:` triggers a patch bump). However, there are cases where the auto-calculated bump is not what we want.

The workflow supports explicit overrides via the `bump-type` input on the `workflow_dispatch` trigger:

- **`auto`** — Let Release Please determine the bump from commit history (default).
- **`patch`** — Force a patch bump.
- **`minor`** — Force a minor bump.
- **`major`** — Force a major bump.
- **`explicit`** — Supply an exact version string (e.g., `1.2.3` or `1.4.0-beta.1`) via the `release-version` input.

For non-`auto` bump types, the workflow computes the next version from the current version in `.release-please-manifest.json`, creates an empty commit with the `Release-As: X.Y.Z` trailer, and pushes it to `main`. Release Please then picks up the trailer and uses the specified version instead of auto-calculating.

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
  "packages": {
    ".": {
      "release-type": "simple",
      "changelog-path": "CHANGELOG.md",
      "bump-minor-pre-major": true,
      "extra-files": [
        "version/version.go",
        "gradle.properties"
      ]
    }
  }
}
```

Example `.release-please-manifest.json`:
```json
{
  ".": "0.7.2"
}
```

### GitHub Actions Workflow

The release workflow has two triggers:

- **`push` to `main`** — but the job only runs when the head commit message starts with `chore: release` or `chore(main): release` (i.e., the Release PR merge commit). This avoids running Release Please on every single merge to `main` and ensures it only fires when a release is actually being finalized.
- **`workflow_dispatch`** — a maintainer triggers from the GitHub UI to create or update the Release PR with a specific bump type.

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
          Explicit version to release (e.g. 1.2.3 or 1.4.0-beta.1). Only used
          when bump-type is set to 'explicit'. Leave blank otherwise.
        required: false
        type: string

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    # On push: only run when the release PR itself was merged.
    # On workflow_dispatch: always run (creates/updates the release PR).
    if: |
      github.event_name == 'workflow_dispatch' ||
      startsWith(github.event.head_commit.message, 'chore: release') ||
      startsWith(github.event.head_commit.message, 'chore(main): release')
    outputs:
      release_created: ${{ steps.release.outputs.release_created }}
      tag_name: ${{ steps.release.outputs.tag_name }}
    steps:
      - name: Generate token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}

      - name: Checkout
        uses: actions/checkout@v4
        with:
          token: ${{ steps.app-token.outputs.token }}
          fetch-depth: 0

      - name: Compute release-as
        id: compute-release-as
        if: github.event_name == 'workflow_dispatch'
        run: |
          BUMP="${{ inputs.bump-type }}"
          if [[ "$BUMP" == "patch" || "$BUMP" == "minor" || "$BUMP" == "major" ]]; then
            CURRENT=$(jq -r '.["."]' .release-please-manifest.json | cut -d'-' -f1)
            IFS='.' read -r MAJOR MINOR PATCH <<< "$CURRENT"
            if [[ "$BUMP" == "major" ]]; then
              NEXT="$((MAJOR + 1)).0.0"
            elif [[ "$BUMP" == "minor" ]]; then
              NEXT="${MAJOR}.$((MINOR + 1)).0"
            else
              NEXT="${MAJOR}.${MINOR}.$((PATCH + 1))"
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
          inputs.bump-type == 'major')
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          if [[ "${{ inputs.bump-type }}" == "explicit" ]]; then
            VERSION="${{ inputs.release-version }}"
          else
            VERSION="${{ steps.compute-release-as.outputs.value }}"
          fi
          git commit --allow-empty \
            -m "chore: release ${VERSION}" \
            -m "Release-As: ${VERSION}"
          git push

      - uses: googleapis/release-please-action@v4
        id: release
        with:
          token: ${{ steps.app-token.outputs.token }}
          config-file: release-please-config.json
          manifest-file: .release-please-manifest.json

  post-release:
    needs: release-please
    if: needs.release-please.outputs.release_created == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Generate token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}

      - name: Checkout
        uses: actions/checkout@v4
        with:
          token: ${{ steps.app-token.outputs.token }}

      - name: Post-release summary
        run: |
          echo "### Release ${{ needs.release-please.outputs.tag_name }} published" >> "$GITHUB_STEP_SUMMARY"
          echo "" >> "$GITHUB_STEP_SUMMARY"
          echo "Tag \`${{ needs.release-please.outputs.tag_name }}\` is now available." >> "$GITHUB_STEP_SUMMARY"
          echo "Add downstream steps here (e.g. publish packages, deploy)." >> "$GITHUB_STEP_SUMMARY"
```

Key details:

- **`Generate token`** mints a short-lived GitHub App token at the start of each job using `actions/create-github-app-token@v1`. This token is passed to the checkout step and to Release Please, ensuring all git operations and API calls use the App identity.
- **Job-level `if` guard:** On `push` events, the job is skipped unless the head commit message starts with `chore: release` or `chore(main): release`. This ensures Release Please only runs on push when the Release PR merge commit lands — not on every merge to `main`. On `workflow_dispatch`, the job always runs.
- **`Compute release-as`** only runs on `workflow_dispatch`. It reads the current version from `.release-please-manifest.json` and calculates the next semver for `patch`, `minor`, or `major` bumps. For `explicit`, the user-supplied version is used directly.
- **`Push Release-As commit`** only runs on `workflow_dispatch` for non-`auto` bump types. It creates an empty commit with the `Release-As: X.Y.Z` trailer on `main`, which Release Please picks up to override its auto-calculated version.
- **`release-please-action`** runs on every invocation (both triggers) and creates or updates the Release PR, or finalizes the release (creates the tag and GitHub Release) when the Release PR merge commit is detected.
- **`post-release`** is a downstream job that runs only when Release Please creates a release (i.e., when the Release PR is merged). It also mints its own App token for any downstream steps that need to interact with GitHub.

### Identity and Signing

Instead of using a Personal Access Token (PAT) tied to an individual maintainer, the workflow uses a **dedicated GitHub App** as its bot identity. This has several advantages:

- **No long-lived secrets.** The App's static credentials (`APP_ID` + `APP_PRIVATE_KEY`) are exchanged for a short-lived token at the start of each workflow run via [`actions/create-github-app-token@v1`](https://github.com/actions/create-github-app-token). The token expires after the run completes, so the blast radius of any credential compromise is limited to a single run.
- **Least privilege.** The App is granted only `contents: write` and `pull-requests: write` — the minimum permissions needed to push the `Release-As` commit, manage the Release PR, and create tags.
- **Verified commits.** Commits and tags made via the App token are automatically marked as **Verified** by GitHub, so no GPG key management is required.
- **Not tied to a person.** Unlike a PAT, the App identity is owned by the account or organization, not an individual. There is no risk of losing access when a maintainer rotates out.

**Setup steps:**

1. Create a GitHub App under **Settings → Developer settings → GitHub Apps** (or under the org's settings for org-wide use).
2. Disable webhooks (not needed).
3. Grant **Contents: Read & write** and **Pull requests: Read & write** permissions.
4. Generate a private key (`.pem` file).
5. Install the App on the target repository (or repositories).
6. Store `APP_ID` and `APP_PRIVATE_KEY` as repository (or organization) secrets.

For organization migration, the same App can be installed across multiple repositories from a single place, making it easy to manage centrally.

### End-to-End Flow

**On push to `main` (Release PR merge):**

When a Release PR is merged, the merge commit message starts with `chore: release` (or `chore(main): release`). The job-level `if` guard matches this pattern, so Release Please runs and finalizes the release — creating the git tag and the GitHub Release. For all other merges to `main`, the job is skipped entirely.

**On `workflow_dispatch` (manual trigger):**

1. **Trigger:** A maintainer navigates to **Actions → release-please → Run workflow** in the GitHub UI and selects the desired bump type (`auto`, `patch`, `minor`, `major`, or `explicit`).

2. **Version override (if applicable):** For non-`auto` bump types, the workflow computes the target version and pushes an empty `Release-As` commit to `main`.

3. **Release PR created/updated:** Release Please analyzes the commits since the last release, generates the changelog, updates all `x-release-please-version` markers, and opens (or updates) the Release PR targeting `main`.

4. **Review:** The team reviews the Release PR diff — changelog accuracy, version correctness, and any additional notes.

5. **Merge:** A maintainer merges the Release PR. This triggers the `push` path described above, which finalizes the release.

6. **Post-release:** The `post-release` job runs, producing a summary in the Actions UI. This job is the extension point for downstream steps such as publishing packages or triggering deployments.

7. **Publish:** The existing tag-triggered workflow (`.github/workflows/main.yaml`) detects the `v*` tag, runs the test suite, and publishes to the appropriate registry. The GitHub Release itself is created by Release Please in step 5.

### Consistency via sdk-generator

The release workflow, Release Please configuration, and Conventional Commits validation check will be **templated in the [sdk-generator](https://github.com/openfga/sdk-generator)** and applied uniformly to all SDK repositories. This ensures that any improvements to the release process propagate automatically to every SDK.

## Migration
[migration]: #migration

### Changelog Format Migration

The OpenFGA SDKs currently follow a variant of Keep a Changelog with `[Unreleased]` sections and `Added` / `Fixed` / `Changed` groupings. While the structure is broadly consistent, the formatting is not strict enough to be reliably parsed by automated tooling.

A **one-time manual migration** of every SDK repository's `CHANGELOG.md` to the GitHub changelog format is required. This migration must be done before enabling Release Please on each repository so that the tool can correctly parse existing entries and append new ones. The steps are:

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

> **Note on the language repository:** The [openfga/language](https://github.com/openfga/language) repository is a mono-repo that contains multiple packages (one per language). Each package is released independently with its own tag prefix — e.g., `pkg/js/v0.2.1`, `pkg/go/v0.2.0`, etc. Release Please supports this via its multi-package configuration, where each package path in `release-please-config.json` maps to its own manifest entry, tag prefix, and set of `x-release-please-version` markers.

### GPG Key Retirement

With this flow, release tags are created by Release Please via the GitHub API using the GitHub App token. GitHub automatically signs API-created tags with its own verified signature (`GPG Key ID: B5690EEEBB952194`), so the tags receive the **Verified** badge without any maintainer-managed GPG keys. Existing GPG signing infrastructure (keys in CI secrets, `gpg-agent` setup steps) can be removed as part of the migration.

### Conventional Commits Enforcement

The PR title validation workflow described in [Conventional Commits Validation](#conventional-commits-validation) will be added to all SDK repositories and configured as a **required status check** in each repository's branch protection rules. Existing commit history does not need to be rewritten — the validator applies only to new PRs going forward.

### Rollout Plan

1. **Pilot:** Deploy the workflow to a single SDK repository (e.g., the Go SDK) to validate the end-to-end flow.
2. **Iterate:** Incorporate feedback, adjust configuration, and refine the changelog format.
3. **Expand:** Roll out to all remaining SDK/CLI/extension repositories.

## Drawbacks
[drawbacks]: #drawbacks

- **Changelog format change.** Moving from Keep a Changelog to the GitHub changelog format is a visible change for contributors and users who follow the changelog. However, the formats are very similar, and the GitHub format has the benefit of automatic generation with contributor attribution.

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

- **Nightly builds:** Build `main` on every merge (or nightly) to provide users with a "latest" build for testing prior to an official release.

- **Publishing to GitHub Packages Registry (GPR):** Publish SDK artifacts to GPR in addition to the primary language registries (npm, PyPI, Maven Central, NuGet, pkg.go.dev).

- **Pre-release versioning:** The workflow supports pre-release increments in principle via the `explicit` bump type, but the exact UX for triggering pre-release versions (e.g., `1.0.0-beta.1`) might differ across SDKs and their individual registries, so there might need to be constraints or checks on allowed values for these.
