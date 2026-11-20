# CI automation

There are two independent questions to answer:

1. **How do PRs get changesets?** Either non-blocking (a bot/comment) or blocking (`changeset status --since main`).
2. **How do version and publish run?** Normally with `changesets/action` v2 (SHA-pinned in the examples), which opens a "Version Packages" PR and publishes when that PR merges.

## Prerequisites

- Repository setting **Settings → Actions → General → Allow GitHub Actions to create and approve pull requests** must be enabled. Without it, the version job fails with `GitHub Actions is not permitted to create or approve pull requests` or a `github-actions[bot]` permission error.
- `changesets/action@v2` is required for Changesets v3. The v1 action only supports v2.
- For trusted publishing, configure the package on npm first (see below).

## Ensure PRs have changesets

### Non-blocking: PR status comment

The `changesets/action/pr-status` and `pr-comment` sub-actions check for changesets and comment on the PR without failing CI. This is the recommended default because not every change needs a release.

```yaml
name: Comment Changesets PR Status

on:
  pull_request_target:

permissions: {} # reset permissions; grant per job

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}

jobs:
  pr-status:
    # skip the release PR, which never has changesets
    if: ${{ !startsWith(github.head_ref, 'changeset-release/') }}
    runs-on: ubuntu-slim
    permissions:
      contents: read
    outputs:
      comment-body: ${{ steps.pr-status.outputs.comment-body }}
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - id: pr-status
        uses: changesets/action/pr-status@ae32849d5ba541f9ae29e40e22a623bc13562f51 # v2.1.2

  pr-comment:
    needs: pr-status
    runs-on: ubuntu-slim
    permissions:
      pull-requests: write
    steps:
      - uses: changesets/action/pr-comment@ae32849d5ba541f9ae29e40e22a623bc13562f51 # v2.1.2
        with:
          body: ${{ needs.pr-status.outputs.comment-body }}
```

Security note: `pull_request_target` runs with write permissions. The workflow above only checks out and reads code; it does not execute code from the fork. Do not add build/test steps to a `pull_request_target` job that checks out the PR head.

### Blocking: status check

If the repository requires a changeset for every changed package:

```bash
changeset status --since main
```

In CI this exits 1 when packages changed but no new changesets exist since `main`, and 0 when nothing changed. Allow an escape hatch for changes that release nothing:

```bash
changeset add --empty -m "Update CI configuration"
```

The Changesets GitHub Bot is an alternative non-blocking approach.

## Version + publish with trusted publishing

Trusted publishing (OIDC) is the recommended way to publish from GitHub Actions. It avoids long-lived npm tokens. Staged publishing is not currently supported by Changesets; use trusted publishing.

Recommended job structure:

- `select-mode` decides whether the run should version or publish.
- `version` creates/updates the Version Packages PR when changesets exist.
- `pack` builds and packs tarballs when publishing.
- `publish` publishes the packed artifacts with `id-token: write`.

```yaml
name: Publish

on:
  push:
    branches:
      - main

permissions: {}
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}

jobs:
  select-mode:
    runs-on: ubuntu-latest
    outputs:
      mode: ${{ steps.select-mode.outputs.mode }}
      publish-plan-artifact-id: ${{ steps.select-mode.outputs.publish-plan-artifact-id }}
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - uses: actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1 # v7.1.0
        with:
          node-version: 24
      - run: npm install
      - id: select-mode
        uses: changesets/action/select-mode@ae32849d5ba541f9ae29e40e22a623bc13562f51 # v2.1.2

  version:
    if: needs.select-mode.outputs.mode == 'version'
    needs: select-mode
    runs-on: ubuntu-latest
    outputs:
      version-dir-artifact-id: ${{ steps.version.outputs.version-dir-artifact-id }}
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - uses: actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1 # v7.1.0
        with:
          node-version: 24
          package-manager-cache: false
      - run: npm install
      - id: version
        uses: changesets/action/version@ae32849d5ba541f9ae29e40e22a623bc13562f51 # v2.1.2

  pack:
    if: needs.select-mode.outputs.mode == 'publish'
    needs: select-mode
    runs-on: ubuntu-latest
    outputs:
      pack-dir-artifact-id: ${{ steps.pack.outputs.pack-dir-artifact-id }}
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - uses: actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1 # v7.1.0
        with:
          node-version: 24
          package-manager-cache: false
      - run: npm install
      - run: npm run build
      - id: pack
        uses: changesets/action/pack@ae32849d5ba541f9ae29e40e22a623bc13562f51 # v2.1.2
        with:
          publish-plan-artifact-id: ${{ needs.select-mode.outputs.publish-plan-artifact-id }}

  publish:
    needs: pack
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write # only this job needs OIDC for trusted publishing
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - uses: actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1 # v7.1.0
        with:
          node-version: 24
          package-manager-cache: false
      - run: npm install
      - uses: changesets/action/publish@ae32849d5ba541f9ae29e40e22a623bc13562f51 # v2.1.2
        with:
          pack-dir-artifact-id: ${{ needs.pack.outputs.pack-dir-artifact-id }}
```

Notes:

- **pnpm projects:** replace the separate Node/pnpm setup and install steps with `pnpm/setup` v3 (SHA below):

  ```yaml
  - uses: pnpm/setup@731dfb6463bbd11c11fd0ee11fe00ed0e36d48a1 # v3.2.0
    with:
      version: 12
      runtime: node@24
      cache: true
      require-lockfile: true
  ```

  It installs pnpm 11+ and the requested Node runtime, then runs `pnpm install` by default. Drop the separate `actions/setup-node` and `pnpm install` steps; set `install: false` only if a later step handles installation. Keep the Changesets action steps unchanged.
