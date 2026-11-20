# release-plz GitHub Actions

The action is `release-plz/action`. It is a composite action that installs `cargo-semver-checks`, `cargo-binstall`, and the requested release-plz CLI version, configures the git user from the token, and runs `release-plz release-pr` and/or `release-plz release`.

Examples here use `release-plz/action@66b324fd6c72d05d2a08568492ecdd62bfb8a62b` (full commit for **v0.5.148**). That action defaults to release-plz **0.4.3**. Pin a different CLI with the `version` input.

## Contents

- [Quickstart workflow](#quickstart-workflow)
- [Action inputs](#action-inputs)
- [Action outputs](#action-outputs)
- [Permissions](#permissions)
- [Tokens and authentication](#tokens-and-authentication)
- [Trusted publishing](#trusted-publishing)
- [Concurrency](#concurrency)
- [persist-credentials and signed tags](#persist-credentials-and-signed-tags)
- [Security](#security)
- [Advanced recipes](#advanced-recipes)
- [Enterprise Server](#enterprise-server)
- [Common failures](#common-failures)

## Quickstart workflow

Two jobs, both on pushes to the main branch:

```yaml
name: Release-plz

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: write

jobs:
  release-plz-release:
    name: Release-plz release
    runs-on: ubuntu-latest
    if: github.repository_owner == 'your-org'
    permissions:
      contents: write        # push tags and create releases
      pull-requests: read    # inspect the release PR and its commits
    steps:
      - name: Checkout repository
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          fetch-depth: 0             # full history is required
          persist-credentials: false

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@39008827e6b4319a349e9a42dfc90579dd25f1ca # stable branch

      - name: Run release-plz
        uses: release-plz/action@66b324fd6c72d05d2a08568492ecdd62bfb8a62b # v0.5.148
        with:
          command: release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}

  release-plz-pr:
    name: Release-plz PR
    runs-on: ubuntu-latest
    if: github.repository_owner == 'your-org'
    permissions:
      contents: write        # push the release branch
      pull-requests: write   # create/update the release PR
    concurrency:
      group: release-plz-${{ github.ref }}
      cancel-in-progress: false
    steps:
      - name: Checkout repository
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          fetch-depth: 0
          persist-credentials: false

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@39008827e6b4319a349e9a42dfc90579dd25f1ca # stable branch

      - name: Run release-plz
        uses: release-plz/action@66b324fd6c72d05d2a08568492ecdd62bfb8a62b # v0.5.148
        with:
          command: release-pr
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

- `fetch-depth: 0` is required. A shallow checkout breaks commit analysis and release-PR detection.
- The `if: github.repository_owner == 'your-org'` guard prevents fork pushes from trying to release. Replace `your-org`.
- The examples pin `actions/checkout` v7.0.1; v6 works the same for these inputs.
- `dtolnay/rust-toolchain` is pinned to the `stable` branch commit; its default `toolchain` input remains `stable`, so Rust itself still tracks the latest stable release.
- The PR job's concurrency group is deliberately non-canceling: a second push waits for the first PR update instead of racing it.
- The release job has no canceling concurrency group. A canceled release can be skipped.

## Action inputs

| Input | Default | Meaning |
| --- | --- | --- |
| `command` | both | `release-pr`, `release`, or omit to run both in one action invocation. Two separate jobs are usually clearer. |
| `registry` | crates.io | Registry name present in the cargo config. |
| `config` | auto | Config file path; otherwise `release-plz.toml`, `.release-plz.toml`, `.config/release-plz.toml`. |
| `manifest_path` | root `Cargo.toml` | Workspace or package manifest. |
| `version` | `0.4.3` in action v0.5.148 | release-plz CLI version to install. |
| `token` | none | Registry token; overrides `CARGO_REGISTRY_TOKEN` / `CARGO_REGISTRIES_<NAME>_TOKEN`. |
| `forge` | `github` | `github`, `gitea`, or `gitlab`. `backend` is a backward-compatible alias. |
| `verbose` | false | Adds `-v` to the CLI. |
| `dry_run` | false | Adds `--dry-run` to the `release` command. If `command` is omitted (both commands run), the flag is added only to `release`; `release-pr` still runs for real. |

The action's `token` input is for the **cargo registry**, not GitHub. GitHub authentication comes from the `GITHUB_TOKEN` environment variable.

## Action outputs

When `command: release-pr`:

| Output | Meaning |
| --- | --- |
| `prs` | Array of release PRs (currently at most one). |
| `pr` | The release PR object: `head_branch`, `base_branch`, `html_url`, `number`, `releases`. |
| `prs_created` | `"true"` / `"false"`. |

When `command: release`:

| Output | Meaning |
| --- | --- |
| `releases` | Array of release objects: `package_name`, `prs`, `tag`, `version`. |
| `releases_created` | `"true"` / `"false"`. |

Example: read outputs and use them in a follow-up step:

```yaml
      - name: Run release-plz
        id: release-plz
        uses: release-plz/action@66b324fd6c72d05d2a08568492ecdd62bfb8a62b # v0.5.148
        with:
          command: release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}

      - name: Announce release
        if: steps.release-plz.outputs.releases_created == 'true'
        env:
          RELEASES: ${{ steps.release-plz.outputs.releases }}
        run: |
          echo "$RELEASES" | jq -r '.[] | "\(.package_name) \(.version) \(.tag)"'
```

`prs_created` / `releases_created` are strings, not booleans; compare with `'true'` in expressions.

## Permissions

| Job | Required permissions |
| --- | --- |
| `release-plz-pr` | `contents: write`, `pull-requests: write` |
| `release-plz-release` | `contents: write`, `pull-requests: read` |

- `contents: write` covers tags and forge releases.
- `pull-requests: read` is needed by `release` to detect release PRs and commit associations (especially with `release_always = false`).
- `pull-requests: write` is only needed by the PR job.
- If you use trusted publishing, add `id-token: write` to the release job.
- Repository setting **Settings → Actions → General → Workflow permissions → Allow GitHub Actions to create and approve pull requests** must be enabled for the PR job. This is a repository setting, not a workflow permission.

## Tokens and authentication

### Default `GITHUB_TOKEN`

The default `GITHUB_TOKEN` is enough to open the release PR and publish packages. It has two important limitations:

1. It **cannot trigger further workflow runs**. Checks on the release PR (triggered by `pull_request`) and workflows triggered by `release` / tag pushes will not run.
2. The account is `github-actions[bot]`; the release PR author is the bot.

If you only need release-plz itself, use `GITHUB_TOKEN`.

### PAT or GitHub App

Use a PAT or GitHub App token when you need release-plz's PR/tag to trigger other workflows (CI checks, binary builds, announcements).

**Fine-grained PAT**:

- Repository access: select the repositories.
- Contents: read/write.
- Pull requests: read/write.
- Store as `RELEASE_PLZ_TOKEN` and pass it to both `actions/checkout` (if git pushes) and the release-plz step:

```yaml
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          fetch-depth: 0
          persist-credentials: false

      - uses: release-plz/action@66b324fd6c72d05d2a08568492ecdd62bfb8a62b # v0.5.148
        with:
          command: release-pr
        env:
          GITHUB_TOKEN: ${{ secrets.RELEASE_PLZ_TOKEN }}
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

**GitHub App** (used by the release-plz project itself):

```yaml
      - uses: actions/create-github-app-token@f418cff399f7aef4c2f0ad47e25ca7a6abbdd6fc # v3.4.0
        id: app-token
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}

      - uses: release-plz/action@66b324fd6c72d05d2a08568492ecdd62bfb8a62b # v0.5.148
        with:
          command: release-pr
        env:
          GITHUB_TOKEN: ${{ steps.app-token.outputs.token }}
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

App permissions: Contents read/write, Pull requests read/write; add Administration read/write only if you use protected tags. Install the App only on the repositories that need it.

The pinned SHA is `actions/create-github-app-token` v3.4.0; it runs on Node 24 and keeps the same `app-id` and `private-key` inputs. v2 declares `node20` and is not supported on GitHub-hosted runners after 2026-09-23.

### PAT account identity

The PAT owner becomes the author of the release PR and the release commits. Use a dedicated machine user if you do not want a personal account in the history.

## Trusted publishing

crates.io trusted publishing lets the release job obtain a short-lived token via OIDC. The release-plz action implements the same crates.io API calls as `rust-lang/crates-io-auth-action`.

1. Configure trusted publishing on crates.io for every crate.
2. Do **not** set `CARGO_REGISTRY_TOKEN` and do **not** add `rust-lang/crates-io-auth-action`.
3. Add `id-token: write` to the release job:

```yaml
    permissions:
      contents: write
      pull-requests: read
      id-token: write
```

```yaml
      - uses: release-plz/action@66b324fd6c72d05d2a08568492ecdd62bfb8a62b # v0.5.148
        with:
          command: release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          # no CARGO_REGISTRY_TOKEN
```

New crates cannot be published with trusted publishing on crates.io. Publish each new crate manually once, then let trusted publishing take over. This is a crates.io limitation.

## Concurrency

Use `concurrency` only on the PR job:

```yaml
    concurrency:
      group: release-plz-${{ github.ref }}
      cancel-in-progress: false
```

- `cancel-in-progress: false` makes a new run wait for the previous PR update; this prevents force-push conflicts.
- Do **not** add a canceling concurrency group to the release job. If a release is canceled, that version may never be published until the next commit.

## persist-credentials and signed tags

`actions/checkout` defaults to `persist-credentials: true`. release-plz normally uses the forge API to create commits, branches, and tags, so `persist-credentials: false` is the safer default.

Set `persist-credentials: true` when:

- **Tags are signed** (`git config --get tag.gpgSign` is true). The API cannot sign tags, so release-plz falls back to the git CLI to push them.
- A later step uses `git push` (for example, committing files into the release PR).

When persisting credentials, also pass the token to checkout if you use a PAT:

```yaml
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          fetch-depth: 0
          persist-credentials: true
          token: ${{ secrets.RELEASE_PLZ_TOKEN }}
```

## Security

- **Pin the action to a commit SHA** instead of a floating major tag. A compromised action could abuse `CARGO_REGISTRY_TOKEN` to publish malicious crates. The release-plz docs explicitly recommend this; crates.io does the same.
- **Pin the CLI version** with the `version` input when you need reproducible behavior. The action's default changes when the action is updated.
- **Keep tokens in `env:`**, not in the `run:` command line. Shell history and process listings can expose command-line secrets.
- **Use least privilege**: `contents: write` + `pull-requests: read` for release, adding `pull-requests: write` only for the PR job.
- **Guard fork pushes** with `if: github.repository_owner == 'your-org'`; never run release jobs with secrets on untrusted forks.
- **Do not use `pull_request_target`** to run release-plz against fork code with a write token.

## Advanced recipes

### Run checks before releasing

Prefer running tests/lints on PRs and merging only green PRs. If a release job must run checks, put them before the release-plz step:

```yaml
      - run: cargo test --workspace
      - uses: release-plz/action@66b324fd6c72d05d2a08568492ecdd62bfb8a62b # v0.5.148; command: release
```

This delays the release job and serializes checks with publishing; the PR-based approach runs them in parallel.

### Add labels to released PRs

Use the `releases` output and `gh` to label the code PRs included in a release:

```yaml
      - name: Label released PRs
        if: steps.release-plz.outputs.releases_created == 'true'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          RELEASES: ${{ steps.release-plz.outputs.releases }}
        run: |
          echo "$RELEASES" | jq -r -c '.[]' | while IFS= read -r release; do
            pkg=$(echo "$release" | jq -r '.package_name')
            ver=$(echo "$release" | jq -r '.version')
            echo "$release" | jq -r -c '.prs[]?' | while IFS= read -r pr; do
              num=$(echo "$pr" | jq -r '.number')
              gh pr edit "$num" --add-label "released: $pkg-$ver"
            done
          done
```

Create labels first (`gh label create ... --force`) if they do not exist. For release PR labels, use `pr_labels` in `release-plz.toml` instead.

### Auto-merge the release PR

Only if branch protection and required checks allow it. After `release-pr`, read `prs_created` / `pr` and enable auto-merge:

```yaml
      - name: Enable auto-merge
        if: steps.release-plz.outputs.prs_created == 'true'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR: ${{ steps.release-plz.outputs.pr }}
        run: gh pr merge --auto --merge "$(echo "$PR" | jq -r '.number')"
```

Use `--merge` (not squash) when `release_always = false`; the release PR commit association depends on the merge strategy.

### Commit files to the release PR

```yaml
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          fetch-depth: 0
          persist-credentials: true
      # ... run release-plz release-pr with id: release-plz ...
      - name: Update generated files in the release PR
        if: steps.release-plz.outputs.prs_created == 'true'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR: ${{ steps.release-plz.outputs.pr }}
        run: |
          gh pr checkout "$(echo "$PR" | jq -r '.number')"
          # edit files
          git add -A
          git commit -m "chore: update generated files"
          git push
```

`persist-credentials: true` is required for the `git push`.

### Close the old release PR first

If you prefer a new PR each time instead of force-pushing the existing one:

```yaml
      - name: Close old release PR
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          pr=$(gh pr list --state=open --json number,headRefName \
            --jq '.[] | select(.headRefName | startswith("release-plz-")) | .number')
          if [ -n "$pr" ]; then gh pr close "$pr"; fi
```

Then run `release-pr`.

### Run on a schedule

```yaml
on:
  schedule:
    - cron: '0 0 * * MON'
```

The rest of the workflow is unchanged. Scheduled runs are useful for dependency-only changes that do not produce commits.

### Submodules

```yaml
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          fetch-depth: 0
          persist-credentials: false
          submodules: recursive
```

Use `submodules: true` for one level, `recursive` for nested submodules. If a submodule is hosted on another forge, the checkout step needs credentials for it; see the GitHub docs on submodule authentication.

### Workspace or nested manifest

```yaml
        with:
          manifest_path: rust-crates/my-crate/Cargo.toml
```

`manifest_path` scopes the action to a workspace member. The config file lookup still starts from the working directory; pass `config` explicitly if it lives elsewhere.

## Enterprise Server

GitHub Enterprise Server is supported. release-plz derives the API host from the git remote. Set `forge: github` explicitly (the action defaults to GitHub, but explicit is safer), and use the Enterprise `GITHUB_TOKEN`/PAT. If you pin actions by SHA, keep the action's internal `actions/*` references reachable in your Enterprise environment.

## Common failures

| Symptom | Fix |
| --- | --- |
| `release-pr` says no changes, but there are commits | Check `fetch-depth: 0`, the default branch name, and whether the changed files are packaged by cargo (`cargo package --list`); commits outside the package directory may not count. |
| Release PR created but no CI runs on it | Default `GITHUB_TOKEN` cannot trigger workflows. Use a PAT/GitHub App token. |
| `release` does nothing after merge | `release_always = false` did not find the release PR association (squash merge, merge queue, branch prefix); or the version is already published. |
| `release` fails with `git release not configured` | The CLI/action needs a forge token. In the action, set `GITHUB_TOKEN`; in CLI use `--git-token`/`--forge`. |
| `cargo publish` fails with auth | `CARGO_REGISTRY_TOKEN` missing/invalid; for private registries use `CARGO_REGISTRIES_<NAME>_TOKEN`. |
| Trusted publishing fails | `id-token: write` missing, crates.io trusted publishing not configured, or the crate has never been published (new crates need a manual first publish). |
| Action starts behaving differently | Pin the full action SHA and the `version` input; the floating `@v0.5` changes with every release. |
| Release job canceled mid-publish | Remove `concurrency` from the release job; do not cancel publishing. |
