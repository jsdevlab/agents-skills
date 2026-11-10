# `.licenserc.yaml` configuration reference

`license-eye` reads `.licenserc.yaml` from the working directory by default. Use `-c/--config <path>` to point at another file. Paths in `header.paths` / `header.paths-ignore` are matched against files relative to the working directory, while `dependency.files` entries are resolved relative to the config file's directory. Run commands from the repository root so header paths line up with the file tree.

The config is YAML with two top-level sections, `header` and `dependency`. Either can be omitted, but omitting `header` disables both header checks and dependency checks in surprising ways (see [Dependency-only configurations](#dependency-only-configurations)).

## Contents

- [Config loading and defaults](#config-loading-and-defaults)
- [Header section](#header-section)
- [License fields](#license-fields)
- [Built-in SPDX header templates](#built-in-spdx-header-templates)
- [Paths and exclusions](#paths-and-exclusions)
- [License location threshold](#license-location-threshold)
- [PR comments](#pr-comments)
- [Language overrides](#language-overrides)
- [Multiple licenses in one repository](#multiple-licenses-in-one-repository)
- [Dependency section](#dependency-section)
- [Starter configurations](#starter-configurations)
- [Common configuration mistakes](#common-configuration-mistakes)

## Config loading and defaults

Two shapes are accepted:

```yaml
# Single header (V1 style)
header:
  license:
    spdx-id: Apache-2.0
```

```yaml
# One entry per license region (V2 style)
header:
  - license:
      spdx-id: Apache-2.0
    paths:
      - 'module-a/**'
  - license:
      spdx-id: MIT
    paths:
      - 'module-b/**'
```

When `.licenserc.yaml` does not exist, `license-eye` does **not** fail. It loads a built-in default:

```yaml
header:
  license:
    spdx-id: Apache-2.0
    copyright-owner: Apache Software Foundation
  paths-ignore:
    - '**/*.md'
    - '**/*.json'
    - '**/*.txt'
    - '.gitignore'
    - '.gitmodules'
    - 'LICENSE'
    - 'NOTICE'
    - 'go.mod'
    - 'go.sum'
  comment: on-failure
```

That default writes the ASF-specific Apache header. For any other project, create a config before running `check` or `fix`, even if you intend the same SPDX ID — the copyright owner and the ASF-specific text differ.

## Header section

### Map vs. list

- **Map form** (`header:` with `license`, `paths`, … directly below) applies one license to one set of paths.
- **List form** (`header:` as a YAML sequence) applies each entry independently. Every entry is checked and fixed in turn. Scope each entry with `paths`; otherwise it defaults to `**` and every file must satisfy every entry's license.

### Fields

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `license.spdx-id` | string | — | SPDX identifier used to look up built-in header text and, for `dependency check`, the compatibility matrix. |
| `license.copyright-owner` | string | — | Replaces `[owner]` in the header text. An ASF-like value changes the Apache-2.0 template. |
| `license.copyright-year` | string | current year | Replaces `[year]`; set a fixed year or range to avoid annual churn. |
| `license.software-name` | string | — | Replaces `[software-name]`. |
| `license.content` | string | — | Full custom header text. Wins over `spdx-id` for both check and fix. |
| `license.pattern` | string | — | Optional regexp for existing headers that differ from the canonical text; also used to remove old headers during `fix`. |
| `paths` | string[] | `['**']` | Files/globs to check. Supports `**`. |
| `paths-ignore` | string[] | `[]` (Git ignore rules apply separately) | Files/globs to skip. Takes precedence over `paths`. |
| `comment` | `on-failure` \| `always` \| `never` | none | When to comment on a pull request. Requires a token and a `pull_request` event. |
| `license-location-threshold` | int | `80` | Maximum index of the header in the normalized file content. |
| `language` | map | built-in | Comment-style overrides keyed by language name. |

## License fields

### `spdx-id`

The SPDX ID selects the header template used by `fix` and the expected text used by `check`. The built-in templates are intentionally small:

| SPDX ID | Built-in template |
| --- | --- |
| `Apache-2.0` | generic Apache header, or the ASF-specific header when the owner matches ASF |
| `MIT` | MIT license text |
| `EPL-2.0` | short EPL-2.0 notice |
| `MPL-2.0` | MPL-2.0 notice |
| `GPL-3.0-or-later` | GPL notice |
| `AGPL-3.0-only`, `AGPL-3.0-or-later` | AGPL notices |
| `MulanPSL-2.0` | Mulan notice |

For any other SPDX ID (BSD-3-Clause, ISC, Unlicense, …), `spdx-id` alone is **not enough**: current releases warn `failed to find a license template for spdx id ...`, `check` can never pass, and `fix` inserts an empty comment (for example a bare `//` line) while still reporting success. Set `license.content` to the exact header text you want in that case.

### `content`

```yaml
header:
  license:
    spdx-id: BSD-3-Clause
    copyright-owner: Acme Corp
    content: |
      Copyright [year] [owner]

      Redistribution and use in source and binary forms, with or without
      modification, are permitted provided that the following conditions are met:
      ...
```

`content` is used verbatim for checking and fixing. Supported placeholders: `[year]`, `[owner]`, `[software-name]`. If `content` is non-empty it wins over the SPDX template, even when `spdx-id` is also set.

### `pattern`

`pattern` handles existing headers that are semantically the same but textually different — a different copyright line, an extra disclaimer, a legacy URL. It is a Go regular expression processed as follows:

1. The pattern is normalized (comment markers are **not** stripped; whitespace is flattened, case is folded).
2. Checking wraps it as `(?i).*<pattern>.*` and searches the normalized file content.
3. Fixing composes a comment-style-aware regexp from the same pattern and removes any match before inserting the new header.

Practical rules:

- Write plain license text, not a regexp with comment markers (`//`, `#`, `/*`).
- Escape regexp metacharacters such as `.` and parentheses.
- Keep it anchored to a distinctive phrase; a pattern that matches ordinary code can be deleted by `fix`.
- Without `pattern`, `fix` prepends the new header but leaves an older non-matching header in place, producing a duplicate. Set `pattern` when migrating existing headers.

## Built-in SPDX header templates

The templates live in `assets/header-templates/` upstream. Their general shape:

- `Apache-2.0` — `Copyright [year] [owner]` plus the Apache-2.0 warranty notice.
- `Apache-2.0` with an ASF-like owner — the ASF contributor-agreement header used by Apache projects.
- `MIT` — the full MIT license text (verbose as a file header; use `content` for a shorter notice).
- `EPL-2.0` — copyright line, short EPL notice, and `SPDX-License-Identifier: EPL-2.0`.
- `MPL-2.0`, `GPL-3.0-or-later`, AGPL variants, `MulanPSL-2.0` — their respective notices.

`software-name` and `copyright-year` are optional; `year` defaults to the current year at run time.

## Paths and exclusions

- `paths` supports `doublestar` globs: `**`, `*`, `?`, character classes.
- `paths-ignore` is evaluated after `paths`; any match wins.
- A trailing `/` or an exact name also matches a directory tree.
- In a Git repository, file discovery uses the Git index plus working-tree changes, so untracked files ignored by Git are not scanned. Add tracked generated files to `paths-ignore` explicitly.
- Outside a Git repository, `paths` is glob-expanded directly and `.gitignore` is not applied.
- The config file itself is a tracked file. If `paths: ['**']`, `.licenserc.yaml` must either carry the header or be excluded. ASF's own config carries the ASF header.

Typical exclusions:

```yaml
  paths-ignore:
    - '**/*.md'
    - '**/*.json'        # JSON has no comments; cannot be fixed
    - '**/*.txt'
    - 'LICENSE'
    - 'NOTICE'
    - 'dist/**'
    - 'target/**'
    - 'vendor/**'
    - 'node_modules/**'
    - '**/generated/**'
    - '**/testdata/**'
    - '.licenserc.yaml'
```

Exclude generated or vendored files because their upstream license governs them, not because CI is red.

## License location threshold

`license-location-threshold` is the maximum index of the matched license in the **normalized** file content. Normalization strips comment markers, flattens whitespace, lowercases, and collapses many license-equivalent variants, so the index is not a line number.

- Default: `80`.
- A long shebang, an encoding declaration, or a generated preamble can push the header past the threshold.
- Raise it only for files that cannot avoid a preamble. Lowering it makes checks stricter and can reject legitimate headers.

## PR comments

`comment` controls GitHub pull-request comments:

| Value | Behavior |
| --- | --- |
| `on-failure` | Comment only when invalid files are found. |
| `always` | Comment after every run on a pull request. |
| `never` (or unset) | Never comment. |

Comments require all of:

- a `pull_request` event (`GITHUB_EVENT_NAME=pull_request`);
- `GITHUB_TOKEN` (the action passes `github.token` by default);
- `pull-requests: write` permission;
- the GitHub Actions environment variables that the `apache/skywalking-eyes/header` action sets (the bare CLI can comment only when those variables are provided manually).

Set `comment: never` or the action input `token: ""` to disable comments deliberately. Commenting is best-effort; a failure to comment does not change the check result.

## Language overrides

`fix` needs a comment style for the file's extension or filename. Built-in styles include:

| Style ID | Shape | Typical languages |
| --- | --- | --- |
| `DoubleSlash` | `//` | Go, Rust, C-family, Protobuf |
| `SlashAsterisk` | `/* ... */` | Java, Kotlin, Scala, CSS, JS/TS |
| `Hashtag` | `#` | Python, Ruby, Shell, YAML, TOML, Dockerfile |
| `PythonStyle` | `#`, after shebang/encoding line | Python |
| `AngleBracket` | `<!-- ... -->` | HTML, XML, Markdown, Vue, Svelte |
| `DoubleDash` | `--` | SQL, Lua, Haskell |
| `Semicolon` | `;` | INI, Clojure |
| `Percent` | `%` | MATLAB, LaTeX |
| `PowerShellStyle` | `<# ... #>`, after shebang | PowerShell |
| `PhpTag` | `<?php` boundary | PHP |

To teach `fix` a new extension or filename, add a `language` entry keyed by a language name. `extensions` and `filenames` select files; `comment_style_id` must be one of the style IDs known to the binary.

```yaml
header:
  license:
    spdx-id: Apache-2.0
    copyright-owner: Acme Corp
  language:
    AcmeTemplate:
      extensions:
        - '.acme'
      filenames:
        - 'Acmefile'
      comment_style_id: DoubleSlash
```

`filenames` entries are literal suffixes, not globs: `Acmefile` matches any path ending in `Acmefile`. Extensionless files such as Java `META-INF/services/*` therefore need one entry per filename, or an exclusion. `check` verifies extensionless text files even without an override, but `fix` cannot add a header unless an extension or filename suffix matches. If those service files should carry `#` headers, list each name with `comment_style_id: Hashtag`.

The upstream `assets/languages.yaml` maps hundreds of extensions to styles. If a file type is absent there, `fix` reports `unsupported file: <path>` and exits non-zero after fixing the rest. Either add an override, exclude the path, or convert the file to a supported type.

## Multiple licenses in one repository

Use the list form and scope every entry with `paths`:

```yaml
header:
  - license:
      spdx-id: Apache-2.0
      copyright-owner: Apache Team
    paths:
      - 'apache-module/**'
    paths-ignore:
      - '**/*.md'
  - license:
      spdx-id: MIT
      copyright-owner: MIT Team
    paths:
      - 'mit-module/**'
    paths-ignore:
      - '**/*.md'
```

Important: older upstream documentation shows a `path:` key on each entry. Current releases do not read that field; it is silently ignored, and both headers apply to all files, causing every file to fail one of the checks. Use `paths` instead.

If the regions cannot be expressed with globs, use separate config files and invoke the tool once per module:

```bash
license-eye -c module-a/.licenserc.yaml header check
license-eye -c module-b/.licenserc.yaml header check
```

Remember that `header.paths` is relative to the working directory, so run each command from the repository root or from the module directory as appropriate.

## Dependency section

```yaml
dependency:
  files:
    - go.mod
  licenses:
    - name: github.com/example/legacy
      version: v1.2.3,v1.2.4
      license: MIT
  excludes:
    - name: com.example:build-tool
      recursive: true
  threshold: 75
  require_fsf_free: false
  require_osi_approved: false
```

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `files` | string[] | `[]` | Dependency declaration files. Relative paths are resolved against the config file's directory. |
| `licenses` | list | `[]` | Manual license overrides for dependencies the resolver cannot identify. |
| `excludes` | list | `[]` | Dependencies to omit from analysis. |
| `threshold` | int | `75` | Minimum percentage of a license file that must match before the license is identified. |
| `require_fsf_free` | bool | `false` | Treat non-FSF-free licenses as incompatible. |
| `require_osi_approved` | bool | `false` | Treat non-OSI-approved licenses as incompatible. |

Supported `files` patterns:

| Pattern | Resolver |
| --- | --- |
| `*.mod` | Go modules |
| `package.json` | npm, or pnpm when `packageManager` names pnpm or a `pnpm-lock.yaml` exists |
| `pom.xml` | Maven |
| `*.jar` | JAR license scan |
| `Cargo.toml` | Cargo |
| `Gemfile.lock` | Bundler |
| `*.gemspec` | Ruby gemspec |

`licenses[].name` supports shell-style globs (`filepath.Match`), for example `org.apache.skywalking:*`. `version` is a comma-separated list of exact versions; omit it to match every version. `excludes[]` uses the same name/version matching; `recursive` is honored only by the Maven resolver.

For resolver behavior and side effects, see `dependency-licenses.md`.

## Dependency-only configurations

A config containing only `dependency:` parses successfully but produces **zero** header entries. As a result:

- `dependency check` loops over no headers and exits 0 without checking anything.
- `header check` also exits 0 without checking anything.

This is a silent false pass. If a repository wants dependency checks without header checks, include a header section with the real SPDX ID and ignore everything from the header checker:

```yaml
header:
  license:
    spdx-id: Apache-2.0
    copyright-owner: Acme Corp
  paths-ignore:
    - '**'

dependency:
  files:
    - go.mod
```

`dependency check` then uses `Apache-2.0` as the main license, while `header check` sees no eligible files.

## Starter configurations

### Single license, common exclusions

```yaml
header:
  license:
    spdx-id: Apache-2.0
    copyright-owner: Acme Corp
    copyright-year: '2024-2026'
  paths:
    - '**'
  paths-ignore:
    - '**/*.md'
    - '**/*.json'
    - '**/*.txt'
    - 'LICENSE'
    - 'NOTICE'
    - 'dist/**'
    - 'vendor/**'
  comment: on-failure
```

### Monorepo with per-module licenses

See [Multiple licenses in one repository](#multiple-licenses-in-one-repository). Add a separate `dependency.files` list for each module if their dependency graphs differ:

```yaml
dependency:
  files:
    - module-a/go.mod
    - module-b/package.json
```

### Apache Software Foundation project

```yaml
header:
  license:
    spdx-id: Apache-2.0
    copyright-owner: Apache Software Foundation
  paths-ignore:
    - 'dist'
    - 'licenses'
    - '**/*.md'
    - 'LICENSE'
    - 'NOTICE'
  comment: on-failure
```

The ASF-like owner automatically selects the ASF contributor header. Do not use this config for a non-ASF project merely because it uses Apache-2.0.

## Common configuration mistakes

| Mistake | Effect | Fix |
| --- | --- | --- |
| Only `dependency:` present | Both commands silently pass. | Add a `header` section with `spdx-id`. |
| `spdx-id: BSD-3-Clause` with no `content` | `check` fails forever; `fix` inserts an empty comment. | Provide `license.content`. |
| `path:` on a list entry | Ignored; all headers apply everywhere. | Use `paths`. |
| `.licenserc.yaml` not excluded and no header | The config file itself fails the check. | Add the header or list the file in `paths-ignore`. |
| JSON under `paths` | `fix` errors with `unsupported file`. | Exclude `**/*.json`. |
| `pattern` contains `//` or `/*` | It never matches normalized content. | Write plain license text. |
| `pattern` too broad | `fix` deletes real code. | Narrow it to a distinctive phrase. |
| Overly broad `paths-ignore` | Checks pass without checking the files that matter. | Review exclusions as a policy change. |
