---
name: release-plz
description: >-
  Use this skill for Rust release automation with release-plz: creating release PRs from conventional commits, bumping versions, publishing crates, creating git tags and GitHub/GitLab/Gitea releases. It covers `release-plz.toml` (`[workspace]`, `[[package]]`, `[changelog]`), `release_always`, `version_group`, `git_only`, `publish`, changelog customization, semver checks, the CLI commands (`update`, `release-pr`, `release`, `set-version`), and CI setup with the GitHub Action or GitLab CI component, including tokens, permissions, trusted publishing, concurrency, and troubleshooting. Trigger it when the user mentions release-plz, a release PR, Rust version bumps or changelogs, cargo publish automation, `release-plz.toml`, a release job in GitHub Actions or GitLab CI, or asks why a Rust package was not published. Chinese cues — Rust 发版, 发布 PR, 版本号, 变更日志, cargo 发布, GitLab CI 发版. Skip general semver theory, non-Rust package publishing, and generic CI/Docker work.
---

# Release-plz

[release-plz](https://release-plz.dev) automates Rust releases from conventional commits. It has two separate phases, and mixing them up is the most common source of confusion:

- **`release-plz release-pr`** — compares the current commit graph with the last released version, computes the next versions, updates `Cargo.toml` and changelogs, and opens or updates a **release PR**. It does not publish anything.
- **`release-plz release`** — looks for package versions in `Cargo.toml` that are not in the cargo registry yet, then publishes them, creates git tags, and creates GitHub/GitLab/Gitea releases. It does not edit `Cargo.toml` or push commits.

A normal CI setup runs both on every push to the main branch. The PR job prepares the next release; the release job publishes any version that is already in `Cargo.toml` but absent from the registry. Merging the release PR is what moves the new version into `Cargo.toml`, so the next release job sees it.

`release-plz update` is the local version of the PR job: it edits the working tree without committing or pushing. `release-plz set-version` is the manual override for a version that release-plz computed incorrectly.

## Inspect the repository first

Release behavior depends on the workspace shape, the registry, and existing CI. Read these before changing anything:

```bash
# Workspace / package layout, versions, publish settings
cat Cargo.toml
find . -name Cargo.toml -not -path './target/*'
grep -R "publish\|version" --include=Cargo.toml .

# Existing release-plz config (lookup order)
ls release-plz.toml .release-plz.toml .config/release-plz.toml 2>/dev/null
cat release-plz.toml 2>/dev/null

# Existing release history and CI
git tag --sort=-creatordate | head -20
git log --oneline -20
grep -R "release-plz\|cargo publish\|release-pr" .github/workflows .gitlab-ci.yml 2>/dev/null

# Tooling
release-plz --version
cargo --version
```

Then establish:

1. **Single crate or workspace?** Workspace members and `[workspace.package].version` change how versions and changelogs are written.
2. **Which packages are publishable?** `publish = false` or `publish = []` in `Cargo.toml`, private crates, and examples that do not declare `publish`.
3. **Which registry?** crates.io by default; private registries need a cargo registry name and a `CARGO_REGISTRIES_<NAME>_TOKEN` variable.
4. **Which forge and CI?** GitHub Action, GitLab CI component, or a manual CLI job. The forge determines the token and release behavior.
5. **Which release strategy?** Default "release on every main commit" (`release_always = true`) or "release only when the release PR is merged" (`release_always = false`, GitHub only).

## Choose a release strategy

| Situation | Settings | CI shape |
| --- | --- | --- |
| Normal published crates | Defaults (`release_always = true`) | `release-pr` + `release` on push to main |
| Publish only after merging the release PR | `release_always = false` | Same two jobs; `release` skips non-release-PR commits. GitHub only. |
| Private / unpublished crates, tags only | `git_only = true` | Same two jobs; `cargo publish` is skipped |
| Publish to a private registry | `registry = "my-registry"` in the action/CLI; `CARGO_REGISTRIES_MY_REGISTRY_TOKEN` | Same two jobs |
| Tag/release but no crates.io publish | `publish = false` | `release` still checks the registry; use `git_only` to avoid it |
| Subset of workspace released together | `version_group = "group-name"` per package | Same two jobs |
| Exclude a package entirely | `release = false` on that package | Same two jobs; package is ignored |

`release_always = false` is supported on GitHub only. It decides a release is pending by checking whether the latest commit belongs to a PR whose branch starts with `pr_branch_prefix` (default `release-plz-`). Use the default merge strategy for release PRs; with squash merges, the release PR commit is rewritten and the release can be skipped or raced.

`git_only = true` cannot be combined with `publish = true`; it is an error. Packages with `publish = false` in `Cargo.toml` are still tagged in git-only mode.

## Configuration essentials

release-plz looks for the config file in this order and uses the first that exists:

1. `release-plz.toml`
2. `.release-plz.toml`
3. `.config/release-plz.toml`

The file is optional; defaults are designed to work out of the box. The full field reference lives in `references/configuration.md`.

```toml
[workspace]
allow_dirty = false
changelog_update = true
dependencies_update = false
git_release_enable = true
git_tag_enable = true
release = true
release_always = true
semver_check = true
pr_branch_prefix = "release-plz-"
pr_labels = ["release"]
publish_timeout = "30m"

[[package]]
name = "internal-tools"
publish = false
release = false
```

### Fields that change release behavior

| Field | Default | Effect |
| --- | --- | --- |
| `release` | `true` | `false` excludes a package from update, PR, and release. |
| `release_always` | `true` | `false` releases only commits associated with a release PR. GitHub only. |
| `publish` | `true` | `false` skips `cargo publish` but still creates tags/releases and still checks the registry. |
| `git_only` | `false` | Use git tags instead of the registry; skips `cargo publish`. Cannot combine with `publish = true`. |
| `git_tag_enable` | `true` | `false` stops tag creation, which also prevents the forge release. |
| `git_release_enable` | `true` | `false` keeps tags but skips GitHub/GitLab/Gitea releases. |
| `changelog_update` | `true` | `false` skips changelog edits. |
| `version_group` | none | Packages sharing a value get the same next version (the highest in the group). |
| `semver_check` | `true` | Runs `cargo-semver-checks` for library crates and bumps major on detected breakage. |
| `dependencies_update` | `false` | `true` runs `cargo update` for all dependencies; `false` updates only workspace packages. |
| `release_commits` | all commits | Regex; a package is bumped only if at least one commit matches. Filtered commits still appear in the changelog. |
| `custom_major_increment_regex` / `custom_minor_increment_regex` | none | Use non-conventional prefixes to force major/minor bumps. |
| `features_always_increment_minor` | `false` | Makes `feat:` bump minor on `0.x`; violates Cargo SemVer for `0.x`. |
| `pr_branch_prefix` | `release-plz-` | Release PR branch prefix; changing it orphans the old PR. |
| `pr_draft`, `pr_labels`, `pr_name`, `pr_body` | see reference | PR presentation controls. |
| `publish_features`, `publish_all_features`, `publish_no_verify`, `publish_allow_dirty`, `publish_timeout` | see reference | Pass-through to `cargo publish`. |

### Version bumps in `0.x`

By default, release-plz follows Cargo SemVer for `0.x`: a `feat:` commit bumps the patch version (`0.1.0 -> 0.1.1`), not the minor version. `cargo-semver-checks` bumps the major version when it detects API breakage, including `0.x -> 1.0.0`. Set `features_always_increment_minor = true` only if you deliberately want `feat:` to bump `0.x -> 0.(x+1)`; the docs warn that this violates Cargo SemVer.

### Changelog

release-plz uses git-cliff internally and defaults to Keep a Changelog. The `[changelog]` section controls `header`, `body`, `trim`, `protect_breaking_commits`, `tag_pattern`, `sort_commits`, `commit_preprocessors`, `postprocessors`, `commit_parsers`, and `link_parsers`. Full examples and the default templates live in `references/changelog-and-semver.md`; the complete git-cliff template syntax is linked, not copied.

## GitHub Actions

Use two jobs: `release-pr` (writes the PR) and `release` (publishes). Do not put them in the same job; `release` needs different permissions and must not be canceled by concurrency.

```yaml
name: Release-plz

on:
  push:
    branches: [main]

permissions:
  contents: write

jobs:
  release-plz-release:
    name: Release-plz release
    runs-on: ubuntu-latest
    if: github.repository_owner == 'your-org'
    permissions:
      contents: write
      pull-requests: read
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
          persist-credentials: false
      - uses: dtolnay/rust-toolchain@stable
      - name: Run release-plz
        uses: release-plz/action@1fb72a9583c3601c197b3bd6fc6ce50d58e28ef3 # v0.5.148
        with:
          command: release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}

  release-plz-pr:
    name: Release-plz PR
    runs-on: ubuntu-latest
    if: github.repository_owner == 'your-org'
    permissions:
      contents: write
      pull-requests: write
    concurrency:
      group: release-plz-${{ github.ref }}
      cancel-in-progress: false
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
          persist-credentials: false
      - uses: dtolnay/rust-toolchain@stable
      - name: Run release-plz
        uses: release-plz/action@1fb72a9583c3601c197b3bd6fc6ce50d58e28ef3 # v0.5.148
        with:
          command: release-pr
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

Key points:

- **`fetch-depth: 0` is mandatory.** release-plz needs the full history and the release PR commit.
- **`persist-credentials: false` is the secure default.** Set it to `true` only when signing tags or when later steps use `git push`.
- **Do not add a canceling concurrency group to the release job.** A canceled release can be skipped. Put concurrency only on the PR job with `cancel-in-progress: false`.
- **The default `GITHUB_TOKEN` cannot trigger further workflows.** PR checks and `release`/`tag` workflows will not run. Use a PAT or a GitHub App token if you need them.
- **Trusted publishing:** omit `CARGO_REGISTRY_TOKEN`, add `id-token: write` to the release job, and configure crates.io trusted publishing. New crates still need a manual first publish.
- **Pin the action.** Use a release tag or a commit SHA; the docs' `@v0.5` moves with every release. v0.5.148 pins commit `1fb72a9583c3601c197b3bd6fc6ce50d58e28ef3` and defaults to release-plz 0.4.3; override with the `version` input if needed.
- **Outputs:** `prs`, `pr`, `releases`, `prs_created`, `releases_created`. Use `pr` / `releases` for labels, comments, or follow-up jobs.

Full workflows, PAT/App setup, trusted publishing, labels, auto-merge, scheduled runs, and outputs: `references/github-actions.md`.

## GitLab CI

GitLab uses the official CI component. It creates `release-plz:pr` and `release-plz:release` jobs and runs them on the default branch (not tags).

```yaml
include:
  - component: $CI_SERVER_FQDN/release-plz/components/release-plz@0.1.0
    inputs:
      stage: deploy
```

Prerequisites:

- Create a project or group access token with the **Maintainer** role and the `api` + `write_repository` scopes.
- Store it as a masked CI/CD variable named `RELEASE_PLZ_TOKEN`.
- Find the bot user created for the token, derive its email as `<username>@noreply.<gitlab-host>` (for gitlab.com: `<username>@noreply.gitlab.com`), and store it as `RELEASE_PLZ_BOT_EMAIL`.
- For a private registry, store its token as `CARGO_REGISTRIES_<NAME>_TOKEN`.

Component inputs include `stage`, `image`, `cwd`, `tags`, `git_credentials`, `git_submodule_recursive`, `bot_email`, `bot_name`, `release_args`, and `pr_args`. The component handles the token remote, git identity, cargo cache, shallow-checkout protection, and both release-plz commands.

If the component is not an option, a manual job can run `release-plz release-pr --forge gitlab --git-token "$RELEASE_PLZ_TOKEN"` and `release-plz release --forge gitlab --git-token "$RELEASE_PLZ_TOKEN"`; it must install Rust and release-plz, set the git identity, and configure the remote with the token. Full examples and submodule handling: `references/gitlab-ci.md`.

## CLI quick reference

```bash
release-plz update --allow-dirty                 # edit versions/changelogs locally, no commit
release-plz release-pr --git-token "$TOKEN"      # open/update the release PR
release-plz release --dry-run                    # simulate publishing
release-plz release                              # publish unpublished versions
release-plz set-version my-crate@1.2.3           # manual version override
release-plz generate-schema                      # writes .schema/latest.json
```

`update` has no `--dry-run`; it edits files. `release --dry-run` performs checks without uploading, but still needs forge credentials to inspect release PRs. `set-version` updates `Cargo.toml`, local dependency requirements, `Cargo.lock`, and the changelog. `generate-schema` writes to `.schema/latest.json` (do not commit it). `init` is interactive and uses the `gh` CLI; do not run it in an automated session.

For a local comparison without the registry, pass `--registry-manifest-path <path-to-previously-released-Cargo.toml>`; this is useful for offline testing and for simulating what release-plz will do. Full command/flag reference: `references/cli.md`.

## Troubleshooting map

| Symptom | Likely cause |
| --- | --- |
| Release PR is not created | Missing `pull-requests: write`, `fetch-depth: 0`, or the action hit the default `GITHUB_TOKEN` restriction; check the job log and repo Actions permissions. |
| CI checks do not run on the release PR | The default `GITHUB_TOKEN` cannot trigger workflows. Use a PAT or GitHub App token. |
| `release` does nothing after merging the PR | `release_always = false` did not associate the commit with a release PR (wrong branch prefix, squash merge, or merge queue); or the version is already published. |
| `release` hangs | Set a lower `publish_timeout` (default `30m`) and check registry access; use `-vv` for DEBUG logs. |
| Version bump is not what you expected | `feat:` on `0.x` bumps patch by default; semver checks may bump major; `release_commits` / custom regex may filter. Fix the release PR or use `set-version`. |
| Release PR opens too often | Any packaged file change triggers it; inspect `cargo package --list` and adjust `include`/`exclude` in `Cargo.toml`. |
| `git release not configured` | The CLI `release` needs `--git-token`/`--forge` (or the GitHub Action env). |
| Action misbehaves after an update | Pin the action to a version/SHA and pin the release-plz CLI with the `version` input. |
| GitLab job cannot push or open MRs | Token lacks Maintainer role or `api`/`write_repository`; missing `RELEASE_PLZ_BOT_EMAIL`; token not masked. |
| `cargo-semver-checks` missing or slow | The GitHub Action installs it; a manual CLI/CI job must install it. Disable with `semver_check = false` only if intentional. |

Detailed causes and fixes: `references/troubleshooting.md`.

## References

- `references/configuration.md` — every `[workspace]`, `[[package]]`, and `[changelog]` field, defaults, interactions, and recipes.
- `references/cli.md` — commands, flags, local workflows, JSON output, and `--registry-manifest-path`.
- `references/github-actions.md` — full workflows, inputs/outputs, tokens, permissions, trusted publishing, and advanced recipes.
- `references/gitlab-ci.md` — component setup, inputs, tokens, manual CLI fallback, and submodules.
- `references/changelog-and-semver.md` — changelog templates, parsers, preprocessors, and semver check behavior.
- `references/troubleshooting.md` — symptom-by-symptom fixes and migration notes.
