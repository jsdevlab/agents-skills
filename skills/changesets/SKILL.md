---
name: changesets
description: Use this skill to work with Changesets (@changesets/cli) in a repo that has a .changeset/ folder, changesets/action in CI, or is adopting it — create or edit a changeset, decide the semver bump it declares, write the consumer-facing changelog entry it generates, run version bumps and npm publishes, review a Version Packages PR, tune .changeset/config.json (fixed/linked groups, access, ignore), automate releases, and manage prerelease or snapshot versions. Also use it for the tool's failure modes — "no unreleased changesets", a stuck release PR, missing pkg@x.y.z tags — and when an npm monorepo asks which packages a pending release will publish. Skip general semver theory, hand-maintained CHANGELOG.md files, PyPI/Docker publishing, lockfile or peer-dependency conflicts, and test/lint CI work. Chinese cues — 发版, 版本发布, 发包, 版本号.
---

# Changesets

Changesets separates "intent to release" from "actually releasing". A changeset is a Markdown file in `.changeset/` whose YAML frontmatter names the packages and semver bumps, and whose body becomes the changelog entry. Later, `changeset version` consumes all pending changesets to update `package.json` files, dependency ranges, and `CHANGELOG.md`; `changeset publish` publishes the new versions and creates git tags.

That separation is the point: contributors record release intent while the change is fresh, and maintainers review the aggregate before publishing. Work with the workflow rather than bypassing it — do not hand-edit versions or changelogs that `changeset version` is supposed to generate.

## Before changing anything: inspect the repository

Release decisions depend on context you cannot guess: package names, the configured base branch, fixed/linked groups, and whether prerelease mode is active. Read these first.

```bash
# Package manager and workspace layout
# pnpm-lock.yaml => pnpm, package-lock.json => npm, yarn.lock => yarn
cat package.json
cat pnpm-workspace.yaml        # pnpm only
cat .changeset/config.json
# Pending changesets and prerelease state
ls .changeset/*.md
cat .changeset/pre.json        # only exists in prerelease mode
# Git context: branch, pending edits, commits since the base branch
git status --short
git branch --show-current
git log --oneline -5
```

Then build a small mental model:

1. **Package manager** — from the lockfile. Use that manager's command form consistently (see the shorthand below).
2. **Workspace packages** — read every `packages/*/package.json` (or the globs in the workspace file). Changeset frontmatter uses the exact `name` field from `package.json`, not the folder name. Note which packages are `"private": true`.
3. **`.changeset/config.json`** — `baseBranch`, `access`, `fixed`, `linked`, `ignore`, `privatePackages`, and `changelog` all change release behavior. See `references/config.md` when any of them matters.
4. **Pending changesets** — read them before adding another; they describe what is already queued and may already cover this change.
5. **Prerelease mode** — `.changeset/pre.json` means `version` produces prerelease versions (e.g. `1.2.0-beta.0`) and moves changesets to `.changeset/pre/`. Do not enter or exit prerelease mode casually.
6. **Git state** — the diff tells you which packages changed and how significant the change is. `baseBranch` must exist locally for `status` and `add --since` to work; fetch it if needed.

If `.changeset/` is missing, the project is not using Changesets yet. Ask whether to set it up before creating config or installing `@changesets/cli`; installation changes the dependency graph.

### Command shorthand

Below, `CHANGESET` means the project's package-manager command:

| Manager | Command |
| --- | --- |
| pnpm | `pnpm changeset` |
| npm | `npx @changesets/cli` |
| yarn | `yarn changeset` |

## Creating a changeset

### 1. Decide whether a changeset is needed

A changeset is release intent, not a commit log. Add one when a consumer of a published package can observe the change:

- API surface changes: new/removed/renamed exports, changed signatures, new options, deprecations.
- Bug fixes, performance fixes, or security fixes that users should receive.
- Dependency updates that affect the package's behavior or requirements.
- Documentation that ships inside the published package.

Usually no changeset for: tests, CI configuration, repo-only docs, formatting, internal tooling. The official guidance is that not every change needs a changeset, and blocking PRs on a missing changeset is discouraged — unless this repository has opted into that policy. If the repo requires one and the change truly releases nothing, use an empty changeset (`CHANGESET add --empty -m "..."` or a file with empty frontmatter).

If the user's intent is ambiguous, infer from the diff, state the assumption, and prefer the smaller bump when the change is not consumer-visible.

### 2. Choose the semver bump

`version` flattens all pending changesets for a package into a single bump — the highest one wins — so each changeset only needs to describe its own change accurately.

