# Troubleshooting release-plz

Start with the facts:

```bash
release-plz --version
cargo --version
cat Cargo.toml
cat release-plz.toml 2>/dev/null
git status --short
git tag --sort=-creatordate | head -10
git log --oneline -20
```

Then reproduce locally where possible:

```bash
release-plz update --allow-dirty -vv
# or, against a copy of the last release:
release-plz update --registry-manifest-path /path/to/previous/Cargo.toml --allow-dirty -vv
```

`-vv` enables DEBUG logs; `-vvv` enables TRACE. `RELEASE_PLZ_LOG=DEBUG` has the same effect without verbosity flags.

## Contents

- [Release PR problems](#release-pr-problems)
- [Version bump problems](#version-bump-problems)
- [Release problems](#release-problems)
- [GitHub Actions problems](#github-actions-problems)
- [GitLab CI problems](#gitlab-ci-problems)
- [Registry and publishing problems](#registry-and-publishing-problems)
- [Changelog problems](#changelog-problems)
- [Migration notes](#migration-notes)
- [Debug checklist](#debug-checklist)

## Release PR problems

### The release PR is not created

Check, in order:

1. **Are there release-worthy changes?** `release-plz update` locally and inspect `git diff`. If no package version/changelog changes, there is nothing to PR.
2. **Is `fetch-depth: 0` set?** A shallow checkout prevents release-plz from finding the last released commit and analyzing the diff. This is the most common GitHub Actions cause.
3. **Does the job have `pull-requests: write` and `contents: write`?**
4. **Is the repository setting "Allow GitHub Actions to create and approve pull requests" enabled?** This is separate from workflow permissions.
5. **Is `release = false` set at workspace level?** If so, opt packages back in with `release = true`.
6. **Is `release_commits` filtering everything?** Check the regex; it must match at least one commit in the package diff.
7. **Is the package excluded by `publish = false`?** `publish = false` does not prevent the release PR; it only skips `cargo publish`. `release = false` does.
8. **Are the changed files part of the package?** Run `cargo package --list`; a commit that only touches files outside the package's packaged file set may not count.

### The release PR is not updated; a new one is opened

If the existing release PR has non-bot commits beyond the first, release-plz closes it and opens a new one to preserve the maintainer's history. If the PR cannot be updated (force-push conflict), it is also closed. Review the old PR, then let the new one proceed.

### The branch prefix changed and the old PR is orphaned

release-plz only manages branches starting with `pr_branch_prefix` (default `release-plz-`). Changing the prefix makes the old PR invisible. Close the old PR and let a new one be created.

### `release-pr` says no changes but the release PR exists

The action's `prs_created` output is `"false"` when the PR already exists and was updated rather than newly created. The PR is still there; check `pr` / `prs` output and the branch.

## Version bump problems

### `feat:` only bumped the patch version on `0.x`

This is intentional Cargo SemVer behavior: `feat:` bumps patch on `0.x`; minor is reserved for breaking changes. Options:

- Keep the default and accept `0.1.0 -> 0.1.1`.
- Set `features_always_increment_minor = true` to make `feat:` bump minor (`0.1.0 -> 0.2.0`). The docs warn this violates Cargo SemVer for `0.x`.
- Use `custom_minor_increment_regex` for non-conventional prefixes.

### The version bumped more than expected

- `cargo-semver-checks` detected an API break and bumped major.
- A `breaking change` footer or `!` in a conventional commit triggered a major bump.
- A local dependency changed, causing a patch bump in a dependent package.
- `version_group` assigned the highest next version in the group.
- `dependencies_update = true` updated the lockfile and changed the diff.

Inspect the release PR body: it lists each package's previous/next version and the semver-check result. Use `release-plz set-version` to override before merging.

### The release PR opens too often

release-plz opens a PR when any file packaged by the crate changes. Run `cargo package --list` to see the packaged files, then use `include` / `exclude` in `Cargo.toml` to stop irrelevant files (docs, fixtures, generated assets) from triggering releases. `release_commits` can also gate on commit types, but the commits still appear in the changelog.

### A package gets bumped although it had no changes

A dependency change is a change to the dependent package. If the dependent's `Cargo.toml` range or lockfile changes, release-plz bumps it (usually patch). `changelog_include` can include the dependency's commits in the dependent's changelog.

## Release problems

### `release` does nothing after merging the release PR

- **`release_always = false`**: the latest commit must be associated with a PR whose branch starts with `pr_branch_prefix`. Squash merges rewrite the commit; use the default merge strategy. A merge queue can also race. Check the release job log for "no release PR" or similar.
- **The version is already published**: release-plz checks the registry; if the version exists, it does nothing. Confirm with `cargo search` or the registry UI.
- **`release = false`** on the package.
- **`git_only = true`**: versions come from tags, not the registry. If the tag exists, there is nothing to release.

### `release` hangs

Set a lower `publish_timeout` (default `30m`) to get a bounded failure:

```toml
[workspace]
publish_timeout = "5m"
```

Then check registry connectivity, authentication, and `cargo publish` manually. Use `-vv` to see whether the hang is in `cargo publish`, the registry check, or the forge API.

### `git release not configured. Did you specify git-token and forge?`

The CLI `release` needs a forge token even when forge releases are disabled; it queries the forge to inspect release PRs. Pass `--git-token` (or set `GIT_TOKEN`) and `--forge github|gitlab|gitea`. In GitHub Actions the action supplies `GITHUB_TOKEN`.

### The release job is canceled

Do not put a canceling `concurrency` group on the release job. A new push can cancel a pending release, and that version may not be published until another change lands. Use concurrency only on the PR job with `cancel-in-progress: false`.

### The released commit is not the release PR commit

With squash-and-merge, GitHub creates a new commit, so release-plz releases the latest main commit. With the default merge strategy, release-plz checks out the latest release-PR commit and releases that. Use the default merge strategy when exact commit selection matters (merge queues, multiple maintainers).

### Tags are missing on the remote

`release` creates tags locally and the action pushes them through the API. If you use `--no-git-tag` or `git_tag_enable = false`, no tag is pushed. If you run the CLI locally, push tags yourself. Signed tags need `persist-credentials: true` and an origin that points at the target repository.

## GitHub Actions problems

### The action cannot create or update the PR

- `pull-requests: write` missing on the PR job.
- Repository setting "Allow GitHub Actions to create and approve pull requests" disabled.
- `GITHUB_TOKEN` is read-only due to repository defaults or fork PR context.

### Checks do not run on the release PR

The default `GITHUB_TOKEN` cannot trigger workflows. Use a fine-grained PAT or a GitHub App token. This also applies to workflows triggered by `release: published` or tag pushes.

### `fetch-depth: 0` is missing

Symptom: empty changelog, wrong version, "cannot find the last release", or no release PR. Add `fetch-depth: 0` to `actions/checkout`.

### Authentication fails when signing tags or pushing

Set `persist-credentials: true` and pass the token to checkout:

```yaml
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          fetch-depth: 0
          persist-credentials: true
          token: ${{ secrets.RELEASE_PLZ_TOKEN }}
```

release-plz falls back to the git CLI for signed tags because the GitHub API cannot sign them.

### The action starts failing after a release

Pin the action to a full commit SHA and pin the CLI with the `version` input. A floating `@v0.5` picks up new action releases automatically.

### Trusted publishing fails

- `id-token: write` missing on the release job.
- crates.io trusted publishing not configured for every crate.
- `CARGO_REGISTRY_TOKEN` still set; remove it entirely.
- The crate has never been published. crates.io does not allow trusted publishing for a brand-new crate; publish once manually.

## GitLab CI problems

### The component job cannot push or open an MR

The token needs Maintainer role and both `api` and `write_repository` scopes. Check that it is stored as a **masked** `RELEASE_PLZ_TOKEN`. If the variable is protected, the default branch must be protected too.

### Commit author is empty or the job fails creating the commit

Set `RELEASE_PLZ_BOT_EMAIL` to the token's bot user email. The format is `<username>@noreply.<gitlab-host>` (for gitlab.com: `<username>@noreply.gitlab.com`).

### The component says the branch moved

The component checks `git rev-parse HEAD == $CI_COMMIT_SHA`. If the default branch changed while the job was starting, it exits with a message to restart CI on the latest commit. Re-run the pipeline on the latest commit; do not bypass the check.

### The component does not run

- It runs only on the default branch, not tags.
- It uses the `deploy` stage by default; pass `stage:` if your pipeline lacks it.
- Branch pipelines must be enabled.

### Shallow clone / missing tags

Set `GIT_DEPTH: "0"` in the project or job variables. A shallow clone can make release-plz miss the previous release.

### The component image lacks Cargo

The default image is `rust`. If you override `image`, it must contain `cargo` and `git`.

### Submodules on another forge fail

Set `git_submodule_recursive: true` and provide `git_credentials` for the other host. The component writes them to `~/.git-credentials` and updates submodules before running release-plz.

## Registry and publishing problems

### `cargo publish` authentication failure

- crates.io: `CARGO_REGISTRY_TOKEN` missing, expired, or lacking `publish-new` / `publish-update`.
- Private registry: the variable is `CARGO_REGISTRIES_<NAME>_TOKEN`; `<NAME>` must match the registry name in the cargo config.
- The action's `token` input / CLI `--token` overrides the environment variable; make sure it is not empty.

### A new crate cannot be published with trusted publishing

crates.io limitation: the first publish of a new crate must use a registry token. Publish it manually, then enable trusted publishing.

### `publish = false` but release-plz still asks the registry

`publish = false` skips `cargo publish` but release-plz still checks the registry to determine the previous version. If the package will never be published, use `git_only = true` instead. `git_only = true` + `publish = true` is a configuration error.

### npm/pnpm-style side effects?

release-plz is Rust-only; it runs `cargo publish`, `cargo update`, `cargo-semver-checks`, and git/forge operations. It does not touch Node package managers. If a CI job mixes release-plz with npm publishing, keep the steps separate.

## Changelog problems

| Symptom | Cause / fix |
| --- | --- |
| No changelog file | `changelog_update = false`, or the package is excluded by `release = false`. |
| Changelog has no release link | No origin remote; set `repo_url` or run in a checkout with an origin. |
| Commit missing | `commit_parsers` skipped it, or the commit did not touch the package; check `changelog_include`. |
| Breaking commit missing | A parser skipped it; set `protect_breaking_commits = true`. |
| Wrong section headings | Customize `commit_parsers`; defaults are Added/Changed/Deprecated/Fixed/Security/Other. |
| Template renders incorrectly | `[changelog]` uses git-cliff/Tera 1; `git_release_*` and `pr_*` use Tera 2. Check which section the template belongs to. |
| Previous release missing | `tag_pattern` excludes the tag; widen it or remove it. |

More changelog details: `changelog-and-semver.md`.

## Migration notes

### From `cargo-release`

- cargo-release edits versions/tags locally; release-plz separates "prepare release PR" from "publish".
- Map release hooks/checks into CI before the release job or into PR checks.
- `cargo-release` config keys do not map 1:1; start with defaults and add only the fields you need.
- Use `release = false` for packages cargo-release handled manually, then opt in deliberately.

### From `semantic-release`

- release-plz is Cargo-aware: it reads `Cargo.toml`, workspace members, registry versions, and `cargo package` contents.
- Tag format is `<package>-v<version>` (or `v<version>` for a single released package), not necessarily `v<version>`.
- Changelog defaults are Keep a Changelog via git-cliff, not the semantic-release template.
- `release_always` controls whether every main commit releases or only release-PR merges.

### From manual releases

1. Run `release-plz update` locally and inspect the diff.
2. Add `release-plz.toml` only for behavior you need (private crates, version groups, changelog path).
3. Add the GitHub/GitLab CI jobs.
4. If you want to keep manual publishing, set `publish = false`; if you want tag-only releases, set `git_only = true`.
5. Do a `release --dry-run` before enabling real publishing.

## Debug checklist

```bash
release-plz --version
release-plz -vv update --allow-dirty
git diff
git tag --sort=-creatordate | head
git log --oneline --decorate -20
cargo package --list
cargo metadata --no-deps --format-version 1 | jq '.packages[] | {name, version, publish}'
```

For CI:

- Check the action/component version and the CLI `version` input.
- Check `fetch-depth` / `GIT_DEPTH`.
- Check job permissions and repository Actions settings.
- Check that the token is the expected one (default `GITHUB_TOKEN` vs PAT/App).
- Check the release PR branch prefix and merge strategy when `release_always = false`.
- Check the registry token variable name and scope.
- Re-run with `-vv` or `RELEASE_PLZ_LOG=DEBUG` and read the last failing step, not the first warning.