- The pinned SHAs for `actions/checkout` v7.0.1 and `pnpm/setup` v3.2.0 run on the runner's Node 24 runtime. Do not use older action majors that still declare `node20`; Node 20 was removed from GitHub-hosted runners on 2026-09-23.
- `id-token: write` belongs only on the publish job. Splitting build/test/publish is a security improvement recommended by npm.
- Optionally add a GitHub Environment with required reviewers to the `publish` job so a maintainer must approve each publish.
- The examples pin the `actions/checkout` v7.0.1 and `actions/setup-node` v7.1.0 SHAs; the official docs may show v7 tags. Keep action SHAs current in your repository.

### First publish and trusted publishing

npm requires a package to exist before trusted publishing can be configured. For a brand-new package:

1. Create a stub `package.json` (`"name": "pkg-a", "version": "0.0.0"`) and run `npm publish` locally.
2. Configure the trusted publisher in the package's npm settings.
3. Add a changeset that bumps to the intended initial version (e.g. `"pkg-a": minor` for `0.1.0`) and release through CI.

For public scoped packages, ensure `access: "public"` or `publishConfig.access` is set before the first publish.

## Token-based publishing

Token-based publishing with granular access tokens is no longer recommended: tokens expire within 90 days and npm is deprecating 2FA-bypass tokens for publishing. Use it only when a registry does not support trusted publishing.

```yaml
# ... same select-mode/version jobs as above ...
  publish:
    if: needs.select-mode.outputs.mode == 'publish'
    needs: select-mode
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - uses: actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1 # v7.1.0
        with:
          node-version: 24
          registry-url: https://registry.npmjs.org/
          package-manager-cache: false
      - run: npm install
      - run: npm run build
      - uses: changesets/action/publish@ae32849d5ba541f9ae29e40e22a623bc13562f51 # v2.1.2
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

Store the token as the `NPM_TOKEN` repository secret. It needs permission to publish the relevant scopes and, if 2FA is enabled, "Bypass two-factor authentication". Never print it or pass it to jobs that do not publish.

## Version only (no npm publishing)

Useful for internal packages or changelog-only repositories:

```yaml
name: Version

on:
  push:
    branches: [main]

permissions: {}
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}

jobs:
  version:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - uses: actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1 # v7.1.0
        with:
          node-version: 24
          package-manager-cache: false
      - run: npm install
      - run: npm run build
      - uses: changesets/action@ae32849d5ba541f9ae29e40e22a623bc13562f51 # v2.1.2
```

The unified `changesets/action@v2` opens the version PR and, by default, also publishes when there are no changesets. For version-only behavior, use the `changesets/action/version@v2` sub-action instead of the unified action, or configure the action to skip publishing. Check the action's current inputs for your version.

## Publish git tags only

For private packages or when a separate tag-triggered workflow publishes artifacts:

1. Set `"private": true` in each package's `package.json`.
2. Set `"privatePackages": { "version": true, "tag": true }` in `.changeset/config.json`.
3. Run the publish job without npm authentication; the SHA-pinned `changesets/action/publish` v2 creates tags, and downstream workflows react to them.

Use `changeset git-tag` manually if a different tool publishes packages.

## npm authentication details

`actions/setup-node` with `registry-url` writes an `.npmrc` like:

```ini
//registry.npmjs.org/:_authToken=${NODE_AUTH_TOKEN}
```

The environment-variable indirection is safer than storing the token in `.npmrc`. If you need custom registries or scopes, write `~/.npmrc` yourself and omit `registry-url`:

```ini
//registry.npmjs.org/:_authToken=${NODE_AUTH_TOKEN}
@foo:registry=https://my-registry.example.com/
//my-registry.example.com/:_authToken=${NODE_FOO_AUTH_TOKEN}
@bar:registry=https://npm.pkg.github.com/
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

- **pnpm:** an `.npmrc` in the project directory does not expand environment variables (security restriction). Put auth in the home directory.
- **yarn:** does not use `.npmrc`; use `~/.yarnrc.yml` (`npmAuthToken`, `npmScopes`).

## Version PRs and workflow triggering

PRs created with the default `GITHUB_TOKEN` do not trigger other workflows. If the version PR must run CI (tests, checks, or a publish workflow), create a GitHub App token or use a PAT and pass it to the action:

```yaml
- uses: actions/create-github-app-token@f418cff399f7aef4c2f0ad47e25ca7a6abbdd6fc # v3.4.0
  id: app-token
  with:
    client-id: ${{ vars.APP_CLIENT_ID }}
    private-key: ${{ secrets.APP_PRIVATE_KEY }}
    permission-contents: write
    permission-pull-requests: write

- uses: changesets/action/version@ae32849d5ba541f9ae29e40e22a623bc13562f51 # v2.1.2
  with:
    github-token: ${{ steps.app-token.outputs.token }}
```

Also ensure neither the version commit nor the merge commit contains `[skip ci]` (or variants), including via the `commit` config's `skipCI` option. Otherwise tests or publish steps silently skip.

## changesets/action v2 input names

If migrating from v1, rename inputs:

| v1 | v2 |
| --- | --- |
| `version` | `version-script` |
| `publish` | `publish-script` |
| `commit` | `commit-message` |
| `title` | `pr-title` |

Pass a custom token via `github-token`. If later steps depend on the local git state, set `push-with-git-cli: true`.
