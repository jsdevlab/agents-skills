# Changesets CLI reference

The CLI is `@changesets/cli` (v3 at the time of writing). Run it through the project's package manager:

| Manager | Invocation |
| --- | --- |
| pnpm | `pnpm changeset <command>` |
| npm | `npx @changesets/cli <command>` |
| yarn | `yarn changeset <command>` |

Requirements for v3: Node.js `^22.11 || ^24 || >=26`; pnpm `>=10`, npm `>=10.9`, or yarn `>=4.5.2`.

## Contents

- [init](#init)
- [add (default)](#add-default)
- [version](#version)
- [publish](#publish)
- [publish-plan](#publish-plan)
- [pack](#pack)
- [status](#status)
- [git-tag](#git-tag)
- [pre](#pre)
- [Exit codes and non-interactive use](#exit-codes-and-non-interactive-use)

## init

```bash
changeset init
```

Creates `.changeset/` with a README and config. **Interactive in v3** — it asks whether to use the GitHub changelog integration. In an automated session it will block or fail on stdin. Create `.changeset/config.json` manually (see `config.md`) or ask the user to run `init` in a terminal.

## add (default)

```bash
changeset [add] [options]
```

Creates a changeset and writes it to `.changeset/<random-name>.md`. With no flags it is fully interactive: choose packages, choose bump types, write a summary, confirm.

Options:

| Option | Meaning |
| --- | --- |
| `--major <pkg>` | Comma-separated package(s) to major bump |
| `--minor <pkg>` | Comma-separated package(s) to minor bump |
| `--patch <pkg>` | Comma-separated package(s) to patch bump |
| `-m, --message <text>` | Summary for the changeset body |
| `--empty` | Create an empty changeset (no releases) |
| `--since <ref>` | Detect changed packages since a branch/tag/ref instead of `baseBranch` |
| `--open` | Open the generated file in the editor |

Examples:

```bash
# One package, one bump
changeset add --patch "@scope/pkg-a" -m "Fix crash on empty input"

# Several packages with different bumps in one changeset
changeset add --minor "@scope/pkg-a" --patch "@scope/pkg-b" -m "Add streaming; fix pkg-b"

# Same bump for several packages
changeset add --patch "@scope/pkg-a,@scope/pkg-b" -m "Update shared parser"

# CI-only change in a repo that requires a changeset
changeset add --empty -m "Update CI configuration"

# Suggest packages changed since a non-default branch
changeset add --since next
```

Package names must match `package.json` exactly. Flags can be combined; a package must not appear at two bump levels in the same invocation.

When the repo has no git history or the base branch is missing, `add` prints a warning about detecting changed packages but still writes the changeset when explicit flags are supplied.

## version

```bash
changeset version [options]
```

Consumes all pending changesets, updates package versions, updates internal dependency ranges, writes `CHANGELOG.md` files, and deletes the consumed changeset files. It does not publish.

Options:

| Option | Meaning |
| --- | --- |
| `--ignore <pkg>` | Skip packages; repeatable. Same safety rails as the `ignore` config |
| `--snapshot [name]` | Produce snapshot versions instead of normal versions |
| `--snapshot-prerelease-template <tpl>` | Override `snapshot.prereleaseTemplate` for this run |

Important behavior:

- **Exit code 1 when there are no unreleased changesets.** This is expected in v3. Do not treat it as a tool failure.
- No `--dry-run` exists. Preview with `changeset status --output <temp-file>` or run on a throwaway branch.
- Commit the result before publishing; `publish` assumes the last commit is the version commit.
- Empty changesets consume no version bump.
- In prerelease mode, changesets are moved to `.changeset/pre/` instead of deleted.

## publish

```bash
changeset publish [options]
```

For each workspace package, compares the local version with the registry and publishes versions that are not already published. Creates git tags `pkg-name@X.X.X` (`vX.X.X` in a single-package repo) by default.

Options:

| Option | Meaning |
| --- | --- |
| `--tag <name>` | Publish with a custom npm dist-tag (prerelease, snapshot, backport) |
| `--otp <code>` | One-time password for npm 2FA |
| `--from-pack-dir <dir>` | Publish from tarballs created by `pack` |
| `--no-git-tag` | Do not create git tags (help lists `--git-tag`, default true) |

Warnings:

- `publish` publishes any local version missing from the registry. Set `"private": true` on packages that must never publish.
- It assumes the last commit is the version commit. Do not commit between `version` and `publish`.
- First public publish of a scoped package needs `access: "public"` (config or `publishConfig`).
- Push tags afterwards: `git push --follow-tags`.
- Publishing is irreversible; confirm with the user first.

## publish-plan

```bash
changeset publish-plan [--output <file>]
```

Lists packages ready to publish or tag. `--output` writes JSON for `pack`/`publish` to consume (experimental format, stable within one CLI version). Useful in CI to split version and publish.

## pack

```bash
changeset pack --out-dir <dir>
changeset pack --from-publish-plan <publish-plan.json> --out-dir <dir>
```

Packs publishable packages into tarballs. The output directory can be passed to `publish --from-pack-dir <dir>`. `--from-publish-plan` reads the JSON produced by `publish-plan --output`, so CI can split the version, pack, and publish steps. Useful in CI to separate build and publish steps.

## status

```bash
changeset status [--since <ref>] [--verbose] [--output <file>]
```

Shows existing changesets and the releases they imply. If packages changed but no changesets exist, exits 1. `--output` writes JSON for other tools.

Warnings:

- Requires git and a resolvable `baseBranch`. In a shallow checkout or non-git directory it fails with `Failed to find where HEAD diverged from "<branch>"`.
- Fails while `version` or `publish` is mid-run.
- If you need status at release time, capture it immediately before `version`.

## git-tag

```bash
changeset git-tag
```

Creates git tags for the current versions without publishing to npm. Run `version` first so tags match the release. Useful when another tool publishes packages (or when packages are private).

## pre

```bash
changeset pre enter <tag>
changeset pre exit
```

Enters or exits prerelease mode for all packages; does not version or publish by itself. `pre enter beta` writes `.changeset/pre.json` and makes subsequent `version` produce `X.Y.Z-beta.N`. `pre exit` marks an intent to return to stable; run `version` and `publish` afterwards.

Read `prereleases-and-snapshots.md` before using this. Prerelease mode removes many safety rails.

## Exit codes and non-interactive use

- Interactive commands: `init`, `add`/`changeset` with no flags. They need a TTY.
- Non-interactive commands: `version`, `publish`, `status`, `publish-plan`, `pack`, `git-tag`, `pre enter/exit`, and `add` with explicit bump flags and `-m`.
- `version` exits 1 when no changesets; `status` exits 1 when changed packages lack changesets; `publish` exits non-zero on registry/auth errors.
- In scripts, treat "no changesets" as a normal outcome. For example:
  ```bash
  changeset version || [ $? -eq 1 ] || exit $?
  ```
  Prefer checking the message or running `status --output` rather than blind `|| true`, which would hide real failures.
