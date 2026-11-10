---
name: license
description: >-
  Use this skill for license-compliance work in code repositories: adding or fixing SPDX/copyright license headers repo-wide, diagnosing a failing license-header check or a CI job named license-header / dependency-license, configuring `.licenserc.yaml`, using license-eye (apache/skywalking-eyes) for header check/fix/diff or dependency resolve/check, wiring skywalking-eyes actions into CI, auditing dependency license compatibility, handling Unknown or incompatible licenses, and regenerating a LICENSE summary from the dependency graph. Trigger on terse asks like "CI license check failed", "add SPDX headers", "fix copyright headers", or "is this dependency license OK". Chinese cues: 许可证头, 版权头, 依赖许可证检查, 开源协议合规. Skip ESLint or other linter/build-tool header comments, authoring NOTICE or third-party attribution files, general license choice or relicensing advice, explaining an existing LICENSE's terms, and website copyright notices.
---

# License compliance with skywalking-eyes

[Apache SkyWalking Eyes](https://github.com/apache/skywalking-eyes) (`license-eye`) is one binary with two independent jobs:

- **`header`** — check, fix, or diff license headers in source files.
- **`dependency`** — resolve dependency licenses, check compatibility, and generate a LICENSE summary.

They share `.licenserc.yaml` but fail for different reasons. Diagnose the job that failed before changing anything: a header failure is a file-content problem; a dependency failure is a dependency-graph or policy problem.

`license-eye` is configuration-driven and somewhat blunt. In a Git repository it walks tracked files plus uncommitted changes, applies `paths` / `paths-ignore`, and compares a normalized header text. When `.licenserc.yaml` is missing it silently uses a built-in Apache-2.0 / Apache Software Foundation config — correct only for ASF projects. Everywhere else that default produces the wrong headers, so always confirm the config before fixing or adopting the tool.

## Inspect before changing anything

Run from the repository root. Config paths and Git discovery are relative to the working directory, and `fix` rewrites files in place.

```bash
# Config, project license, and CI usage
ls -la .licenserc.yaml LICENSE NOTICE 2>/dev/null
sed -n '1,160p' .licenserc.yaml 2>/dev/null
grep -R "skywalking-eyes\|license-eye" .github/workflows 2>/dev/null

# Project type and dependency manifests
ls go.mod pom.xml package.json Cargo.toml Gemfile.lock 2>/dev/null
head -20 LICENSE 2>/dev/null

# Tooling available
license-eye --version || go version || docker --version
git status --short
```

Then establish four facts:

1. **Is there a `.licenserc.yaml`?** No config means the Apache-2.0/ASF default. Do not let that decide a non-ASF project's license; ask for the intended SPDX ID and copyright owner first.
2. **Which job failed?** Header jobs fail on file contents. Dependency jobs fail on the dependency graph, unknown licenses, or compatibility policy.
3. **What is the repository's policy?** Read existing `paths-ignore` and generated-file exclusions. Do not loosen exclusions just to make CI green — excluded paths are a licensing decision the maintainers own.
4. **What toolchain does the dependency check need?** Go, Maven, Node/npm/pnpm, Cargo, or Ruby. The binary alone is not enough.

## Header workflow

### Check first

```bash
license-eye header check                    # uses .licenserc.yaml
license-eye -c path/to/.licenserc.yaml header check
license-eye header check path/to/dir        # override configured paths for this run
```

The summary line `valid: N, invalid: M, ignored: K, fixed: 0` is the truth. Exit code is 1 when at least one file is invalid, and the invalid list is printed. Work from that list, not from the summary alone.

### Fix, review, re-check

```bash
license-eye header fix
git diff --stat
git diff                                    # inspect inserted headers
license-eye header check                    # must now exit 0
```

`fix` inserts the configured header at the top of each invalid file, after a shebang or `<?php` where the language requires it. It rewrites files in place, so run it on a clean working tree and review the diff. When the failing set is known, pass paths explicitly to avoid touching unrelated files:

```bash
license-eye header fix src/new-module
```

Exit-code semantics matter:

- `header fix` exits 0 when it fixed every invalid file. That is why fix mode cannot gate CI: it turns failures into edits.
- It exits non-zero if a file cannot be fixed — typically an unsupported file type or an unreadable encoding. Fix what it can, then handle the rest.
- `header check` and `header diff` exit 1 on invalid files. Those are the commands to use as CI gates.

### Diagnose a header that looks present

When the header is visibly there but `check` fails, run:

```bash
license-eye header diff
```

It compares the configured license and the file in the same normalized form as `check` (comment markers stripped, whitespace flattened, case-insensitive), so every displayed difference is a real cause:

- `[-text-]` is expected by the config but missing in the file.
- `{+text+}` is in the file but not expected by the config.

Common causes are a typo in the license URL, a different copyright line, or a header that starts too far into the file (`license-location-threshold`, default 80 normalized characters).

### Unsupported file types

`check` reads any text file. `fix` only writes files whose extension or filename has a known comment style (Go, Java, Python, JS/TS, Rust, C/C++, YAML, XML, HTML, Markdown, Shell, SQL, and others in the upstream `assets/languages.yaml` table). JSON has no comment syntax, so it cannot be fixed — exclude it. If a new extension is missing, add a `header.language` override (see `references/configuration.md`) or add the path to `paths-ignore` if the file genuinely should not carry a header.

One trap: `license-eye` checks its own `.licenserc.yaml` like any other tracked file. Either add the header to it or list it in `paths-ignore`.

## Configure `.licenserc.yaml`

A minimal config for a non-ASF project:

```yaml
header:
  license:
    spdx-id: Apache-2.0
    copyright-owner: Acme Corp
    copyright-year: '2020-2026'   # omit to use the current year

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
    - 'generated/**'

  comment: on-failure   # PR comments: on-failure, always, never
```

Key semantics:

- **`spdx-id` only has built-in header text for a small set of licenses.** Current releases ship templates for Apache-2.0, MIT, EPL-2.0, MPL-2.0, GPL-3.0-or-later, AGPL-3.0-only/or-later, and MulanPSL-2.0. For any other SPDX ID, set `license.content` explicitly. Otherwise `fix` inserts an empty comment and `check` can never pass.
- **Apache-2.0 plus an ASF-like owner** (`Apache Software Foundation` / `ASF`) selects the ASF-specific header rather than the generic Apache one. This is another reason the missing-config default is dangerous outside ASF projects.
- **`content` replaces the SPDX template** for both check and fix, and supports `[year]`, `[owner]`, and `[software-name]` placeholders.
- **`pattern`** is an optional case-insensitive regexp for existing headers that differ from `content` / `spdx-id`. Write it as plain license text without comment markers; it is matched against normalized file content and is also used by `fix` to delete an old header before inserting the new one. Keep it narrow — a broad pattern can delete real content.
- **`paths` defaults to `['**']`**, and `paths-ignore` wins over `paths`. In a Git repo, gitignored files are already skipped; list tracked generated files explicitly.
- **`license-location-threshold`** defaults to 80. Raise it only for files with a long unavoidable preamble; lowering it hides headers that are too far down.
- **The `header` section can be a list**, one entry per license region, each with its own `paths`. Use this for a monorepo whose modules carry different licenses. Scope each entry with `paths`; the `path:` key shown in some older upstream examples is ignored by current releases.

Full option reference, multi-license examples, and language overrides: `references/configuration.md`.

## Dependency licenses

The `dependency` command has two modes:

```bash
license-eye dependency resolve                         # print resolved licenses
license-eye dependency resolve --summary LICENSE.tpl   # write LICENSE next to the template
license-eye dependency resolve --license out/LICENSE   # write to an explicit path
license-eye dependency resolve --output licenses/      # save each dependency's license text
license-eye dependency check                           # fail on unknown or incompatible licenses
```

`resolve` is the audit/report mode. It exits non-zero when dependencies remain unidentified (current releases; the upstream README's older "always 0" note is stale), so inspect the report rather than relying on the exit code alone. `check` reuses the resolved graph and compares each license against the built-in compatibility matrix for the project's `spdx-id`; unknown licenses count as failures. The main license must be an SPDX ID with a matrix, and the config needs a `header` section — without `header`, `Config.Headers()` is empty and `dependency check` silently does nothing.

**Before running either mode locally, warn about side effects.** Resolution is ecosystem-specific and not read-only:

| Manifest | Resolver behavior | Side effects |
| --- | --- | --- |
| `go.mod` | `go mod download`, `go mod download -json` | Downloads modules to the Go cache |
| `pom.xml` | `mvn` / `mvnw dependency:resolve`; may fall back to `clean install -DskipTests` | Builds the project, writes `target/`, populates the local Maven repository |
| `package.json` (npm) | may run `npm ci`; always runs `npm prune --production`, then `npm ls` | **Removes devDependencies from `node_modules`** |
| `package.json` (pnpm) | may run `pnpm ci` / `install --frozen-lockfile`; uses `pnpm licenses list` | Downloads and installs production packages; does not prune like npm |
| `Cargo.toml` | `cargo fetch`, `cargo metadata` | Populates the Cargo cache |
| `Gemfile.lock` / `*.gemspec` | reads the lockfile, queries RubyGems / gem.coop for missing licenses | Network access; no installs |

Because of the npm prune and Maven build, prefer running dependency jobs in CI or a disposable checkout. If you must run locally on a Node project, reinstall dev dependencies afterwards (`npm ci`) and say so. In a non-interactive session, npm/pnpm resolution also waits five seconds for a skip keypress and then installs packages; pre-install dependencies or pipe `s` plus a newline if they are already present.

Handling failures:

- **Unknown license** (`Unknown` in the report): verify the dependency's actual license and add a `dependency.licenses` override. Do not guess or blanket-ignore.
- **Incompatible license**: this is a policy decision. `--weak-compatible` accepts weak-copyleft combinations; `--fsf-free` / `--osi-approved` are stricter. Confirm with the project owner before relaxing.
- **Dependency not distributed** (build-only or test-only): add it to `dependency.excludes`; Maven also supports `recursive: true` for its transitive tree. Non-compile Maven scopes are excluded automatically.
- **No `header` section**: dependency check silently passes. Add a `header` section with the project `spdx-id` (and `paths-ignore: ['**']` if you deliberately want dependency-only checking).

To generate a committed LICENSE summary and catch drift in CI:

```bash
license-eye dependency resolve --summary path/to/LICENSE.tpl || exit 1
git diff --exit-code path/to/LICENSE
```

`--summary` writes `LICENSE` in the template's directory unless `--license` is also passed. Resolve on the same OS as CI: platform-specific dependency variants can change the output, so a diff produced on macOS/Windows may be an artifact rather than a stale LICENSE.

Full per-ecosystem notes, summary template variables, and compatibility flags: `references/dependency-licenses.md`.

## GitHub Actions

Pin the action; do not use `@main`. The actions are composite and build the Go binary on each run, so a release tag or commit SHA keeps behavior stable.

```yaml
permissions:
  contents: read

jobs:
  license-header:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: apache/skywalking-eyes/header@v0.9.0
        with:
          config: .licenserc.yaml
          mode: check
```

For dependency compatibility:

```yaml
  dependency-license:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: apache/skywalking-eyes/dependency@v0.9.0
        with:
          config: .licenserc.yaml
          mode: check
```

Notes:

- `mode: fix` writes headers but exits 0 when it succeeds, so it cannot block CI. Use `check` as the gate, and a separate fix-and-commit job if automation is wanted.
- PR comments need `pull-requests: write`, a `pull_request` event, and a token. Set `comment: never` or `token: ""` to disable them.
- Dependency checks need the relevant toolchain (`setup-go`, `setup-java`, `setup-node`, `ruby/setup-ruby`, …) and network access on the runner.
- If the CLI is used outside Actions, install it once with `go install github.com/apache/skywalking-eyes/cmd/license-eye@v0.9.0` instead of invoking the action.

Full workflows, fix-and-commit, dependency resolve/drift, and container examples: `references/ci.md`.

## Troubleshooting map

| Symptom | Likely cause |
| --- | --- |
| `header check` says `invalid: N` but the header looks right | Typo or whitespace/case difference; run `header diff`. |
| Header is far below the top of the file | `license-location-threshold` (default 80) is too low. |
| `fix` does nothing | Path is excluded by `paths` / `paths-ignore`, or the file is not tracked/visible in the Git repo. |
| `fix` reports `unsupported file` | No comment style for that extension; add `header.language` or exclude the file. |
| `fix` inserts `//` with no text | `spdx-id` has no built-in template; set `license.content`. |
| Missing config behaves unexpectedly | Built-in default is Apache-2.0/ASF; create `.licenserc.yaml`. |
| `dependency check` passes with no output | No `header` section or no `spdx-id`; it has nothing to check against. |
| `resolve` says `Unknown` | Add a verified `dependency.licenses` override, or exclude an undistributed dependency. |
| `dependency check` fails on a permissive license | Compatibility matrix or `--weak-compatible` policy; confirm before relaxing. |
| LICENSE drift only on one OS | Platform-specific dependency variants; regenerate on Linux/CI. |
| npm resolution destroyed dev dependencies | `npm prune --production`; reinstall with `npm ci` and run dependency checks in CI next time. |

More detailed causes and fixes: `references/troubleshooting.md`.

## References

- `references/configuration.md` — every `.licenserc.yaml` field, defaults, multi-license and language recipes.
- `references/cli.md` — command tree, flags, exit codes, installation methods, summary templates.
- `references/dependency-licenses.md` — each resolver's requirements and side effects, compatibility checks, overrides and excludes.
- `references/ci.md` — GitHub Actions workflows, permissions, fix patterns, LICENSE drift checks.
- `references/troubleshooting.md` — symptom-by-symptom fixes, especially false passes and false failures.

Adapted from the Apache SkyWalking `license` skill and the upstream skywalking-eyes documentation.
