# Dependency license resolution and checking

`license-eye dependency` has two modes over the same resolver pipeline:

- **`resolve`** — walk `dependency.files`, identify each dependency's license, print a report, and optionally write license texts or a LICENSE summary.
- **`check`** — run the same resolution, then compare every resolved license against the compatibility matrix for the project's main license.

Both modes are only as good as the resolver for each ecosystem. They run external package-manager commands, read package metadata and license files, and can have side effects on the checkout. Understand the resolver before running it in a working tree.

## Contents

- [How resolution works](#how-resolution-works)
- [Ecosystem behavior](#ecosystem-behavior)
- [Compatibility checking](#compatibility-checking)
- [Overrides and exclusions](#overrides-and-exclusions)
- [LICENSE summaries and drift checks](#license-summaries-and-drift-checks)
- [Safety checklist](#safety-checklist)
- [Common failures](#common-failures)

## How resolution works

`dependency.files` is a list of manifests. Each file is offered to the resolvers in order; the first resolver that claims it handles it:

| Resolver order | File pattern | Toolchain |
| --- | --- | --- |
| 1 | `*.mod` (typically `go.mod`) | `go` |
| 2 | `package.json` in a pnpm project | `pnpm` |
| 3 | other `package.json` | `npm` |
| 4 | `pom.xml` | `mvnw` or `mvn` |
| 5 | `*.jar` (glob supported) | none (reads the archive) |
| 6 | `Cargo.toml` | `cargo` |
| 7 | `Gemfile.lock` | none (reads the lockfile; network for licenses) |
| 8 | `*.gemspec` | none (reads the gemspec; network for licenses) |

If no resolver claims a file, the command fails with `unable to find a resolver to resolve dependency declaration file`. Relative `files` entries are resolved against the config file's directory, but several resolvers then `chdir` into the manifest's directory, so run the command from the repository root or a consistent location.

Resolution gathers direct and transitive dependencies. Whether a dependency is included depends on the ecosystem's notion of distribution: production dependencies for Node, compile/runtime scopes for Maven, module dependencies for Go, and so on.

## Ecosystem behavior

### Go (`go.mod`)

Commands executed:

```bash
go mod download           # or: go mod download -modfile <name>
go mod download -json     # same, JSON output
```

- Requires the `go` toolchain on `PATH` and network access for modules not already cached.
- The main module itself is not reported; only its dependencies.
- For each module, the resolver reads the module directory and walks upward looking for `LICENSE`, `LICENCE`, or `COPYING` (with optional `.md`/`.txt`), then identifies the license from the text.
- Local `replace` directives and vendored code may not be represented in the `go.mod` module list; verify the report against `go list -m all` when completeness matters.
- Side effect: populates the Go module cache. No source files are modified.

### Maven (`pom.xml`)

Commands executed:

```bash
./mvnw --version || mvn --version
./mvnw help:evaluate -Dexpression=settings.localRepository -q -DforceStdout
./mvnw dependency:resolve
# if dependency:resolve fails:
./mvnw clean install -Dcheckstyle.skip=true -Drat.skip=true -Dmaven.test.skip=true
./mvnw dependency:tree -DoutputFile=<temp>
```

- Prefers `./mvnw` when present, otherwise `mvn`.
- `os.Chdir` to the POM's directory before running Maven. Do not interleave multi-module resolution with commands that depend on the original working directory.
- A failed `dependency:resolve` triggers a full **`clean install -DskipTests`**. That is a real build: it writes `target/`, installs artifacts into the local Maven repository, and can take a long time. Run it in CI or a disposable checkout, not on a dirty working tree you care about.
- Excluded scopes: `test`, `provided`, and `system` are skipped automatically. `compile` and `runtime` are included.
- Licenses are read from the dependency's JAR (license files, then `META-INF/MANIFEST.MF`) and, failing that, from its POM (`<licenses>` or header comments).
- `excludes[].recursive: true` is supported only for Maven; it removes the named artifact's transitive tree as well.

### JAR (`*.jar`)

- The pattern is glob-expanded, so `libs/*.jar` works.
- The resolver opens each JAR, looks for a license-like file (`license`, `licence`, `copyright`, `copying`, case-insensitive), then falls back to `META-INF/MANIFEST.MF` for a license URL.
- Reports the JAR's base name as the dependency name; version is `Unknown` unless the caller supplied one.
- No external toolchain, but a shaded JAR with an embedded LICENSE may report the embedded component's license.

### npm (`package.json`)

A project is resolved with npm unless pnpm is detected (see below).

Commands and behavior:

- May run `npm ci` if the five-second skip prompt times out or receives anything other than `s`/`S`.
- Always runs `npm prune --production` (silently) before listing.
- Runs `npm ls --all --production --parseable` to enumerate installed production packages.
- Reads each package's `package.json` license field and license files from `node_modules`.

Side effects and caveats:

- **`npm prune --production` removes devDependencies from `node_modules`.** After a local dependency check, run `npm ci` again if you need the dev toolchain. Prefer CI or a disposable checkout.
- `npm ci` itself wipes and reinstalls `node_modules`.
- The skip prompt is interactive: `Try to install nodejs packages in 5 seconds, press [s/S] and ENTER to skip`. In an agent session with closed stdin, it may proceed to install. Pre-install dependencies and pipe `s` if you want to guarantee the skip:

  ```bash
  printf 's\n' | license-eye dependency resolve
  ```

  The prune still runs during listing.
- Only production dependencies are checked. Dev-only packages are out of scope by design.
- Workspaces: listing walks the installed tree from the manifest's directory; pnpm workspaces are resolved once at the workspace root.

### pnpm (`package.json`)

pnpm is selected when either of these is true, checked in the `package.json`'s directory and then each parent:

- the `packageManager` field names pnpm (for example `"pnpm@10.0.0"`), or
- a `pnpm-lock.yaml` exists.

Behavior:

- Uses `pnpm licenses list` to enumerate production packages, then reads their licenses the same way as npm.
- May run `pnpm ci` (pnpm 8+) or `pnpm install --frozen-lockfile` (older pnpm) if the skip prompt is not answered.
- A pnpm workspace is resolved once for the whole workspace, regardless of how many workspace members are listed in `dependency.files`.
- Unlike npm, the listing path does not run `npm prune`; it is less destructive to `node_modules`, though the install prompt can still run a clean install.

### Cargo (`Cargo.toml`)

Commands executed:

```bash
cargo fetch
cargo metadata --format-version=1 --all-features
```

- Requires the `cargo` toolchain and network access for uncached crates.
- Reads the `license` field from package metadata; for packages without one, reads license files in the crate directory (`LICENSE`, `LICENCE`, `COPYING`, and variants).
- The `license-file` field is not used as the authoritative source.
- Side effect: populates the Cargo cache. No project files are modified.

### Ruby (`Gemfile.lock` and `*.gemspec`)

- `Gemfile.lock` should be committed; the resolver reads the locked graph from it.
- If a `.gemspec` exists in the same directory, the project is treated as a **library**: only runtime dependencies (and their transitive closure) are included; development dependencies are ignored.
- Without a `.gemspec`, the project is treated as an **app**: all dependencies in `Gemfile.lock` are considered.
- License data is read from the lockfile/installed gemspecs when available; otherwise the resolver queries the RubyGems-compatible API (gem.coop, falling back to rubygems.org). This needs network access and may be rate-limited.
- A `*.gemspec` file can also be listed directly; the resolver parses `add_dependency` / `add_runtime_dependency` and walks installed gemspecs.
- No packages are installed.

## Compatibility checking

`dependency check` determines the main license from each `header.license.spdx-id` and looks up the built-in matrix for that SPDX ID. The matrix classifies each dependency license as compatible, weak-compatible, or incompatible. Unknown licenses are reported as failures too.

Key behaviors:

- The main license must have an SPDX ID and a matrix. A missing `header` section means there are no main licenses and the command exits 0 without checking — a silent false pass.
- With multiple `header` entries, every entry's SPDX ID is checked against the graph. A monorepo with Apache and MIT modules will check both matrices.
- `--weak-compatible` / `-w` treats weak-compatible licenses (for example certain weak-copyleft licenses) as compatible. This is a legal-policy decision, not a technical one; confirm the conditions of the weak license are actually met.
- `--fsf-free` / `-f` and `--osi-approved` / `-o` add a stricter requirement on top of the matrix. Note that `-o` means "OSI-approved" here, while `-o` on `resolve` means "output directory".
- There is no per-dependency "ignore this incompatible license" flag. Use `dependency.licenses` to correct an unknown license, or `dependency.excludes` to remove a dependency that is genuinely not distributed.

## Overrides and exclusions

### Checking without failing CI on unknowns

If a dependency's license is genuinely known but not detectable (an unusual license file, a vendored copy), add an override after verifying the actual license:

```yaml
dependency:
  licenses:
    - name: github.com/example/legacy
      version: v1.2.3
      license: MIT
```

`name` accepts shell-style globs (`filepath.Match`), and `version` is a comma-separated list of exact versions; omit it for all versions. Do not use overrides to turn an unknown into a permissive license without checking the dependency's LICENSE file.

For build-only dependencies:

```yaml
dependency:
  excludes:
    - name: com.example:annotation-processor
    - name: org.apache.skywalking:*
      recursive: true       # Maven only
```

Maven's `test`, `provided`, and `system` scopes are already excluded; only add excludes for compile/runtime dependencies that are not part of the distributed artifact.

## LICENSE summaries and drift checks

`resolve --summary <template>` renders the project license plus dependency groups and writes `LICENSE` next to the template. Use `--license <path>` to choose a different output path; without `--summary`, `--license` uses a built-in default template.

The template receives:

| Variable | Meaning |
| --- | --- |
| `.LicenseContent` | Project license text (SPDX template for `spdx-id`, otherwise `content`). |
| `.Groups` | Dependency groups sorted by SPDX ID. |
| `.Groups[].LicenseID` | SPDX ID for the group. |
| `.Groups[].Deps` | Dependencies in the group, sorted by name. |
| `.Deps[].Name`, `.Deps[].Version`, `.Deps[].LicenseID` | Dependency details. |

Minimal template:

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

A CI drift check regenerates the summary and fails if the committed file changes:

```bash
license-eye dependency resolve --summary path/to/LICENSE.tpl || exit 1
if [ -n "$(git diff -U0 -- path/to/LICENSE)" ]; then
  echo "LICENSE is not updated; commit the regenerated file"
  git diff -U0 -- path/to/LICENSE
  exit 1
fi
```

Caveats:

- **Resolve on the same OS as CI.** Some ecosystems resolve platform-specific dependency variants (notably native Node packages and some Go build constraints). A diff produced on macOS/Windows may be a platform artifact, not a stale LICENSE. Regenerate on Linux when CI is Linux.
- A summary requires a single `header` entry. With multiple entries, `resolve --summary` fails with `unable to write summary as multiple licenses were provided in configuration`.
- The generated file is deterministic for a given dependency graph and OS; do not hand-edit it. Change the template or the graph instead.
- Commit the template and the generated LICENSE. Without the template, other contributors cannot reproduce the file.

## Safety checklist

Before running `dependency resolve` or `dependency check` locally, confirm:

1. **Working tree state.** Dependency checks can build Maven projects and prune npm packages. Commit or stash unrelated work; prefer a disposable checkout for Maven and npm.
2. **Toolchain.** The required package manager is installed and on `PATH`. The license-eye binary alone cannot resolve dependencies.
3. **Network.** Go, Maven, Cargo, and Ruby resolution may download modules or query APIs. Restricted CI runners need a proxy or a pre-populated cache.
4. **Node dev dependencies.** After an npm run, `node_modules` may be production-only. Run `npm ci` to restore dev dependencies.
5. **Interactivity.** npm/pnpm resolution waits five seconds for a skip keypress. In an agent session, pipe `s` if dependencies are already installed, or expect an install.
6. **Working directory.** Go, Maven, and Node resolvers `chdir` into the manifest directory. Do not depend on the original cwd afterwards in the same process; separate CLI invocations are fine.
7. **CI parity.** Generate committed LICENSE summaries on the same OS as CI.

## Common failures

| Message / symptom | Cause | Fix |
| --- | --- | --- |
| `unable to find a resolver to resolve dependency declaration file` | Unsupported file name or a typo in `dependency.files`. | Use one of the supported patterns. Relative paths are relative to the config file. |
| `neither found mvnw nor mvn` | Maven not installed and no wrapper. | Install Maven or add `mvnw`. |
| `dependencies download error` | `mvn dependency:resolve` failed, and the fallback build also failed. | Run Maven manually, fix the build, then retry. |
| `cannot find license file` | The resolver found the package but no LICENSE/COPYING file. | Verify the actual license and add a `dependency.licenses` override, or exclude an undistributed dependency. |
| `cannot identify license content` | A license file exists but does not match a known license with enough confidence. | Add an override after checking the file; optionally lower `dependency.threshold` (default 75) only if the file is a partial license. |
| `Unknown` dependencies in the report | Same as above, or a package manager could not install/list packages. | Inspect the dependency, add an override, or fix the toolchain/network. |
| `the following licenses are unknown or incompatible with the main license` | `dependency check` found unknown or incompatible licenses. | Resolve the specific dependency: override, exclude, or adjust policy. |
| No output, exit 0 | No `header` section, or an unsupported/empty graph. | Add `header` with `spdx-id`; verify `dependency.files`. |
| Node resolution destroyed dev dependencies | `npm prune --production` ran. | `npm ci`; run dependency checks in CI next time. |
| Maven build wrote `target/` or installed artifacts | The resolver fell back to `clean install`. | Run in CI or a clean checkout; clean the workspace afterwards. |
| LICENSE differs only on another OS | Platform-specific dependency variants. | Regenerate on the CI OS. |
| Ruby resolution times out or returns Unknown | RubyGems API unreachable or rate-limited. | Ensure network access, or add overrides based on the committed lockfile/gemspecs. |