| Change | Bump |
| --- | --- |
| Breaking public API: removed/renamed exports, incompatible behavior, dropped runtime support | `major` |
| New feature or public API, new option, non-breaking deprecation | `minor` |
| Bug fix, performance/security fix, internal dependency bump, docs | `patch` |

For packages still below `1.0.0`, many teams treat breaking changes as `minor` by convention. Follow the repository's existing changesets and release notes when they show a policy; otherwise ask before making a breaking-change call.

In a monorepo, Changesets automatically patch-bumps dependents when a dependency's new version would fall outside their declared range. You do not need to list those dependents yourself, but if a dependent's own public API also changed, add a separate changeset for it with the appropriate bump.

### 3. Write the summary for package consumers

The body becomes the changelog entry. A useful summary answers:

- **What** changed, in user-facing terms.
- **Why** it changed, when that is not obvious.
- **How** to migrate, for breaking changes.

Write it as a short paragraph or a bullet list, not a commit subject. Include code snippets when they clarify an API change.

Good:

```md
Add a `stream` option to `parse()`. Passing `{ stream: true }` returns an async
iterable instead of buffering the whole document, which keeps memory flat for
large inputs. The default behavior is unchanged.
```

Weak: `fix parse` — it tells a consumer nothing they can act on.

### 4. Create the file

When you already know the packages and bumps from the diff, writing the file directly is the most deterministic approach:

```md
---
"@scope/pkg-a": minor
"@scope/pkg-b": patch
---

Add a `stream` option to `parse()`. ...
```

Save it as `.changeset/<short-descriptive-name>.md`. Filenames are arbitrary and must be unique; the CLI generates human-readable random names, but descriptive names such as `stream-option.md` are fine. The `.changeset/` folder is only for changesets and config — do not leave unrelated files there.

Alternatively, let the CLI create it:

```bash
# v3: bump flags accept comma-separated package names; flags can be combined
CHANGESET add --minor "@scope/pkg-a" --patch "@scope/pkg-b" -m "Add a stream option to parse()"
CHANGESET add --patch "@scope/pkg-a,@scope/pkg-b" -m "Fix empty input handling"
CHANGESET add --empty -m "Update CI only"
```

**The plain `CHANGESET` / `CHANGESET add` command is interactive.** In a non-interactive agent session it will block or fail on stdin. Always pass `--major`, `--minor`, `--patch`, and `-m`, or write the file directly.

`changeset init` is also interactive in v3 (it asks about GitHub changelog integration). Do not run it from an automated session. If setup is actually needed, create `.changeset/config.json` yourself using the default in `references/config.md`, or ask the user to run `CHANGESET init` in their terminal.

### 5. Verify the result

Re-read the changeset file and check every package name against its `package.json`; a typo makes the changeset fail or target nothing. Then:

```bash
CHANGESET status
```

`status` prints pending changesets and the releases they imply. It needs the configured `baseBranch` to exist locally; if the checkout is shallow or the branch is missing, fetch it or pass `--since <valid-ref>`. In a repository without git history, `status` will fail with "Failed to find where HEAD diverged" — inspect the changeset files directly instead.

## Versioning and publishing

These are two separate, reviewable steps. Do not skip the review, and do not commit anything between them: `publish` assumes the last commit is the version commit.

### Version

```bash
CHANGESET version
```

This consumes every pending changeset (the files are deleted), bumps versions, updates internal dependency ranges, and writes `CHANGELOG.md` files. Review the diff before committing:

```bash
git diff
git add .
git commit -m "Version packages"
```

Two behaviors surprise people:

- **v3 exits with code 1 when there are no unreleased changesets.** That is expected, not a failure of the tool. Scripts must tolerate it.
- **There is no `--dry-run`.** To preview, run `CHANGESET status --output <temp-file>` (do not put it in `.changeset/`), or run `version` on a throwaway branch and inspect the diff.

Empty changesets produce no version bump. `fixed`/`linked` config can bump packages you did not mention — see `references/config.md`.

### Publish

Publishing is irreversible and outward-facing. Confirm with the user before running it, especially for a first publish, a scoped package, or a non-`latest` dist-tag.

```bash
CHANGESET publish
git push --follow-tags
```

`publish` checks each package's version against the registry and publishes only versions that are not already published, then creates git tags named `pkg-name@X.X.X` (or `vX.X.X` in a single-package repo). Push the tags; they are how people find release code.

Pre-flight checks:

- **First publish:** `access` in `.changeset/config.json` defaults to `"restricted"`. For public scoped packages set `"access": "public"` or add `"publishConfig": { "access": "public" }` to that package. Otherwise npm will reject the publish or publish privately.
- **Never publish a new package accidentally:** `publish` publishes any local version not on the registry. Set `"private": true` on packages that must never be published (test fixtures, apps).
- **Prerelease/snapshot/backport:** pass a dist-tag (`--tag beta`) so the version does not become `latest`.
- **Auth:** `npm whoami` confirms the registry identity. With 2FA, `--otp <code>` avoids an interactive prompt.

