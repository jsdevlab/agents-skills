# Prereleases and snapshot releases

Both features produce non-stable versions, but for different purposes:

- **Prerelease mode** is a sustained alpha/beta/rc cycle that eventually becomes a stable release. State lives in `.changeset/pre.json`.
- **Snapshot releases** are one-off test builds from a branch or PR. They are never merged back as version changes.

Prereleases remove many safety rails. Read this whole file before entering prerelease mode, and confirm with the user first — recovering from a bad prerelease state is difficult.

## Prerelease mode

### How it works

`changeset pre enter beta` puts the repository into prerelease mode. Subsequent `changeset version` calls produce versions such as `1.2.0-beta.0`, and `changeset publish` publishes them under the `beta` dist-tag (never `latest` for existing packages). `changeset pre exit` marks the intent to return to stable; the next `version` produces normal versions.

`.changeset/pre.json` stores the state:

```json
{
  "mode": "pre",
  "tag": "beta"
}
```

After `pre exit` it becomes `"mode": "exit"` until the stable version runs.

### Branch strategy

Two common approaches:

1. **Prerelease on the default branch, backport branch for stable.** Before entering, create a branch from the last stable tag (e.g. `v1`) to receive critical fixes. Changesets documents this as the default pattern.
2. **Prerelease on a separate branch (e.g. `next`).** Set `baseBranch` to `next` in `.changeset/config.json` while in prerelease mode, allow the release workflow to run on that branch, and merge `main` into `next` periodically. Before the stable release, set `baseBranch` back to `main`, run `pre exit`, commit, and merge back.

### Before entering

Consider making one last stable release to clear pending changesets. Otherwise they are included in the first prerelease.

```bash
changeset version
changeset publish
changeset pre enter beta
git add .changeset/pre.json
git commit -m "Enter beta prerelease mode"
```

### Releasing prerelease versions

Run the normal flow:

```bash
changeset version
git add .
git commit -m "Version packages (beta)"
changeset publish --tag beta   # the CLI uses the prerelease tag automatically, but --tag is explicit
```

In CI with `changesets/action`, the version PR title is suffixed with `(beta)`.

Dependents may be bumped even when they would not be in a stable release: prerelease versions generally do not satisfy normal ranges. For example, if `pkg-b` goes from `2.0.0` to `2.1.0-beta.0`, a dependent `pkg-a` with range `^2.0.0` gets a patch bump and its range updates.

```text
pkg-a @ 1.0.1-beta.0  depends on pkg-b ^2.1.0-beta.0
pkg-b @ 2.1.0-beta.0
```

### Changing the prerelease tag

During a cycle you may move from `alpha` to `beta` to `rc`. Edit `.changeset/pre.json` directly:

```json
{ "mode": "pre", "tag": "beta" }
```

No branch rename is needed.

### Managing changesets during prerelease

Versioned changesets are not deleted; they move to `.changeset/pre/`. These files provide the changelog entries for the eventual stable release. Before exiting, review them:

- Keep entries that matter to stable consumers.
- Edit or delete prerelease-only fixes (e.g. a bug introduced and fixed within the beta) so they do not pollute the stable changelog.
- Deleting all of them is safe; the stable release still works, just without those changelog entries.

### Exiting and releasing stable

```bash
changeset pre exit
git add .
git commit -m "Exit prerelease mode"
changeset version
git add .
git commit -m "Version packages"
changeset publish
```

For a separate prerelease branch, revert `baseBranch` and CI branch filters before merging back.

### Prerelease pitfalls

- You cannot prerelease only a subset of packages; the whole repository enters the mode.
- A **new, never-published package** published during prerelease mode still gets the `latest` dist-tag, because npm requires every package to have one. Plan initial publishes carefully.
- Safety rails around ignored packages and dependency ranges still apply, but many versioning assumptions are relaxed. Review diffs closely.
- If you publish to a registry other than npm, prerelease dist-tag behavior differs; v3 uses the configured prerelease tag on registries that do not auto-assign `latest`.

## Snapshot releases

### When to use

Snapshots are for testing a branch or PR without consuming changesets or choosing a real next version. Versions look like `0.0.0-{tag}-{datetime}` (by default) and are not intended for production.

Alternative: services such as `pkg.pr.new` publish ephemeral builds to their own registry, avoiding npm pollution entirely.

### Workflow

1. Create changesets as usual (snapshots consume the pending changesets into version numbers).
2. Use a dedicated branch; do not merge the snapshot version changes back.
3. Produce and publish:

```bash
changeset version --snapshot mytag
changeset publish --tag mytag --no-git-tag
```

`--snapshot` with no name produces `0.0.0-{datetime}`. With a name it produces `0.0.0-mytag-{datetime}`.

4. Tell testers to install by exact version or dist-tag:

```bash
pnpm add your-package@mytag
# or
pnpm add your-package@0.0.0-mytag-20261109000000
```

### Critical rules

- **Always pass a `--tag` when publishing a snapshot.** Without one, the snapshot becomes `latest` and every new install gets the test build. If you did not name the snapshot, use a neutral tag such as `--tag snapshot`.
- **Do not push or merge the snapshot version commit** into a branch you plan to release from. The snapshot is for installation, not for representing the repository's released state. Keep the version and tag you generated for testers.
- Use `--no-git-tag` to avoid creating permanent tags for temporary builds.

### Template and version base

Configure in `.changeset/config.json`:

```json
{
  "snapshot": {
    "useCalculatedVersion": false,
    "prereleaseTemplate": "{tag}-{datetime}"
  }
}
```

Placeholders:

| Placeholder | Value |
| --- | --- |
| `{tag}` | name passed to `--snapshot` |
| `{commit}` | full 40-character git SHA |
| `{commit-short}` | first 7 characters of the SHA |
| `{timestamp}` | Unix timestamp (`Date.now()`) |
| `{datetime}` | `YYYYMMDDHHMMSS` |

`useCalculatedVersion: false` (default) bases snapshots on `0.0.0` so they never satisfy normal ranges. Setting it to `true` uses the calculated next version — useful only if you understand the range-resolution tradeoff (a snapshot could satisfy `^1.0.0`).

### Snapshot releases in CI

Snapshots work well as a PR-triggered workflow:

```yaml
- run: changeset version --snapshot pr-${{ github.event.pull_request.number }}
- run: changeset publish --tag pr-${{ github.event.pull_request.number }} --no-git-tag
```

If the workflow runs on `main` for nightlies, use a tag like `nightly` and the same `--tag`/`--no-git-tag` rules.

### Snapshot pitfalls

- Publishing a snapshot from a branch that also contains a brand-new package can publish that package for the first time; set `"private": true` where appropriate.
- Running `changeset version --snapshot` consumes the changeset files. Keep this on a throwaway branch/commit so the real release is unaffected.
- `version` exits 1 if there are no changesets; snapshot jobs must tolerate that.
