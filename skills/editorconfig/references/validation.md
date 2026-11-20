# Validating and maintaining EditorConfig

EditorConfig only affects editors that have a plugin. Add a checker to catch drift in CI, and align the file with the project's formatters.

## Contents

- [editorconfig-checker](#editorconfig-checker)
- [Configuration](#configuration)
- [Pre-commit](#pre-commit)
- [CI](#ci)
- [Formatter alignment](#formatter-alignment)
- [Troubleshooting](#troubleshooting)
- [Checklist](#checklist)

## editorconfig-checker

[`editorconfig-checker`](https://github.com/editorconfig-checker/editorconfig-checker) checks files against `.editorconfig`. It reads all files tracked by Git when run from the repository root, and supports `--fix` for a safe subset (trailing whitespace, line endings, final newline).

Install a pinned version:

```sh
go install github.com/editorconfig-checker/editorconfig-checker/v4/cmd/editorconfig-checker@v4.0.2
editorconfig-checker
```

Docker:

```sh
docker run --rm --volume="$PWD:/check" \
  ghcr.io/editorconfig-checker/editorconfig-checker:4.0.2
```

Useful flags:

```sh
editorconfig-checker --dry-run
editorconfig-checker --fix
editorconfig-checker --format github-actions
editorconfig-checker --disable-indentation
```

`--fix` handles trailing whitespace, `end_of_line`, and `insert_final_newline`; indentation, charset, and max-line-length violations remain check-only. A second `--fix` is idempotent.

Inline suppression:

```js
const template = `
  intentionally odd indentation`; // editorconfig-checker-disable-line
```

Use `editorconfig-checker-disable-next-line` for the next line, and `editorconfig-checker-disable-file` at the top to skip a generated file. Suppression should be rare and justified.

## Configuration

`.editorconfig-checker.json`:

```json
{
  "$schema": "https://raw.githubusercontent.com/editorconfig-checker/editorconfig-checker/v4.0.2/.editorconfig-checker.schema.json",
  "Verbose": false,
  "Exclude": ["vendor/", "node_modules/", "dist/", "\\.min\\.(js|css)$"],
  "Disable": {
    "EndOfLine": false,
    "Indentation": false,
    "IndentSize": false,
    "InsertFinalNewline": false,
    "TrimTrailingWhitespace": false,
    "MaxLineLength": false,
    "Charset": false
  }
}
```

- `Exclude` entries are regular expressions.
- `Version` can pin the checker version and fail if the binary differs.
- `Disable` turns off individual checks; use it only for files/properties the project cannot satisfy.
- Default excludes and allowed content types apply unless `IgnoreDefaults` is set.

## Pre-commit

```yaml
repos:
  - repo: https://github.com/editorconfig-checker/editorconfig-checker
    rev: v4.0.2
    hooks:
      - id: editorconfig-checker
```

The hook requires Go to build the checker. Pin `rev` to the same version used in CI.

## CI

GitHub Actions example:

```yaml
name: editorconfig

on:
  pull_request:
  push:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - uses: actions/setup-go@d8cbb8045d7f43706b9177cceb36c1f91a6644d5 # v7.1.0
        with:
          go-version: "stable"
      - run: go install github.com/editorconfig-checker/editorconfig-checker/v4/cmd/editorconfig-checker@v4.0.2
      - run: editorconfig-checker --format github-actions
```

- Pin the checker version; `@latest` makes CI nondeterministic.
- Run it on pull requests and the default branch.
- `--format github-actions` produces inline annotations.
- Do not auto-fix in CI; fail and let the contributor or a formatter fix it.
- The pinned SHAs correspond to `actions/checkout` v7.0.1 and `actions/setup-go` v7.1.0; both run on the runner's Node 24 runtime. Node 20 and action majors declaring `runs.using: node20` are not supported on GitHub-hosted runners after 2026-09-23.

## Formatter alignment

EditorConfig is a baseline; formatters own final output. If they disagree, CI and editors will fight.

| Tool | EditorConfig alignment |
| --- | --- |
| Prettier | `useTabs`, `tabWidth`, `endOfLine` |
| ESLint `indent` | `indent_style`, `indent_size` (if the rule is enabled) |
| rustfmt | `hard_tabs`, `tab_spaces` |
| gofmt | tabs; no width option |
| Black | 4 spaces; `line-length` separate |
| shfmt | `-i`, `-ci` |
| terraform fmt | 2 spaces |
| .NET format | `dotnet_*` keys in `.editorconfig` (tooling-specific) |

Examples of conflicts:

- EditorConfig says 2 spaces; Prettier `tabWidth: 4` → Prettier rewrites on save/CI.
- EditorConfig says spaces; `rustfmt` `hard_tabs = true` → rustfmt wins in CI.
- EditorConfig trims Markdown trailing whitespace; the rendered hard break disappears.
- EditorConfig says LF; Windows-only scripts are checked out CRLF → line-ending churn.

Pick one source of truth for formatting. EditorConfig should describe the same style, not a competing one.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Checker passes but editor formats differently | Editor plugin/formatter ignores EditorConfig | Configure the formatter or disable its conflicting setting. |
| Checker flags generated files | Generated/minified files do not follow the style | Exclude them in `.editorconfig-checker.json`; do not hand-edit generated files. |
| Checker flags a file that must be different | Language/tool exception | Add a narrower section (`[Makefile]`, `[*.md]`) instead of disabling the check globally. |
| Line endings keep changing | Git autocrlf/`.gitattributes` disagrees with `end_of_line` | Normalize with `.gitattributes`; align `end_of_line`. |
| BOM keeps appearing | `charset = utf-8-bom` or a Windows editor adds one | Use `utf-8`; remove the BOM. |
| Nested package ignores root | A nested `root = true` stops inheritance | Remove it unless the nested directory is an independent root. |
| Settings apply to unexpected files | Glob is broader than intended | Use a narrower section; remember `*` does not cross `/`. |
| `max_line_length` is ignored | Not a core EditorConfig property | Set line length in the formatter/linter. |

## Checklist

- [ ] `.editorconfig` is lowercase, UTF-8, and at the intended root.
- [ ] `root = true` is in the repository root config.
- [ ] Generic `[*]` section comes before specific sections.
- [ ] Indentation matches the project formatters.
- [ ] Markdown keeps trailing whitespace allowed.
- [ ] Makefile and Go use tabs.
- [ ] YAML never uses tabs.
- [ ] Windows-only scripts have the intended line endings.
- [ ] `.gitattributes` agrees with `end_of_line`.
- [ ] `editorconfig-checker` is pinned in CI/pre-commit.
- [ ] Exclusions are limited to generated/vendor files.
- [ ] The file was tested in at least two editors.
