# `release-plz.toml` configuration reference

release-plz works without a config file. When you need to change behavior, it reads the first existing file among:

1. `release-plz.toml`
2. `.release-plz.toml`
3. `.config/release-plz.toml`

Pass an explicit path with `--config <path>` on any command, or the `config` input on the GitHub Action. `release-plz generate-schema` writes the current JSON schema to `.schema/latest.json`; use it to validate an unfamiliar field, and do not commit the generated file.

The file has three sections:

- `[workspace]` — defaults applied to every package.
- `[[package]]` — per-package overrides, selected by `name`.
- `[changelog]` — changelog rendering, using the subset of git-cliff options that release-plz supports.

## Contents

- [Workspace fields](#workspace-fields)
- [Package overrides](#package-overrides)
- [Changelog fields](#changelog-fields)
- [Release strategies](#release-strategies)
- [Common recipes](#common-recipes)
- [Field interactions and pitfalls](#field-interactions-and-pitfalls)

## Workspace fields

```toml
[workspace]
allow_dirty = false
changelog_config = "config/git-cliff.toml"  # deprecated; prefer [changelog]
changelog_path = "CHANGELOG.md"
changelog_update = true
custom_major_increment_regex = "^breaking"
custom_minor_increment_regex = "^minor|^enhancement"
dependencies_update = false
features_always_increment_minor = false
git_only = false
git_release_enable = true
git_release_name = "{{ package }}-v{{ version }}"
git_release_body = "{{ changelog }}"
git_release_type = "prod"
git_release_draft = false
git_release_latest = true
git_release_generate_notes = false
git_tag_enable = true
git_tag_name = "{{ package }}-v{{ version }}"
max_analyze_commits = 1000
# pr_body = "..."          # omit to use the built-in PR body
pr_branch_prefix = "release-plz-"
pr_draft = false
pr_labels = []
# pr_name = "..."          # omit to use the built-in PR title
publish = true
publish_allow_dirty = false
publish_no_verify = false
publish_features = []
publish_all_features = false
publish_timeout = "30m"
release = true
release_always = true
release_commits = ""
repo_url = "https://github.com/owner/repo"
semver_check = true
```

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `allow_dirty` | bool | `false` | Allow `update` / `release-pr` to run with uncommitted changes; those changes become part of the update. This is different from the `release` command's `--allow-dirty` flag and from `publish_allow_dirty`. |
| `changelog_config` | path | none | Path to a git-cliff config. **Deprecated**; use `[changelog]` instead. |
| `changelog_path` | path | `CHANGELOG.md` next to the package's `Cargo.toml` | Shared or custom changelog path, relative to the root `Cargo.toml`. |
| `changelog_update` | bool | `true` | Update changelogs during `update` / `release-pr`. |
| `custom_major_increment_regex` | regex | none | Commit type or full message that forces a major bump, including on `0.x`. |
| `custom_minor_increment_regex` | regex | none | Commit type or full message that forces a minor bump. |
| `dependencies_update` | bool | `false` | `true` runs `cargo update` for all dependencies; `false` updates only workspace members. |
| `features_always_increment_minor` | bool | `false` | Makes `feat:` bump minor on `0.x`; violates Cargo SemVer for `0.x`. |
| `git_only` | bool | `false` | Determine versions from git tags instead of the registry; skips `cargo publish`. Cannot be combined with `publish = true`. |
| `git_release_enable` | bool | `true` | Create a GitHub/GitLab/Gitea release for the tag. |
| `git_release_name` | Tera template | `{{ package }}-v{{ version }}` for multi-package workspaces; `v{{ version }}` for a single public package | Release title. |
| `git_release_body` | Tera template | `{{ changelog }}` | Release body. Supports `{{ changelog }}`, `{{ package }}`, `{{ version }}`, and `{{ remote.contributors }}`. |
| `git_release_type` | `prod` / `pre` / `auto` | `prod` | Mark the release as production or pre-release. `auto` detects a SemVer pre-release suffix (GitHub/Gitea only). |
| `git_release_draft` | bool | `false` | Create the forge release as a draft (GitHub/Gitea only). |
| `git_release_latest` | bool | `true` | Mark the release as latest (GitHub only). Drafts and prereleases cannot be latest. |
| `git_release_generate_notes` | bool | `false` | Use the forge's server-side release notes (GitHub only). When true, release-plz's body is appended before the generated notes; set `git_release_body = ""` to use only generated notes. |
| `git_tag_enable` | bool | `true` | Create a git tag. Without a tag there is no forge release. |
| `git_tag_name` | Tera template | same default as `git_release_name` | Tag name. |
| `max_analyze_commits` | int | `1000` | Commit-analysis limit for packages that have never been published. Ignored once a registry release exists. |
| `pr_body` | Tera template | built-in summary + changelog | Release PR body. The default includes package versions, breaking changes, and a collapsible changelog. |
| `pr_branch_prefix` | string | `release-plz-` | Branch prefix for release PRs. release-plz only manages branches with this prefix. Changing it orphans the current release PR. |
| `pr_draft` | bool | `false` | Open the release PR as a draft. |
| `pr_labels` | array | `[]` | Labels for the release PR. |
| `pr_name` | Tera template | built-in | Release PR title. See the template notes below. |
| `publish` | bool | `true` | Run `cargo publish`. `false` still creates tags/releases and still consults the registry unless `git_only = true`. |
| `publish_allow_dirty` | bool | `false` | Add `--allow-dirty` to `cargo publish`. |
| `publish_no_verify` | bool | `false` | Add `--no-verify` to `cargo publish`. |
| `publish_features` | array | `[]` | Add `--features=a,b` to `cargo publish` verification. |
| `publish_all_features` | bool | `false` | Add `--all-features` to `cargo publish`. |
| `publish_timeout` | duration | `30m` | Timeout for publishing and checking registry state (`30s`, `10m`, `1h`). |
| `release` | bool | `true` | `false` excludes packages from `update`, `release-pr`, and `release`. Set at workspace level and override per package to opt in. |
| `release_always` | bool | `true` | `true` tries to release on every run; `false` releases only commits associated with a release PR. GitHub only. |
| `release_commits` | regex | none | Only bump a package when at least one commit matches. Filtered commits still appear in the changelog. |
| `repo_url` | URL | origin remote | Repository URL used for changelog links and forge operations. |
| `semver_check` | bool | `true` | Run `cargo-semver-checks` for library crates. |

### PR name template

The default title depends on the release shape:

- `chore({{ package }}): release v{{ version }}` when one package in a multi-package workspace changes.
- `chore: release v{{ version }}` when the project has one package, or several packages share the next version.
- `chore: release` when several packages have different next versions.

Custom templates must guard optional variables:

```toml
[workspace]
pr_name = "release{% if package and version %} {{ package }} v{{ version }}{% endif %}"
```

`{{ package }}` is only set when a single package is released; `{{ version }}` is set for a single package or when multiple packages share one version.

### PR body template

The default body already contains the release summary, per-package changelogs, breaking-change sections, and a release-plz footer. If the body exceeds GitHub's 65,536-character limit, the changelog is omitted. Custom bodies receive:

| Variable | Meaning |
| --- | --- |
| `{{ releases }}` | Array of release entries. |
| `{{ release.title }}` | Changelog title with a release-tag diff link (optional). |
| `{{ release.package }}` / `{{ release.previous_version }}` / `{{ release.next_version }}` | Package and versions. |
| `{{ release.changelog }}` | Rendered changelog (optional). |
| `{{ release.semver_check }}` | `compatible`, `incompatible`, or `skipped`. |
| `{{ release.breaking_changes }}` | Breaking-change summary (optional). |

Use `{% if ... %}` for optional fields; release-plz fails if a template references an unset variable directly.

## Package overrides

```toml
[[package]]
name = "my-crate"
changelog_include = ["shared-core"]
changelog_path = "docs/CHANGELOG.md"
changelog_update = true
semver_check = false
publish = true
publish_features = ["serde"]
version_group = "core-group"
release = true
```

Only `name` is required. Most workspace fields can be overridden: `changelog_path`, `changelog_update`, `custom_major_increment_regex`, `custom_minor_increment_regex`, `features_always_increment_minor`, `git_only`, `git_release_*`, `git_tag_*`, `publish*`, `release`, and `semver_check`.

Package-only fields:

| Field | Meaning |
| --- | --- |
| `changelog_include` | Include commits from other packages in this package's changelog. The included commits release this package too, but bump it at most by a patch (or a prerelease increment). |
| `version_group` | Packages with the same value get the same next version, chosen as the highest next version in the group. Only packages with changes participate; a dependent package can pull the group forward. |

`version_group` is useful for a subset of a workspace that must stay in lockstep without forcing the entire workspace to share `[workspace.package].version`.

## Changelog fields

```toml
[changelog]
header = """# Changelog

All notable changes to this project will be documented in this file.
"""
body = """## [{{ version }}]{% if release_link %}({{ release_link }}){% endif %} - {{ timestamp | date(format="%Y-%m-%d") }}
{% for group, commits in commits | group_by(attribute="group") %}
### {{ group | upper_first }}
{% for commit in commits %}
- {% if commit.breaking %}[**breaking**] {% endif %}{{ commit.message }}
{% endfor %}
{% endfor %}
"""
trim = true
protect_breaking_commits = true
tag_pattern = "v[0-9].*"
sort_commits = "newest"
commit_preprocessors = [
  { pattern = "\\(#([0-9]+)\\)", replace = "([#${1}](https://github.com/owner/repo/pull/${1}))" },
]
postprocessors = [
  { pattern = ".*", replace = "dprint fmt --stdin md" },
]
link_parsers = [
  { pattern = "RFC(\\d+)", text = "ietf-rfc$1", href = "https://datatracker.ietf.org/doc/html/rfc$1" },
]
commit_parsers = [
  { message = "^feat", group = "Added" },
  { message = "^fix", group = "Fixed" },
  { message = "^.*", group = "Changed" },
]
```

| Field | Default | Meaning |
| --- | --- | --- |
| `header` | Keep a Changelog header | Text at the top of the changelog. |
| `body` | Keep a Changelog release template | Template for one release. |
| `trim` | `true` | Trim leading/trailing whitespace from the body. |
| `protect_breaking_commits` | `false` | Never skip a breaking commit, even if a parser says `skip = true`. |
| `tag_pattern` | all tags | Regex selecting tags included in the changelog. |
| `sort_commits` | git-cliff default | `oldest` or `newest`. |
| `commit_preprocessors` | link `(#123)` to the PR URL | Regex replacements applied before parsing. Supports `replace_command` with `$COMMIT_SHA`. |
| `postprocessors` | none | Regex replacements applied to the rendered changelog, e.g. run a formatter. |
| `commit_parsers` | feat→added, changed→changed, deprecated→deprecated, fix→fixed, security→security, others→other | Ordered parsers that assign `group`, `scope`, `default_scope`, `skip`, `continue`, or `field`/`pattern`, or match `body`, `footer`, or `sha`. |
| `link_parsers` | none | Extract external references and turn them into links (`pattern`, optional `text`, required `href`). |

Changelog templates use git-cliff's Tera 1 syntax. Release name/body, tag name, PR name/body templates use Tera 2. These are different template engines with different capabilities; do not copy a git-cliff template into `git_release_body` or vice versa. Full changelog examples: `changelog-and-semver.md`.

## Release strategies

### Default: release on every main commit

No config required. Both CI jobs run on each push. A version already in `Cargo.toml` but absent from the registry is published; otherwise nothing happens.

### Release only when the release PR is merged

```toml
[workspace]
release_always = false
pr_branch_prefix = "release-plz-"
```

`release` checks whether the latest commit belongs to a PR whose branch starts with `pr_branch_prefix`. It is GitHub-only. Use the default merge strategy for release PRs: with squash merges, GitHub rewrites the commit and release-plz may not find the PR association, so the release is skipped or a race can publish the wrong commit. The same branch-prefix rule applies if you create a release-triggering PR manually.

### Git-only: tags without a registry

```toml
[workspace]
git_only = true
publish = false
```

Versions come from matching git tags rather than the registry. `cargo publish` is skipped, and packages with `publish = false` in `Cargo.toml` are still tagged. Do not set `publish = true` with `git_only = true`; that combination is rejected.

### Private registry

```toml
[workspace]
publish = true
```

```yaml
# GitHub Action
with:
  registry: my-registry
env:
  CARGO_REGISTRIES_MY_REGISTRY_TOKEN: ${{ secrets.MY_REGISTRY_TOKEN }}
```

The registry name must exist in the project's cargo config. The token variable is `CARGO_REGISTRIES_<NAME>_TOKEN`, uppercase. The action's `token` input / CLI `--token` overrides the environment variable.

### Tag/release without publishing

```toml
[[package]]
name = "private-tool"
publish = false
```

`publish = false` skips `cargo publish` but still creates tags/releases. release-plz still consults the registry for the previous version, so use `git_only = true` if the package will never be published.

### Subset of packages in lockstep

```toml
[[package]]
name = "core-a"
version_group = "core"

[[package]]
name = "core-b"
version_group = "core"
```

The group gets the highest next version among changed members. If a dependent package changes because a group member changed, it can pull the group forward.

## Common recipes

### Only release on features or documentation

```toml
[workspace]
release_commits = "^(feat|docs)[(:]"
```

The package is bumped only when at least one commit matches. Filtered commits still appear in the changelog; use `commit_parsers` to exclude them from the changelog.

### Custom commit prefixes

```toml
[workspace]
custom_major_increment_regex = "^breaking"
custom_minor_increment_regex = "^minor|^enhancement"
```

Conventional commits use the type before `:`. Non-conventional messages match the whole message.

### Workspace version

If the workspace uses `[workspace.package].version`, packages inheriting `version.workspace = true` are updated together by `set-version <version>` and by release-plz's normal update. Packages with their own version keep it.

### Shared changelog

```toml
[workspace]
changelog_path = "CHANGELOG.md"
changelog_update = true
```

All packages write to one changelog. Use `changelog_include` if a package's changelog should also contain commits from another package.

### PR labels and drafts

```toml
[workspace]
pr_labels = ["release"]
pr_draft = true
```

Labels must already exist on the forge, or the token must be allowed to create them.

### Release body with contributors

```toml
[workspace]
git_release_body = """
{{ changelog }}
{% if remote.contributors %}
### Contributors
{% for contributor in remote.contributors %}
- @{{ contributor.username }}
{% endfor %}
{% endif %}
"""
```

Contributor usernames come from PR numbers in commit messages, e.g. `(#123)` or a full PR link.

## Field interactions and pitfalls

| Combination / field | Behavior |
| --- | --- |
| `release = false` at workspace, `true` on a package | Only the opted-in packages are processed. |
| `publish = false` + `git_only = false` | Tags/releases are still created, but the registry is still consulted; the package must have a published version or be treated as a new package. |
| `git_only = true` + `publish = true` | Error; git-only mode skips publishing. |
| `git_tag_enable = false` | No tag, therefore no forge release even if `git_release_enable = true`. |
| `release_always = false` on GitLab | Not supported by the GitHub-only PR-association check; use `true` or the GitLab component's default behavior. |
| `allow_dirty = true` | Affects `update` and `release-pr`; `release` has its own `--allow-dirty` flag. |
| `publish_allow_dirty = true` | Adds `--allow-dirty` to `cargo publish`; unrelated to `allow_dirty` and the CLI flag. |
| `features_always_increment_minor = true` | `feat:` bumps `0.x` minor; the docs warn this violates Cargo SemVer. |
| `changelog_config` | Deprecated; `[changelog]` is the supported path. |
| `pr_branch_prefix` changed | release-plz no longer recognizes the old release PR; close it first. |
| `max_analyze_commits` | Only applies before the first registry release; raise it for a very long initial history. |
| `git_release_generate_notes = true` | GitHub only; release-plz's body is prepended to generated notes. Set `git_release_body = ""` for generated notes only. |
| `git_release_latest = true` with `git_release_draft` / `git_release_type = "pre"` | Drafts and prereleases cannot be latest; the forge may reject or ignore it. |
