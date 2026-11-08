# Changesets configuration reference

Changesets reads `.changeset/config.json` from the repository root. `changeset init` creates it through an interactive prompt (v3 asks about GitHub changelog integration), so in an automated session create the file directly or ask the user to run `init` in a terminal.

Default v3 configuration:

```json
{
  "changelog": "@changesets/cli/changelog",
  "commit": false,
  "fixed": [],
  "linked": [],
  "access": "restricted",
  "baseBranch": "main",
  "changedFilePatterns": ["**"],
  "format": "auto",
  "privatePackages": { "version": false, "tag": false },
  "updateInternalDependencies": "patch",
  "ignore": [],
  "bumpVersionsWithWorkspaceProtocolOnly": false
}
```

## Contents

- [changelog](#changelog)
- [commit](#commit)
- [fixed](#fixed)
- [linked](#linked)
- [access](#access)
- [baseBranch](#basebranch)
- [changedFilePatterns](#changedfilepatterns)
- [format](#format)
- [privatePackages](#privatepackages)
- [updateInternalDependencies](#updateinternaldependencies)
- [ignore](#ignore)
- [bumpVersionsWithWorkspaceProtocolOnly](#bumpversionswithworkspaceprotocolonly)
- [snapshot](#snapshot)
- [Experimental options](#experimental-options)
- [Recipes](#recipes)

## changelog

- **Type:** `false | string | [string, Record<string, any>]`
- **Default:** `"@changesets/cli/changelog"` (alias of `@changesets/changelog-git`)

Controls changelog generation during `version`. `false` disables changelogs. A string names an npm package or a path relative to the config file. A tuple passes options to the generator.

The default generator adds commit links. For GitHub compare links and contributor mentions, use `@changesets/changelog-github`:

```json
{
  "changelog": ["@changesets/changelog-github", { "repo": "owner/repo" }]
}
```

A custom generator default-exports `getReleaseLine(changeset, type, options)` and `getDependencyReleaseLine(changesets, dependenciesUpdated, options)`. Both may return a string or a promise. See [Customize Changelog Format](https://changesets.dev/guide/customize-changelog-format).

## commit

- **Type:** `boolean | string | [string, Record<string, any>]`
- **Default:** `false`

When enabled, `add` and `version` commit their changes automatically. `true` uses the default generator, which produces `docs(changeset): {summary}` for `add` and a `RELEASING: Releasing N package(s)` message for `version`, with `skipCI: "version"`.

A custom generator can implement `getAddMessage(changeset, options)` and `getVersionMessage(releasePlan, options)`. See [Customize Commit Format](https://changesets.dev/guide/customize-commit-format).

Be careful with `skipCI` and `[skip ci]`: if version commits skip CI, the workflows that publish may never run.

## fixed

- **Type:** `string[][]`
- **Default:** `[]`
- **Monorepos only.** Supports picomatch patterns.

Packages in a fixed group are always version-bumped and published together, and always share the same version — even packages with no changes. This is the right choice when packages must stay in lockstep.

```json
{
  "fixed": [["pkg-a", "pkg-b"], ["@scope/*"]]
}
```

Example: `pkg-a` and `pkg-b` are both `1.0.0`. A changeset bumps `pkg-a` minor and `pkg-b` patch; the release makes both `1.1.0`. If later only `pkg-a` gets a minor changeset, both still become `1.2.0`.

Do not use `fixed` merely to keep versions aligned; use `linked` if only packages with changesets should be bumped.

## linked

- **Type:** `string[][]`
- **Default:** `[]`
- **Monorepos only.** Supports picomatch patterns.

Linked packages share version numbers, but are only bumped when they have a changeset (directly or as a dependent). A linked package versions from the highest current version in the group and the highest bump type among the group's changesets.

```json
{
  "linked": [["pkg-a", "pkg-b"], ["@scope/*"]]
}
```

Example: `pkg-a` and `pkg-b` are at `1.0.0`; a patch for `pkg-a` and minor for `pkg-b` makes both `1.1.0`. A later minor changeset for `pkg-a` makes `pkg-a` `1.2.0` but leaves `pkg-b` at `1.1.0`.

If packages must always be published together with identical versions, use `fixed`.

## access

- **Type:** `"restricted" | "public"`
- **Default:** `"restricted"`

Controls npm publish visibility. npm treats scoped packages as restricted by default, so Changesets defaults to `restricted` to avoid accidental public releases. For public packages, set:

```json
{ "access": "public" }
```

Override per package with `publishConfig.access` in `package.json`. To prevent publishing entirely, set `"private": true`.

## baseBranch

- **Type:** `string`
- **Default:** `"main"`

The branch Changesets compares against to detect changed packages. Set it to the branch changes merge into (`main`, `master`, `develop`, a backport branch, etc.). `add --since`, `status --since`, and CI release workflows rely on it.

Locally, the branch must exist and be reasonably up to date. In a shallow CI checkout, fetch it first.

## changedFilePatterns

- **Type:** `string[]`
- **Default:** `["**"]`

Picomatch patterns deciding which changed files mark a package as changed. Useful to ignore tests or docs when detecting changes:

```json
{ "changedFilePatterns": ["src/**", "lib/**"] }
```

This affects detection (`add` suggestions, `status`), not version bump math.

## format

- **Type:** `"auto" | "prettier" | "oxfmt" | "deno" | "biome" | "dprint" | false`
- **Default:** `"auto"`

Formatter applied to generated changeset files and changelogs. `"auto"` detects the project's formatter via `@changesets/format`. `false` disables formatting. Changesets edits only the relevant fields of `package.json` rather than reformatting the whole file.

## privatePackages

- **Type:** `{ version?: boolean; tag?: boolean } | boolean`
- **Default:** `{ "version": false, "tag": false }`

In v3, private packages are neither versioned nor tagged by default. To version private packages (apps, fixtures, or non-npm deliverables that still have a `package.json`):

```json
{ "privatePackages": { "version": true, "tag": true } }
```

`version` updates their version on `changeset version`; `tag` creates git tags on `publish`. Private packages are still never published to npm. This is how Changesets manages Docker images, NuGet/gem packages, and apps via git tags — see [Beyond npm](https://changesets.dev/guide/beyond-npm).

## updateInternalDependencies

- **Type:** `"patch" | "minor"`
- **Default:** `"patch"`
- **Monorepos only.**

How internal dependency ranges are updated when a dependency is released:

- `"patch"`: update the range whenever the depended-on package is released (e.g. `^1.0.0` → `^1.0.1`).
- `"minor"`: update only when the dependency's bump is minor or major; patch releases leave the range alone.

Changesets always updates the range if the new version would fall outside it. The range is only updated when the package containing the dependency is itself being released.

Use `"minor"` to reduce duplicate installs across many interconnected packages; use `"patch"` to keep consumers on fresher code.

## ignore

- **Type:** `string[]`
- **Default:** `[]`
- **Monorepos only.** Supports picomatch patterns.

Temporarily excludes packages from publishing; their changesets are skipped until removed from `ignore`. Intended for short-lived situations, not permanent exclusion (use `"private": true` for that).

Two safety rails cause publishing to fail:

1. A changeset mentions both an ignored package and a non-ignored package.
2. An ignored package requires a dependency update as part of the publish.

These prevent broken repository or published states.

## bumpVersionsWithWorkspaceProtocolOnly

- **Type:** `boolean`
- **Default:** `false`
- **Monorepos only.**

When `true`, only dependency ranges using the `workspace:` protocol are bumped; other ranges are left alone. Useful when some dependencies intentionally pin published versions.

## snapshot

- **Type:** `{ useCalculatedVersion?: boolean; prereleaseTemplate?: string }`

### snapshot.useCalculatedVersion

- **Type:** `boolean`
- **Default:** `false`

By default snapshot versions use `0.0.0` as the base (`0.0.0-mytag-20261109000000`) so they never satisfy a normal semver range. Set `true` to base the snapshot on the calculated next version instead. Only do that if you understand the range-resolution consequences.

### snapshot.prereleaseTemplate

- **Type:** `string`
- **Default:** `"{tag}-{datetime}"` (or `"{datetime}"` when the tag is empty)

Placeholders: `{tag}`, `{commit}` (40-char SHA), `{commit-short}` (7 chars), `{timestamp}` (Unix ms), `{datetime}` (`YYYYMMDDHHMMSS`). `{tag}` errors when `--snapshot` is passed with an empty tag.

## Experimental options

`___experimentalUnsafeOptions_WILL_CHANGE_IN_PATCH` contains options whose shape may change in patch releases. Use with care.

- `updateInternalDependents`: `"out-of-range"` (default) or `"always"` — add dependent packages to the release with patch bumps even when already part of the release.
- `onlyUpdatePeerDependentsWhenOutOfRange`: `boolean` (default `false`) — only bump peer dependents when the peer range is actually left.

## Recipes

### Public scoped packages

```json
{
  "access": "public"
}
```

Or per package:

```json
{
  "name": "@scope/pkg",
  "publishConfig": { "access": "public" }
}
```

### GitHub changelogs

Install `@changesets/changelog-github` and configure:

```json
{
  "changelog": ["@changesets/changelog-github", { "repo": "owner/repo" }]
}
```

In CI, the generator needs a GitHub token (`GITHUB_TOKEN`) to fetch commit/PR information; without it, entries may omit links.

### No changelogs

```json
{ "changelog": false }
```

### Fixed product suite

```json
{ "fixed": [["@scope/core", "@scope/cli", "@scope/react"]] }
```

All members release together with one version.

### Linked packages with independent releases

```json
{ "linked": [["@scope/runtime", "@scope/plugin-api"]] }
```

They share version numbers when released, but do not force releases of untouched packages.

### Temporarily hold a package

```json
{ "ignore": ["@scope/experimental"] }
```

Remove the entry when it should release again. Expect publish failures if other changesets mix it with released packages.

### Version private apps and tag them

```json
{ "privatePackages": { "version": true, "tag": true } }
```

Pair with a workflow that reacts to tags instead of publishing to npm.

### Snapshot template with commit SHA

```json
{ "snapshot": { "prereleaseTemplate": "{tag}-{commit-short}" } }
```

Produces versions like `0.0.0-pr123-abc1234`, which makes the source commit obvious.

### v3 migration checklist for config

- `prettier: false` → `format: false` (or remove it to use `"auto"`).
- `access: "private"` → `"restricted"`.
- Private package versioning is now opt-in: add `"privatePackages": { "version": true }` if you relied on v2 behavior.
- `baseBranch` defaults to `"main"`; set `"master"` explicitly if needed.
- `___experimentalUnsafeOptions_WILL_CHANGE_IN_PATCH.useCalculatedVersionForSnapshots` → `snapshot.useCalculatedVersion`.
- If you used `ignore` only to hide test fixtures from versioning, remove those entries now that private packages are skipped by default.