Use `--no-git-tag` when another tool handles tagging, and `CHANGESET publish-plan` to see what would be published without publishing.

For CI, prefer npm trusted publishing over long-lived tokens; see `references/ci-automation.md`.

## Configuration essentials

Most repositories only touch a few options. Read the file, do not assume defaults.

| Option | Default | Effect |
| --- | --- | --- |
| `access` | `"restricted"` | npm visibility. Set `"public"` for public packages. |
| `baseBranch` | `"main"` | Branch used to detect changed packages. Must exist locally. |
| `changelog` | `"@changesets/cli/changelog"` | Changelog generator; `false` disables changelogs; tuple form passes options (e.g. `["@changesets/changelog-github", { "repo": "owner/repo" }]`). |
| `fixed` | `[]` | Groups versioned and published together, always sharing a version. |
| `linked` | `[]` | Groups sharing version numbers, but only packages with changesets are bumped. |
| `ignore` | `[]` | Temporarily skip packages; publishing fails if mixed with non-ignored packages or required dependency updates. |
| `privatePackages` | `{ version: false, tag: false }` | v3 no longer versions private packages by default. |
| `updateInternalDependencies` | `"patch"` | Whether internal ranges update on patch releases; `"minor"` updates only for minor+ changes. |
| `changedFilePatterns` | `["**"]` | Which changed files mark a package as changed. |
| `format` | `"auto"` | Formatter for generated files: prettier/oxfmt/deno/biome/dprint/false. |

Full option details and recipes: `references/config.md`.

## CI automation

The standard setup uses `changesets/action` v2 (pin its commit SHA in workflows): on pushes to the base branch it opens/updates a "Version Packages" PR when changesets exist, and publishes when the version PR is merged. To require changesets on PRs, run `CHANGESET status --since main` as a check (blocking), or use the Changesets GitHub Bot / `changesets/action/pr-status` (non-blocking).

Two prerequisites are easy to miss:

- Repository setting **Actions → General → Allow GitHub Actions to create and approve pull requests** must be enabled.
- **Trusted publishing** is recommended over npm tokens. Grant `id-token: write` only to the publish job, and split build/test/publish into separate jobs.

Full workflow YAML (trusted publishing, token-based, version-only, git-tags-only) and v2 action input notes: `references/ci-automation.md`.

## Prereleases and snapshot releases

- **Prerelease:** `CHANGESET pre enter beta` → create changesets → `version` / `publish` as usual (versions like `1.2.0-beta.0`, dist-tag `beta`) → `CHANGESET pre exit` → `version` / `publish` for stable. `.changeset/pre.json` holds the state; versioned changesets move to `.changeset/pre/` for the stable changelog. Read `references/prereleases-and-snapshots.md` before doing this — mistakes are hard to undo.
- **Snapshot:** `CHANGESET version --snapshot mytag` then `CHANGESET publish --tag mytag --no-git-tag`. Snapshots exist for testing; never let them become `latest`, and do not merge the snapshot version changes back into the release branch.

## Troubleshooting

Common symptoms and fixes live in `references/troubleshooting.md`. Quick map:

| Symptom | Likely cause |
| --- | --- |
| "Failed to find where HEAD diverged from main" | `baseBranch` missing locally, shallow clone, or a non-git directory. Fetch the branch or pass `--since`. |
| `version` exits 1 | No unreleased changesets in v3. Expected. |
| `changeset init` hangs/fails | It is interactive; create config directly or run it in a terminal. |
| Publish fails with access/404 | `access` is `restricted` for a first public scoped publish, or the scope/registry auth is wrong. |
| Publish fails mentioning ignored packages | `ignore` was mixed with non-ignored releases or a required dependency update. |
| New package published unexpectedly | `publish` publishes any unpublished local version; set `"private": true`. |
| Tags missing on remote | `changeset publish` creates tags locally; push with `git push --follow-tags`. |
| Version PR not created / workflows not running | Missing Actions PR permission, or the version PR needs a PAT/GitHub App token to trigger workflows. |

## References

- `references/cli.md` — every command and flag, including v3 changes and non-interactive usage.
- `references/config.md` — full config reference, recipes, fixed vs linked semantics.
- `references/ci-automation.md` — GitHub Actions workflows and npm authentication.
- `references/prereleases-and-snapshots.md` — prerelease and snapshot workflows in depth.
- `references/troubleshooting.md` — error-by-error fixes and migration notes.
