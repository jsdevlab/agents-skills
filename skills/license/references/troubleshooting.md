# Troubleshooting license-eye

Start by collecting the facts. Most failures come from the config, the working directory, or an ecosystem resolver doing something the user did not expect.

```bash
license-eye --version
pwd
git status --short
sed -n '1,200p' .licenserc.yaml
license-eye -v debug header check 2>&1 | head -80
license-eye -v debug dependency resolve 2>&1 | head -80
```

`-v debug` prints the normalized license, the normalized pattern, the files being considered, and resolver commands. Redirect it when the repository is large.

## Contents

- [False failures: the header is there but check fails](#false-failures-the-header-is-there-but-check-fails)
- [False passes: check succeeds without checking](#false-passes-check-succeeds-without-checking)
- [Fix problems](#fix-problems)
- [Configuration problems](#configuration-problems)
- [Dependency resolution problems](#dependency-resolution-problems)
- [CI and GitHub Actions problems](#ci-and-github-actions-problems)
- [Git and filesystem problems](#git-and-filesystem-problems)

## False failures: the header is there but check fails

### Symptom

`header check` reports a file as invalid, but the file visibly has the configured license.

### Run `header diff` first

```bash
license-eye header diff
```

The output uses the same normalization as `check`:

```text
src/main.go:
    ... copy of the license at [-http://www.apache.org/licenses/license-2.0-] {+http://www.apache.org/licenses/license-2.1+} ...
```

`[-...-]` is expected but missing; `{+...+}` is present but unexpected. Common causes:

- A typo in the license URL, copyright holder, or year.
- A different license version or variant (Apache-2.0 vs. Apache-2.0-ASF, MIT vs. MIT-0).
- Extra text, a legacy disclaimer, or a different line wrapping that normalization does not flatten away.

Fix the file, or set `license.pattern` to accept the variant. If the file's header is an old version that should be replaced, set `pattern` to match it and run `fix`; without `pattern`, `fix` prepends the new header and leaves the old one in place.

### The header is too far down

`license-location-threshold` is the maximum index of the license in the normalized file content, not a line number. Default is 80. A long shebang, an encoding declaration, a generated preamble, or a large comment block before the license can exceed it.

- For files that cannot avoid a preamble, raise the threshold:

  ```yaml
  header:
    license-location-threshold: 200
  ```

- Otherwise, move the license to the top. Raising the threshold globally weakens the check for every file.

### The file has a BOM or a non-UTF-8 encoding

`check` reports undecodable files as failures. Convert the file to UTF-8 (with or without BOM is supported) and re-run. `fix` refuses files it cannot decode, and the command exits non-zero after fixing the rest.

### Multiple header entries conflict

If `header` is a list, every entry is checked against its own `paths`. An entry with no `paths` defaults to `**`, so it checks every file and will fail files that legitimately carry a different module's license. Scope every entry:

```yaml
header:
  - license:
      spdx-id: Apache-2.0
    paths: ['module-a/**']
  - license:
      spdx-id: MIT
    paths: ['module-b/**']
```

The `path:` key is not recognized in current releases; use `paths`. With `path:`, both entries apply everywhere and most files fail one of the checks.

### The config file itself is invalid

`paths: ['**']` includes `.licenserc.yaml`. The config file must carry the header or be excluded. Either:

- run `license-eye header fix .licenserc.yaml`, or
- add `.licenserc.yaml` to `paths-ignore`.

### Windows line endings or a different encoding

The normalizer flattens whitespace and handles CRLF, so line endings alone are rarely the cause. A UTF-16 or legacy-encoded file is. Convert to UTF-8.

## False passes: check succeeds without checking

### No `.licenserc.yaml`

`license-eye` does not fail when the config is missing. It loads a built-in Apache-2.0/ASF config. The check can pass because the files already have some Apache header, even though the project license or copyright owner is wrong. Always verify the config exists and matches the project.

### No `header` section

A config with only `dependency:` parses into zero header entries. Both `header check` and `dependency check` then iterate over nothing and exit 0. This is the most common silent false pass.

Fix: add a `header` section with `spdx-id`. If dependency-only checking is intended, add `paths-ignore: ['**']` under that header so the header checker ignores every file while `dependency check` still has a main license.

### `paths` or `paths-ignore` are too narrow

- `paths` defaults to `**`; if set to a subdirectory, files outside it are never checked.
- `paths-ignore` wins over `paths`. A broad ignore such as `**` or `**/*` makes the check a no-op.
- In a Git repository, only tracked files plus working-tree changes are considered. A file that is not tracked and not visible in `git status` (for example an ignored generated file) is skipped.

Verify with `-v debug`: it logs each file considered and each ignored file.

### Binary files are skipped

`check` sniffs files without a BOM and skips anything whose detected MIME type is not `text/*`. A source file with an unusual encoding can be silently skipped rather than failed. Compare the checked count with the expected file count.

### `dependency check` has no main license

If there is no `header` section or no `spdx-id`, `dependency check` has nothing to compare against and exits 0. Add a header with a real SPDX ID.

## Fix problems

### `fix` does nothing

Likely causes:

1. The file is excluded by `paths` or `paths-ignore`.
2. The file is not tracked and not shown by `git status` (ignored). Add it or adjust the config.
3. The file is not in the current working directory's glob scope. Run from the repository root.
4. The file is already valid according to the configured header; `fix` only touches invalid files.

Run `license-eye -v debug header fix` and check the "Checking file" lines.

### `fix` reports `unsupported file: <path>`

The extension or filename has no comment style. Examples: `.json`, some generated file types, or a new language not in the upstream `assets/languages.yaml`. Options:

- Exclude the file (`paths-ignore`) if it should not carry a header.
- Add a `header.language` override with an existing `comment_style_id`.
- Convert the file to a supported text type, if appropriate.

`fix` still fixes other files and exits non-zero because of the unsupported one. Check the summary for `fixed: N` and the error list.

### `fix` inserts an empty comment (`//` or `#` with no text)

The configured `spdx-id` has no built-in header template. Current releases only ship templates for Apache-2.0, MIT, EPL-2.0, MPL-2.0, GPL-3.0-or-later, AGPL-3.0-only/or-later, and MulanPSL-2.0. For any other SPDX ID, provide `license.content`.

This is a silent-corruption case: the command reports `fixed: N` and exits 0. Check the inserted header after fixing a project with an uncommon SPDX ID.

### `fix` duplicated the header

There was an old header that did not match the configured license, and no `pattern` was set to remove it. Set `pattern` to a distinctive part of the old header, then run `fix` again. Verify the pattern does not match real code before running it.

### `fix` deleted more than the header

The `pattern` was too broad. Restore from version control, narrow the pattern to a distinctive license phrase, and re-run. Keep the regexp anchored and avoid wildcards that span into code.

## Configuration problems

### `spdx-id` is not enough

`spdx-id` only selects a built-in template for the small set listed above. For other IDs, `check` fails with a warning about a missing template, and `fix` inserts an empty header. Set `content`.

### ASF header appears unexpectedly

The missing-config default uses `spdx-id: Apache-2.0` and `copyright-owner: Apache Software Foundation`. The ASF-like owner selects the ASF contributor header. Create a real config with the project's owner.

### `pattern` never matches

`pattern` is normalized without stripping comment markers. If it contains `//`, `#`, or `/*`, it will not match the normalized file content. Remove comment markers and escape regexp metacharacters.

### `license-location-threshold` hides a problem

Lowering the threshold makes checks stricter and can reject legitimate headers; raising it accepts headers deeper in the file. Change it for a reason, not to silence a failure.

### YAML shape is wrong

`header` can be a mapping or a list, but the fields inside each entry must match the schema. A list entry with `license`, `paths`, `paths-ignore`, `comment`, `license-location-threshold`, and `language` is valid. Unknown keys such as `path` are ignored silently. Validate with `license-eye -v debug header check`; malformed YAML fails config loading.

## Dependency resolution problems

### `unable to find a resolver to resolve dependency declaration file`

The `dependency.files` entry does not match any resolver. Supported names: `*.mod`, `package.json`, `pom.xml`, `*.jar`, `Cargo.toml`, `Gemfile.lock`, `*.gemspec`. Check for typos and note that relative paths are resolved from the config file's directory.

### `no module dependencies to download` / empty report

- The manifest has no dependencies.
- The resolver is looking at a different manifest or workspace root.
- For Go, local `replace` directives and the main module are not reported.
- For Node, only production dependencies are included; a project with only devDependencies reports nothing.

### Unknown licenses

`Unknown` means the resolver found the dependency but could not identify a license with sufficient confidence. Steps:

1. Locate the dependency's license file (`node_modules/<pkg>/LICENSE`, the module cache, the Maven local repository, the crate directory, etc.).
2. Read it. Do not assume the license from the package name.
3. Add a verified override:

   ```yaml
   dependency:
     licenses:
       - name: <dependency-name>
         version: <exact-version-or-comma-list>
         license: <SPDX-ID>
   ```

4. Re-run and confirm the dependency moves from `Unknown` to the expected license.

If the dependency is not actually distributed, add it to `excludes` instead. This is a policy decision; record the reason.

### Incompatible licenses

The report names the main license and the incompatible dependency. Options, in order of preference:

1. Replace or remove the dependency.
2. Exclude it if it is build/test-only and not distributed.
3. Accept weak-compatible licenses only after confirming the license conditions (`--weak-compatible`).
4. Change the project license (rarely appropriate; a legal decision).

Do not add an override that lies about the license to silence the check.

### Maven: `neither found mvnw nor mvn`

Install Maven or add the Maven wrapper. On CI, `actions/setup-java` provides Maven; on a minimal container, install it explicitly.

### Maven: build runs and writes `target/`

The resolver falls back to `mvn clean install -DskipTests` when `dependency:resolve` fails. This is expected. Run dependency checks in CI or a disposable checkout. If the build itself is broken, fix it first; the resolver cannot compensate.

### Maven: wrong directory or relative paths break

The resolver `chdir`s into the POM's directory. If `dependency.files` lists multiple manifests, each resolver changes the process working directory. Separate CLI invocations are safer than relying on one process for multiple modules.

### npm: devDependencies disappeared

`dependency resolve` / `check` runs `npm prune --production` before listing packages. This removes devDependencies from `node_modules`. Reinstall them:

```bash
npm ci
```

Run dependency checks in CI or a disposable checkout to avoid this. pnpm resolution does not use this prune path.

### npm/pnpm: command pauses for five seconds

The resolver asks `Try to install nodejs packages in 5 seconds, press [s/S] and ENTER to skip`. In an agent session with closed stdin, it may proceed to install. Pre-install dependencies and pipe `s` if they are already present:

```bash
printf 's\n' | license-eye dependency resolve
```

The npm prune still runs during listing.

### pnpm: project is resolved with npm

pnpm is detected by a `packageManager` field naming pnpm or a `pnpm-lock.yaml` in the manifest's directory or a parent. Add one of those, or use the npm path deliberately. In a pnpm workspace, the lockfile is usually at the workspace root; point `dependency.files` at the root `package.json`.

### Cargo: `cargo: command not found` or a metadata error

Install Rust and make `cargo` available. `cargo fetch` and `cargo metadata --all-features` run first; a feature-gated dependency that fails to resolve will fail the whole check.

### Ruby: slow or unknown licenses

The resolver queries gem.coop / rubygems.org when the lockfile or installed gemspec does not declare a license. Ensure network access. For libraries, a `.gemspec` in the same directory as `Gemfile.lock` changes the dependency set to runtime-only; verify that is the intended policy.

### Summary generation fails

`unable to write summary as multiple licenses were provided in configuration` — the config has a list of `header` entries. Summary generation needs exactly one. Use a separate config for the summary job, or generate per module.

### LICENSE drift on one OS only

Platform-specific dependency variants differ between Linux, macOS, and Windows. Generate the committed LICENSE on the same OS as CI (usually Linux). If a local diff appears only on one platform, it may be an artifact; compare the dependency lists before committing.

## CI and GitHub Actions problems

### The action cannot be found or fails to build

- Pin an existing tag or commit: `apache/skywalking-eyes/header@v0.9.0`. Do not use a branch that may not contain the action at the expected path.
- The action builds with Go; a runner without network access cannot download modules.
- On self-hosted runners, `make install` copies to `/usr/local/bin`; ensure the runner user can write there.

### PR comments are not posted

Check all of:

- The workflow runs on a `pull_request` event.
- `permissions: pull-requests: write` is set.
- The config has `comment: on-failure` or `comment: always`.
- The action input `token` is not empty.
- The PR comes from the same repository. Fork PRs have a read-only token; comments may be rejected.

Set `comment: never` or `token: ""` if comments are not wanted; this also lets you keep `permissions: contents: read`.

### Fix mode passes even though headers were missing

That is expected. `header fix` exits 0 when it fixes the files. Use `header check` as the CI gate. If you want both, run `check` in a required job and `fix` in a separate bot workflow.

### Dependency job fails to find the toolchain

The action only provides `license-eye`; it does not install Go, Maven, Node, Cargo, or Ruby. Add the corresponding setup step before the action, and ensure it runs before the dependency step.

### `flags` input breaks the command

The dependency action interpolates `flags` directly into the shell command. Use simple flags (`--weak-compatible`, `--summary=path/LICENSE.tpl`) and avoid quotes, semicolons, or untrusted values.

### Action version vs. CLI version

If a behavior differs from this skill's examples, check `license-eye --version` and the action tag. Behavior has changed across releases (for example, `resolve` exit codes and pnpm support). Match the documented version, or upgrade deliberately and re-test.

## Git and filesystem problems

### `failed to read git repository. Run 'git fsck' ...`

The repository has dangling or corrupt Git objects, and the file walker cannot read the tree. Follow the message:

```bash
git fsck
git prune
git gc --prune=now --aggressive
```

Re-run the check afterwards. In a shallow or unusual clone, the glob fallback may be more reliable; running outside a Git worktree disables the Git-based file discovery.

### Files outside the Git repository are not checked

In a Git repository, discovery starts from the Git index and working-tree status. Files outside the repository, or files not visible to `git status`, are skipped. Run the tool from inside the intended repository, or use a non-Git directory if glob-based discovery is desired.

### Running from a subdirectory changes behavior

`header.paths` is relative to the working directory, and `git.PlainOpen("./")` looks for a repository at the current directory. Run from the repository root, or from the directory containing the config, and keep the config's paths consistent with that location.

### Permission denied while fixing

`fix` writes files in place. Ensure the process user can write the files. On Windows, a file locked by an editor or another process can cause a write failure; close it and retry.
