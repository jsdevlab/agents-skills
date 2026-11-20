# Changelog and semver checks

release-plz uses [git-cliff](https://git-cliff.org) as a library to build changelogs, and [cargo-semver-checks](https://github.com/obi1kenobi/cargo-semver-checks) to detect API-breaking changes in library crates. This page covers the behavior and the `[changelog]` options that matter in practice. The complete git-cliff template language is linked at the end rather than reproduced.

## Contents

- [Default changelog](#default-changelog)
- [How commits are grouped](#how-commits-are-grouped)
- [Changelog options](#changelog-options)
- [Preprocessors, postprocessors, and link parsers](#preprocessors-postprocessors-and-link-parsers)
- [Semver checks](#semver-checks)
- [Version bumps and `0.x`](#version-bumps-and-0x)
- [Common changelog problems](#common-changelog-problems)

## Default changelog

Each package gets a `CHANGELOG.md` next to its `Cargo.toml` by default. The default format is Keep a Changelog:

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.2.3] - 2026-11-20

### Added

- add the stream option

## [1.2.2] - 2026-10-01

### Fixed

- handle empty input
```

A real release-plz `update` run on a workspace produced:

```markdown
## [0.1.1] - 2026-11-20

### Added

- add a2
```

The `[Unreleased]` section is kept for future changes. The release date defaults to the current UTC date; override it with `--release-date YYYY-MM-DD` on `update` / `release-pr`.

Change the path with `changelog_path` in `[workspace]` or `[[package]]`. It is relative to the root `Cargo.toml`; in GitHub Actions, the repository root. Set `changelog_update = false` to skip changelog edits entirely.

## How commits are grouped

Default `commit_parsers`:

```toml
[changelog]
commit_parsers = [
  { message = "^feat", group = "added" },
  { message = "^changed", group = "changed" },
  { message = "^deprecated", group = "deprecated" },
  { message = "^fix", group = "fixed" },
  { message = "^security", group = "security" },
  { message = "^.*", group = "other" },
]
```

Parsers are evaluated in order and stop at the first match unless a parser sets `continue = true`. Later parsers can override fields set by earlier ones; fields they do not set are preserved.

A commit's changelog fate:

- **group** — section name. The default groups render as Added, Changed, Deprecated, Fixed, Security, Other.
- **skip = true** — omit the commit. `protect_breaking_commits = true` overrides `skip` for breaking commits.
- **scope / default_scope** — set or override the scope shown next to the message.
- **continue = true** — keep applying later parsers; useful for composing footer-based scope and message-based group.
- **body / footer / sha / field + pattern** — match on commit body, footer, SHA, or an arbitrary field.

Examples:

```toml
[changelog]
commit_parsers = [
  { body = ".*security", group = "Security" },
  { footer = "^changelog: ?ignore", skip = true },
  { message = '^fix\\((.*)\\)', group = 'Fix (${1})' },
  { message = "^refactor\\(clippy\\)", skip = true },
  { message = "^doc", group = "Documentation", default_scope = "other" },
  { sha = "f6f2472bdf0bbb5f9fcaf2d72c1fa9f98f772bb2", skip = true },
]
```

A scope-then-group combination:

```toml
[changelog]
commit_parsers = [
  { footer = "^Component: ?Billing$", scope = "billing", continue = true },
  { message = "^feat", group = "Features" },
  { message = "^fix", group = "Fixes" },
]
```

A `feat: add invoices` commit with `Component: Billing` gets the `billing` scope and the `Features` group.

`release_commits` is different: it filters **version bumps**, not changelog entries. Filtered commits still appear in the changelog; use `commit_parsers` with `skip = true` to remove them from the changelog as well.

## Changelog options

```toml
[changelog]
header = "# Changelog\n"
body = "..."
trim = true
protect_breaking_commits = true
tag_pattern = "v[0-9].*"
sort_commits = "newest"
```

| Option | Default | Effect |
| --- | --- | --- |
| `header` | Keep a Changelog header | Top of the file. |
| `body` | Keep a Changelog release template | One release entry. |
| `trim` | `true` | Trim leading/trailing whitespace in the rendered body. |
| `protect_breaking_commits` | `false` | Never skip a breaking commit, even if a parser says `skip = true`. |
| `tag_pattern` | all tags | Only tags matching this regex are included in the changelog. |
| `sort_commits` | git-cliff default | `oldest` or `newest` within each section. |

The body template receives variables such as `version`, `release_link`, `timestamp`, and `commits`. Commit fields include `message`, `scope`, `breaking`, and `links`. A minimal custom body:

```toml
[changelog]
body = """## [{{ version }}]{% if release_link %}({{ release_link }}){% endif %} - {{ timestamp | date(format="%Y-%m-%d") }}
{% for group, commits in commits | group_by(attribute="group") %}
### {{ group | upper_first }}
{% for commit in commits %}
- {% if commit.breaking %}[**breaking**] {% endif %}{{ commit.message }}
{% endfor %}
{% endfor %}
"""
```

`release_link` is populated from the repository URL and tag. If the project has no origin, release-plz warns and the changelog has no release link.

## Preprocessors, postprocessors, and link parsers

```toml
[changelog]
commit_preprocessors = [
  # Replace "(#123)" with a PR link. This is the default behavior.
  { pattern = "\\(#([0-9]+)\\)", replace = "([#${1}](https://github.com/owner/repo/pull/${1}))" },
  # Remove an issue number.
  { pattern = "\\(#([0-9]+)\\)", replace = "" },
  # Use an external command. $COMMIT_SHA is set for the command.
  { pattern = ".*", replace_command = "pandoc -t commonmark" },
]

postprocessors = [
  # Format the rendered changelog.
  { pattern = ".*", replace = "dprint fmt --stdin md" },
]

link_parsers = [
  { pattern = "RFC(\\d+)", text = "ietf-rfc$1", href = "https://datatracker.ietf.org/doc/html/rfc$1" },
]
```

- Preprocessors run on commit messages before parsing/grouping.
- Postprocessors run on the rendered changelog.
- `replace` supports git-cliff's `$1` / `${1}` capture references.
- `replace_command` runs an OS command; `$COMMIT_SHA` is available.
- Link parsers extract references and expose them as `commit.links` in the body template.

These examples are not a full git-cliff tutorial. For the complete template language, see the [git-cliff documentation](https://git-cliff.org/docs/).

Note the template engines: `[changelog]` templates are rendered by git-cliff (Tera 1). `git_release_name`, `git_release_body`, `git_tag_name`, `pr_name`, and `pr_body` are rendered by release-plz (Tera 2). Do not assume a template that works in one works in the other.

## Semver checks

release-plz runs `cargo-semver-checks` for library crates. A crate counts as a library when it has a `[lib]` target whose `crate-type` includes `lib`, `rlib`, or `dylib`.

- Successful check: the release PR shows `(✓ API compatible changes)`.
- Breaking change: `(⚠ API breaking changes)`, with a report; release-plz bumps the major version.
- Skipped: no library or no previously released version; nothing is shown.

`cargo-semver-checks` does not catch every semver violation. Treat it as a safety net, not a replacement for reviewing public API changes. Disable it with `semver_check = false` at workspace or package level, or install it in manual CLI/CI jobs (the GitHub Action installs it automatically).

## Version bumps and `0.x`

release-plz follows Cargo SemVer for `0.x`:

- `fix:` → patch (`0.1.0 -> 0.1.1`).
- `feat:` → patch by default (`0.1.0 -> 0.1.1`), because `0.x` minor is reserved for breaking changes.
- Breaking API change detected by `cargo-semver-checks` → major (`0.4.21 -> 0.5.0` in the docs example; `0.x -> 1.0.0` for a true major? The exact bump depends on cargo-semver-checks and the next-version logic; review the PR).
- `features_always_increment_minor = true` → `feat:` bumps minor on `0.x` (`0.1.0 -> 0.2.0`). The docs warn this violates Cargo SemVer.
- `custom_major_increment_regex` / `custom_minor_increment_regex` → force a bump from non-conventional commit prefixes.

`release_commits` can suppress a bump if no commit matches. A package still gets a patch bump when a local dependency changes, because the dependency change is part of its diff.

`set-version` is the escape hatch when the computed version is wrong; it updates `Cargo.toml`, local dependency requirements, `Cargo.lock`, and the changelog.

## Common changelog problems

| Problem | Cause | Fix |
| --- | --- | --- |
| Changelog not updated | `changelog_update = false` | Set it to `true` at workspace/package level. |
| Commit missing from changelog | A `commit_parsers` entry skipped it, or the commit did not touch the package/dependency | Check parser order and `skip`; for cross-package commits use `changelog_include`. |
| Breaking commit missing | A parser skipped it | Set `protect_breaking_commits = true`. |
| Wrong section names | Default parsers don't match your prefixes | Add `commit_parsers`; default groups are Added/Changed/Deprecated/Fixed/Security/Other. |
| Release link missing | No origin remote / `repo_url` | Set `repo_url` or run in a checkout with an origin. |
| Previous release missing from changelog | `tag_pattern` excludes its tag | Adjust `tag_pattern`; by default all tags are included. |
| Changelog at an unexpected path | Default is next to each `Cargo.toml` | Set `changelog_path` (relative to the root manifest) or a per-package override. |
| Template error in a release body | Tera 2 syntax used where Tera 1 is expected, or vice versa | `[changelog]` uses git-cliff/Tera 1; `git_release_*` / `pr_*` use Tera 2. |
| Changelog diff is huge after a path change | `changelog_path` moved the file, or all packages share one file | Decide whether a shared changelog is intended; use `changelog_include` for cross-package entries. |
