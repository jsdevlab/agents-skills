# `license-eye` CLI reference

The binary is named `license-eye`. It is published as part of Apache SkyWalking Eyes (formerly SkyWalking Eyes). Commands are grouped under `header` and `dependency`; both are configured by the same `.licenserc.yaml`.

## Contents

- [Installation](#installation)
- [Command tree](#command-tree)
- [Global flags](#global-flags)
- [Header commands](#header-commands)
- [Dependency commands](#dependency-commands)
- [Exit codes](#exit-codes)
- [Output and normalization](#output-and-normalization)
- [Docker usage](#docker-usage)
- [Examples](#examples)

## Installation

### Release archive

The examples below use `v0.9.0`; check the releases page for the current version. Download the `skywalking-license-eye-<version>-bin.tgz` archive from the GitHub releases page or the Apache SkyWalking downloads page. It contains per-OS binaries under `bin/linux`, `bin/darwin`, and `bin/windows` (`license-eye` or `license-eye.exe`). Verify the `.sha512` and `.asc` files published alongside it, then put the binary on `PATH`.

### Go install

```bash
go install github.com/apache/skywalking-eyes/cmd/license-eye@v0.9.0
# or the latest tagged release
go install github.com/apache/skywalking-eyes/cmd/license-eye@latest
```

The binary lands in `$(go env GOPATH)/bin` (usually `~/go/bin`). The upstream module currently requires Go 1.27, matching the toolchain the actions set up. With an older Go toolchain, either upgrade, rely on `GOTOOLCHAIN=auto`, or use a release archive or the container image.

### Homebrew (macOS)

```bash
brew install license-eye
```

### Build from source

```bash
git clone https://github.com/apache/skywalking-eyes
cd skywalking-eyes
make build
# binaries under bin/linux, bin/darwin, bin/windows
```

### GitHub Actions

Use the composite actions directly; they build the binary on the runner:

```yaml
- uses: apache/skywalking-eyes/header@a196742f472feaffafea537ce5a2a4c3c53a8de4 # v0.9.0
- uses: apache/skywalking-eyes/dependency@a196742f472feaffafea537ce5a2a4c3c53a8de4 # v0.9.0
```

See `ci.md` for inputs, permissions, and complete workflows. Pin a full commit SHA rather than `@main` or a tag.

### Check the installed version

```bash
license-eye --version
```

There is no `version` subcommand. `--version` is a root flag. A `dev` version means the binary was built from an untagged checkout.

## Command tree

```text
license-eye [global flags] <command>

header (alias: h)
  check (c) [paths...]
  fix   (f) [paths...]
  diff  (d) [paths...]

dependency (aliases: d, deps, dep, dependencies)
  resolve (r) [flags]
  check   (c) [flags]

completion
help
```

`header check` and `header fix` accept file paths, directories, or glob patterns as positional arguments. When present, they **replace** the config's `header.paths` for that invocation; `paths-ignore` still applies. When omitted, the configured `paths` are used.

The `dependency` command's `check` alias collides with the `header` command's `check` alias only at different levels — always spell out `header check` or `dependency check` in scripts.

## Global flags

| Flag | Short | Default | Meaning |
| --- | --- | --- | --- |
| `--config` | `-c` | `.licenserc.yaml` | Config file path. |
| `--verbosity` | `-v` | `info` | Log level: `debug`, `info`, `warn`, `error`, `fatal`, `panic`. |
| `--version` | — | — | Print the version and exit. |

Global flags go before the subcommand:

```bash
license-eye -v debug -c configs/.licenserc.yaml header check
license-eye -v warn -c path/to/.licenserc.yaml dependency resolve
```

## Header commands

### `header check [paths...]`

Walks the configured paths and verifies each text file's normalized content contains the configured license within `license-location-threshold` (default 80 normalized characters).

```bash
license-eye header check
license-eye header check src/ pkg/
license-eye -c module-a/.licenserc.yaml header check
```

Output:

```text
INFO Totally checked 30 files, valid: 12, invalid: 12, ignored: 6, fixed: 0
ERROR the following files don't have a valid license header:
...
```

- Exit code `0` when `invalid: 0`.
- Exit code `1` when any file is invalid.
- Binary files are skipped after MIME sniffing; text files with undecodable encodings are reported as failures rather than aborting the run.
- In a Git repository, only tracked files plus working-tree changes are considered, and untracked files ignored by Git are excluded. Outside Git, `paths` is glob-expanded directly.
- On GitHub Actions, the summary is also appended to `GITHUB_STEP_SUMMARY`.

### `header fix [paths...]`

Checks the same files and inserts the configured header into each invalid one. It writes files in place.

```bash
license-eye header fix
license-eye header fix src/new-module
```

Behavior:

- The header is placed at the top of the file, after a shebang or `<?php` boundary when the comment style requires it.
- If `license.pattern` is set, text matching the pattern (including an old header) is removed before insertion; otherwise an older non-matching header remains below the new one.
- Exit code `0` when every invalid file was fixed. It does **not** indicate that files were invalid; that is why fix mode cannot gate CI.
- Exit code non-zero when a file cannot be fixed, for example `unsupported file: data.json` or an unreadable encoding. Other files are still fixed.
- Files with no known comment style (`comments.FileCommentStyle` returns nil) are unsupported. JSON is the common case.

### `header diff [paths...]`

Shows why each invalid file fails, using the same normalized comparison as `check`:

```text
src/main.go:
    ... copy of the license at [-http://www.apache.org/licenses/license-2.0-] {+http://www.apache.org/licenses/license-2.1+} unless required by applicable law ...
```

- `[-text-]` — expected by the configured license but missing in the file.
- `{+text+}` — present in the file but not expected by the configured license.
- Long unchanged or missing runs are collapsed into `...`.
- Exit code `1` when any file is invalid, `0` otherwise.

Use `diff` before editing a header by hand; it distinguishes a real typo from a threshold problem.

## Dependency commands

Both modes read `dependency.files` and resolve the full dependency graph. The resolvers are ecosystem-specific; see `dependency-licenses.md` for toolchain requirements and side effects.

### `dependency resolve`

Prints a table and optionally writes artifacts:

```text
Dependency             |      License | Version
---------------------- | ------------ | ------
github.com/google/uuid | BSD-3-Clause | v1.6.0
```

| Flag | Short | Meaning |
| --- | --- | --- |
| `--output <dir>` | `-o` | Write each resolved dependency's license text to `<dir>/license-<name>.txt`. |
| `--summary <template>` | `-s` | Render a LICENSE summary; the output file is named `LICENSE` in the template's directory. |
| `--license <path>` | `-l` | Write the rendered summary to this path. If no `--summary` is given, a default template is used. |

Combinations:

```bash
license-eye dependency resolve
license-eye dependency resolve -o licenses/
license-eye dependency resolve -s LICENSE.tpl          # writes ./LICENSE
license-eye dependency resolve -s templates/LICENSE.tpl -l out/LICENSE
license-eye dependency resolve -l out/LICENSE           # default template
```

Behavior and exit codes:

- Prints the report to stdout after writing artifacts.
- Dependencies whose licenses cannot be identified are listed as `Unknown` and included in an error at the end. Current releases return a non-zero exit code when any dependency remains unidentified; older documentation that says "exit code is always 0" is stale. Check the output, not only the exit code.
- With multiple `header` entries, summary generation fails: `unable to write summary as multiple licenses were provided in configuration`. Use a single header entry when generating a summary.
- `--summary` without `--license` uses the template file's directory; it does not read a path from the template.

Summary template variables (Go `text/template`, with sprig functions):

| Variable | Type | Meaning |
| --- | --- | --- |
| `{{.LicenseContent}}` | string | The project license text: the SPDX template for `header.license.spdx-id`, otherwise `header.license.content`. |
| `{{range .Groups}}` | list | Dependency groups keyed by SPDX ID. |
| `{{.LicenseID}}` | string | SPDX ID of the group. |
| `{{range .Deps}}` | list | Dependencies in the group. |
| `{{.Name}}` | string | Dependency name. |
| `{{.Version}}` | string | Dependency version. |
| `{{.LicenseID}}` | string | Dependency's SPDX ID. |

Example template:

```gotemplate
{{.LicenseContent}}
{{ range .Groups }}
========================================================================
{{.LicenseID}} licenses
========================================================================
{{range .Deps}}
    {{.Name}} {{.Version}} {{.LicenseID}}
{{- end }}
{{ end }}
```

### `dependency check`

Resolves the graph and checks each license against the compatibility matrix for the project's main license. The main license comes from each `header.license.spdx-id`; with a list of headers, each is checked.

| Flag | Short | Meaning |
| --- | --- | --- |
| `--weak-compatible` | `-w` | Treat weak-compatible licenses as compatible. Use only after confirming the weak-copyleft conditions are met. |
| `--fsf-free` | `-f` | Treat non-FSF-free licenses as incompatible. |
| `--osi-approved` | `-o` | Treat non-OSI-approved licenses as incompatible. |

`-o` means "OSI-approved" for `dependency check` but "output directory" for `dependency resolve`. Do not copy flags between the two subcommands.

Behavior:

- Exit `0` when all resolved licenses are compatible and known.
- Exit `1` when any license is unknown or incompatible, with a `Dependency | License` table.
- Unknown licenses are treated as failures, not skipped.
- If the config has no `header` section, there are no main licenses to check and the command exits `0` without doing anything. This is a silent false pass.

## Exit codes

| Command | `0` | non-zero |
| --- | --- | --- |
| `header check` | No invalid files | At least one invalid file, or a config/IO error |
| `header diff` | No invalid files | At least one invalid file, or a config/IO error |
| `header fix` | Every invalid file fixed | A file could not be fixed, or a config/IO error |
| `dependency resolve` | Every license identified; requested artifacts written | Unidentified licenses, resolver/toolchain failure, or summary error |
| `dependency check` | All licenses compatible and known | Unknown/incompatible licenses, resolver/toolchain failure, or no resolvable graph |
| any command | Successful run | Missing/invalid config, unknown subcommand, malformed YAML |

`header fix` is not a gate: a successful fix exits `0` even though the repository was non-compliant. Gate CI with `header check`.

## Output and normalization

Header comparison is deliberately tolerant of formatting:

- Leading comment markers are stripped (`//`, `#`, `/*`, `*`, `<!--`, `--`, `;`, `%`, `..`, `rem`, quotes, and others).
- Whitespace is flattened, text is lowercased, and some license-equivalent substitutions are applied (`licence` → `license`, `(c)` → `Copyright `, `https://` → `http://`, and similar).
- The configured license must appear within the first `license-location-threshold` characters of the normalized **file** content, not the first N lines.
- `pattern` is normalized differently: comment markers are not stripped, and it is wrapped in `(?i).*...*`.

This is why `header diff` output is more trustworthy than eyeballing a header.

Dependency output is not normalized in the same way; it reports the SPDX ID the resolver inferred from the license file, `package.json`, POM metadata, or a user override. `Unknown` means the resolver could not identify a license with enough confidence.

## Docker usage

The official image mounts the repository at `/github/workspace` and expects that as the working directory.

Bash / macOS / Linux:

```bash
docker run -it --rm -v "$(pwd):/github/workspace" apache/skywalking-eyes header check
docker run -it --rm -v "$(pwd):/github/workspace" apache/skywalking-eyes header fix
```

PowerShell 7 on Windows:

```powershell
docker run -it --rm -v "${pwd}:/github/workspace" apache/skywalking-eyes header check
docker run -it --rm -v "${pwd}:/github/workspace" apache/skywalking-eyes header fix
```

Notes:

- The published image contains the `license-eye` binary only. Dependency resolution needs the relevant language toolchain, which the base image does not include. Build a custom image or run the binary directly when checking dependencies.
- `ghcr.io/apache/skywalking-eyes/license-eye:latest` tracks the main branch and is not an official Apache release. Prefer the Docker Hub release image or a pinned tag.

## Examples

### Check a pull request locally

```bash
git checkout pr-branch
license-eye header check
```

### Fix only the files a branch touched

Run the check first, read the invalid list, and pass those paths explicitly:

```bash
license-eye header check > /tmp/header-check.log 2>&1 || true
# review the "the following files don't have a valid license header" section
license-eye header fix path/to/file1 path/to/file2
```

To collect the list mechanically on a Unix-like shell:

```bash
git diff --name-only --diff-filter=ACMR -z origin/main...HEAD \
  | xargs -0 license-eye header fix
```

`header fix` accepts files, directories, and globs. Paths that no longer exist are ignored; paths with spaces need proper quoting, which `xargs -0` provides.

### Resolve licenses and keep the output for review

```bash
license-eye dependency resolve -o /tmp/licenses -s LICENSE.tpl -l /tmp/LICENSE
```

### Check a monorepo module

```bash
license-eye -c services/api/.licenserc.yaml header check
```

### Run with debug logging

```bash
license-eye -v debug header check
```

Debug output shows the normalized license, the normalized pattern, and why an individual file passed or failed. It is verbose; redirect it when checking a large repository.

### CI gate

```bash
license-eye header check
license-eye dependency check
```

Run both, but treat them as separate jobs: header failures and dependency failures require different fixes, and dependency resolution may need a different toolchain setup.
