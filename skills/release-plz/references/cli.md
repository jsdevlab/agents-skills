# release-plz CLI reference

The binary is `release-plz`. Commands operate on the Cargo project in the current directory unless `--manifest-path` is given. The CLI and the GitHub Action run the same release-plz version; the action installs the version from its `version` input (0.4.3 for action v0.5.148 unless overridden).

## Contents

- [Commands at a glance](#commands-at-a-glance)
- [Global flags and logging](#global-flags-and-logging)
- [update](#update)
- [release-pr](#release-pr)
- [release](#release)
- [set-version](#set-version)
- [init](#init)
- [generate-schema and utilities](#generate-schema-and-utilities)
- [Local testing with a registry manifest](#local-testing-with-a-registry-manifest)
- [Exit codes and no-op behavior](#exit-codes-and-no-op-behavior)

## Commands at a glance

```text
release-plz update                # edit versions/changelogs locally; no commit, no push
release-plz release-pr            # open/update the release PR
release-plz release               # publish unpublished versions, tag, forge release
release-plz set-version           # manually set package/workspace versions
release-plz init                  # interactive GitHub setup; uses gh
release-plz generate-schema       # write .schema/latest.json
release-plz generate-completions  # shell completions
release-plz check-updates         # check for a newer release-plz
```

Common options on `update`, `release-pr`, and `release`:

| Flag | Meaning |
| --- | --- |
| `--manifest-path <Cargo.toml>` | Project/workspace manifest. Defaults to `./Cargo.toml`. |
| `--registry <name>` | Cargo registry name; must exist in the cargo config. Defaults to the package's `publish` field, then crates.io. |
| `--config <path>` | Config file; otherwise checks `release-plz.toml`, `.release-plz.toml`, `.config/release-plz.toml`. |
| `--repo-url <url>` | Forge repository URL for links/PRs. Defaults to the origin remote. |
| `--forge <github\|gitea\|gitlab>` | Forge kind. GitHub is inferred; set explicitly for GitHub Enterprise Server, GitLab, or Gitea. Alias: `--backend`. |
| `--git-token <token>` | Forge token. Env: `GIT_TOKEN`. Alias: `--github-token`. |
| `-v`, `-vv`, `-vvv` | Increase log verbosity; `-vv` is DEBUG, `-vvv` is TRACE. |

## Global flags and logging

```bash
release-plz --version
release-plz --help
release-plz update --help
release-plz release --help
```

Logs default to INFO. For debugging:

```bash
release-plz -vv update
RELEASE_PLZ_LOG=DEBUG release-plz release
RELEASE_PLZ_LOG=TRACE release-plz release-pr
```

`RELEASE_PLZ_LOG` sets the log level without incrementing verbosity. `-v` adds source locations; `-vv` adds DEBUG; `-vvv` adds TRACE.

## update

Edits versions and changelogs in the working tree based on commits since the last released version. It **does not** commit, push, tag, or publish, and it has **no `--dry-run`**; inspect `git diff` before keeping the result.

```bash
release-plz update
release-plz update --allow-dirty
release-plz update -p my-crate
release-plz update --registry-manifest-path /path/to/previous/Cargo.toml
```

Behavior:

1. Downloads the last released packages from the registry (or reads `--registry-manifest-path`).
2. Compares local packages with the released ones to find new commits.
3. Runs `cargo-semver-checks` for library packages when enabled.
4. Computes the next version from conventional commits and semver.
5. Updates `Cargo.toml`, local dependency requirements, `Cargo.lock`, and changelogs.
6. Optionally runs `cargo update` for all dependencies with `-u/--update-deps`.

Flags:

| Flag | Meaning |
| --- | --- |
| `-p, --package <name>` | Update one package instead of the whole workspace. |
| `--no-changelog` | Do not create/update changelogs. |
| `--release-date <YYYY-MM-DD>` | Set the release date in the changelog; defaults to the current UTC date. |
| `--registry-manifest-path <Cargo.toml>` | Compare against a local copy of the released project instead of the registry. Useful for offline testing and for reproducing what a release would do. |
| `--changelog-config <path>` | Path to a git-cliff config (or `GIT_CLIFF_CONFIG`). Prefer `[changelog]` in `release-plz.toml`. |
| `--allow-dirty` | Allow uncommitted changes in the working directory; they become part of the update. |
| `-u, --update-deps` | Run `cargo update` for all dependencies; without it, only workspace packages are updated. |

A real workspace example: a `feat:` commit on `rp-a` with `rp-b` depending on it produced `rp-a: 0.1.0 -> 0.1.1`, `rp-b: 0.1.0 -> 0.1.1`, a new `CHANGELOG.md` entry under `### Added`, and an updated local dependency requirement. On `0.x`, `feat:` bumps patch by default; see `changelog-and-semver.md`.

## release-pr

Computes the next release and opens or updates the release PR. It runs the same analysis as `update`, then commits the changes to a release branch and opens a PR/MR.

```bash
release-plz release-pr --git-token "$GITHUB_TOKEN"
release-plz release-pr --forge gitlab --git-token "$RELEASE_PLZ_TOKEN"
release-plz release-pr -o json
```

Behavior:

- If a release PR is already open, release-plz updates it. It overwrites the branch and force-pushes; commits made by bots are discarded. If the PR has non-bot commits beyond the first, release-plz closes it and opens a new one to preserve the maintainer's history.
- `-p <package>` does not open a separate PR per package; it overrides the existing release PR with that package's changes.
- On GitHub, the commit is created through the GraphQL API, so it can be verified without a local GPG signature. On GitLab/Gitea, release-plz pushes the branch.
- The PR branch starts with `pr_branch_prefix` (default `release-plz-`). This prefix is also how `release_always = false` recognizes a release PR.

Forge tokens:

| Forge | Token | Scopes/role |
| --- | --- | --- |
| GitHub | `GITHUB_TOKEN`, PAT, or GitHub App token | Contents + Pull requests read/write; the default `GITHUB_TOKEN` cannot trigger further workflows |
| GitLab | Project/group access token | Maintainer; `api` + `write_repository` |
| Gitea | Application token | `read:repository`/`write:repository` and `read:issue`/`write:issue` |

JSON output (`-o json`):

```json
{
  "prs": [
    {
      "head_branch": "release-plz-2026-11-20T...",
      "base_branch": "main",
      "html_url": "https://github.com/owner/repo/pull/123",
      "number": 123,
      "releases": [
        { "package_name": "my-crate", "version": "1.2.3" }
      ]
    }
  ]
}
```

`prs` is empty when no release PR was opened or updated. Only one release PR is supported at a time.

## release

Publishes every package whose current `Cargo.toml` version is not in the cargo registry yet. It does **not** edit `Cargo.toml` or push commits. For each release it:

1. Creates a git tag. The name is `<package>-v<version>` in a multi-package release; the `<package>-` prefix is omitted when only one package is released.
2. Runs `cargo publish` (unless `publish = false`, `git_only = true`, or `--dry-run`).
3. Creates a GitHub/GitLab/Gitea release for the tag (unless `git_release_enable = false`).

```bash
release-plz release --dry-run
release-plz release --git-token "$GITHUB_TOKEN"
release-plz release --forge gitlab --git-token "$RELEASE_PLZ_TOKEN"
release-plz release -o json
```

Important behavior:

- The CLI `release` needs forge credentials even when `git_release_enable = false`: it queries the forge to find release PRs for `release_always = false` and to verify commits. Without a token it fails with `git release not configured. Did you specify git-token and forge?`. In GitHub Actions the action supplies `GITHUB_TOKEN`.
- `--dry-run` performs checks without uploading, but it still needs the forge token; it is not a fully offline mode.
- Tags are returned in JSON even when `git_tag_enable = false`, so you can create the tag yourself.
- If all packages are already published, the command does nothing and exits 0.

Flags:

| Flag | Meaning |
| --- | --- |
| `--dry-run` | Perform checks without uploading. |
| `--token <token>` | Registry token; overrides `CARGO_REGISTRY_TOKEN` / `CARGO_REGISTRIES_<NAME>_TOKEN`. |
| `--no-verify` | Pass `--no-verify` to `cargo publish`. |
| `--allow-dirty` | Pass `--allow-dirty` to `cargo publish`. |
| `-o json` | Print released packages, versions, tags, and PRs. |

JSON output:

```json
{
  "releases": [
    {
      "package_name": "my-crate",
      "prs": [{ "html_url": "https://github.com/owner/repo/pull/120", "number": 120 }],
      "tag": "v1.2.3",
      "version": "1.2.3"
    }
  ]
}
```

### What commit is released

For a release PR merged with the default merge strategy, release-plz checks out the latest commit of the release PR before releasing, then restores the original checkout. This avoids publishing unrelated commits that landed between the PR merge and the release job. With squash-and-merge, GitHub creates a new commit, so release-plz releases the latest main commit instead. If release PRs must release exactly one commit, use the default merge strategy; with a merge queue, be aware of the same race.

Detached HEADs and colocated Jujutsu (`jj`) repositories are supported. Before releasing with jj, create an empty working-copy change on top of the target commit so Git's HEAD points there and the working tree is clean. Signed tags require the origin remote to point at the target repository because signed tags are pushed with the git CLI.

## set-version

Manual version override for when release-plz's computed bump is not what you want (for example, a missing `feat:` prefix). It edits `Cargo.toml`, local dependency requirements, `Cargo.lock`, and the changelog.

```bash
release-plz set-version 1.2.3                 # single package, or workspace.package.version
release-plz set-version my-crate@1.2.3        # one package in a workspace
release-plz set-version a@1.2.3 b@2.0.0       # multiple packages
```

- In a workspace with `[workspace.package].version`, omitting the package name updates the workspace version and all packages inheriting `version.workspace = true`.
- Packages with their own `version` keep it unless named explicitly.
- This command sets package versions, not dependency versions.

## init

Interactive setup for a GitHub repository. It uses the `gh` GitHub CLI to store the cargo registry token and GitHub token as repository secrets, and writes the workflow/config. **Do not run it in an automated or non-interactive session**; create the config and workflow directly instead, or ask the user to run `release-plz init` in their terminal.

## generate-schema and utilities

```bash
release-plz generate-schema          # writes .schema/latest.json
release-plz generate-completions bash
release-plz check-updates
```

`generate-schema` always writes `.schema/latest.json` relative to the current directory; it does not print the schema to stdout. Add `.schema/` to `.gitignore` or move/remove it after inspection.

## Local testing with a registry manifest

`--registry-manifest-path` points at the `Cargo.toml` of the already-released version. This is the reliable way to test release-plz offline or against a synthetic previous release:

```bash
# Create a copy of the project at the last released commit
git worktree add /tmp/project-previous HEAD~1

# Compare the working tree against that copy instead of crates.io
release-plz update --registry-manifest-path /tmp/project-previous/Cargo.toml --allow-dirty
```

This is useful in CI sandboxes and for understanding what a bump would do before publishing. It also avoids the registry/network comparison; `update` still edits files.

## Exit codes and no-op behavior

| Situation | Exit code / behavior |
| --- | --- |
| `update` with no changes | 0; no files changed |
| `release-pr` with no release-worthy changes | 0; `prs` is empty in JSON output |
| `release` with all versions already published | 0; `releases` is empty in JSON output |
| `release --dry-run` with a valid lifecycle | 0; no upload |
| Missing forge token for `release` | Non-zero (`git release not configured...`) |
| `git_only = true` + `publish = true` | Config error |
| `cargo publish` failure / timeout | Non-zero; see `publish_timeout` and `-vv` logs |
| Missing `cargo-semver-checks` in a manual CLI job | Non-zero or skipped check depending on configuration; install it or set `semver_check = false` intentionally |

For CI, remember that `release` is not idempotent in the "no-op" sense: it is idempotent because it checks the registry, but it may still contact the forge. Treat a non-zero exit as a real failure unless the log says the release is already published.
