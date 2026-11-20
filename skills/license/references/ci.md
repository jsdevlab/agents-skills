# GitHub Actions integration

SkyWalking Eyes publishes two composite actions in the same repository:

- `apache/skywalking-eyes/header` — runs `license-eye header <mode>`.
- `apache/skywalking-eyes/dependency` — runs `license-eye dependency <mode>`.

Both actions build the `license-eye` binary from source on the runner using Go, then execute it. They are not precompiled JavaScript actions. Pin a full commit SHA; `@main` can change behavior without notice.

Older workflows may use `apache/skywalking-eyes@<ref>` without a subpath. Since 0.4.0 that form is kept as an alias of `apache/skywalking-eyes/header@<ref>` for backward compatibility. New workflows should use `/header` or `/dependency` explicitly.

## Contents

- [Action inputs](#action-inputs)
- [Header check workflow](#header-check-workflow)
- [Header fix workflow](#header-fix-workflow)
- [Dependency check workflow](#dependency-check-workflow)
- [Dependency resolve and LICENSE drift](#dependency-resolve-and-license-drift)
- [Permissions and PR comments](#permissions-and-pr-comments)
- [Using the CLI instead of the actions](#using-the-cli-instead-of-the-actions)
- [Monorepos and multiple configs](#monorepos-and-multiple-configs)
- [Self-hosted runners](#self-hosted-runners)

## Action inputs

### `apache/skywalking-eyes/header`

| Input | Default | Meaning |
| --- | --- | --- |
| `log` | `info` | Log level passed as `-v`. |
| `config` | `.licenserc.yaml` | Config file path. |
| `token` | `${{ github.token }}` | Token used for PR comments. Set to `""` to disable commenting. |
| `mode` | `check` | `check`, `fix`, or `diff`. |

The action runs:

```bash
license-eye -v "$INPUTS_LOG" -c "$INPUTS_CONFIG" header "$INPUTS_MODE"
```

### `apache/skywalking-eyes/dependency`

| Input | Default | Meaning |
| --- | --- | --- |
| `log` | `info` | Log level passed as `-v`. |
| `config` | `.licenserc.yaml` | Config file path. |
| `mode` | `check` | `check` or `resolve`. |
| `flags` | empty | Extra arguments appended to the command, for example `--summary=path/to/LICENSE.tpl`. |

The action runs:

```bash
license-eye -v "${{ inputs.log }}" -c "${{ inputs.config }}" dependency "${{ inputs.mode }}" ${{ inputs.flags }}
```

Note the `flags` input is interpolated directly into the shell command. Keep it to simple flags and paths; do not pass untrusted values.

## Header check workflow

```yaml
name: License header

on:
  push:
    branches: [main]
  pull_request:
    branches: ['*']
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: write   # only needed if comment != never

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  license-header:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - name: Check license headers
        uses: apache/skywalking-eyes/header@a196742f472feaffafea537ce5a2a4c3c53a8de4 # v0.9.0
        with:
          config: .licenserc.yaml
          mode: check
          # token: ""        # uncomment to disable PR comments
```

- This job fails on invalid headers (`check` exits 1).
- On a `pull_request` event with `pull-requests: write`, it can add review comments and a summary when the config's `comment` is `on-failure` or `always`.
- On `push` events it only checks; comments require a PR context.

## Header fix workflow

`mode: fix` rewrites files but exits 0 when it succeeds, so it cannot gate CI. Use it as a bot that commits fixes, and keep a separate `check` job as the gate.

```yaml
name: Fix license headers

on:
  pull_request:
    branches: ['*']

permissions:
  contents: write

jobs:
  fix-headers:
    if: github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          ref: ${{ github.head_ref }}
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Apply license headers
        uses: apache/skywalking-eyes/header@a196742f472feaffafea537ce5a2a4c3c53a8de4 # v0.9.0
        with:
          config: .licenserc.yaml
          mode: fix
          token: ""

      - name: Commit fixes
        uses: EndBug/add-and-commit@cc9c08ba6c8df3b93a8f2db63e89b98368ae2ae8 # v11.1.1
        with:
          author_name: License Bot
          author_email: license-bot@users.noreply.github.com
          message: 'chore: apply license headers'
          default_author: github_actions
```

`EndBug/add-and-commit@v11` runs on Node 24; v9 still declares `node20` and is
not supported on GitHub-hosted runners after 2026-09-23.

Caveats:

- The `if` guard avoids pushing to fork branches. Do not use `pull_request_target` to get around it; that runs with a privileged token against untrusted code.
- Commits made with the default `GITHUB_TOKEN` do not trigger further workflows. If downstream checks must run, use a GitHub App token or a PAT with the minimum scope.
- `fix` still exits non-zero for files it cannot fix (for example JSON or unknown extensions). The step should fail in that case; fix the config or exclude the file.
- Review the bot's commit like any other change; `fix` can rewrite a header unexpectedly when `pattern` is too broad.

An alternative is to run `fix` in a scheduled or manual workflow and open a PR with `peter-evans/create-pull-request`, keeping human review in the loop.

## Dependency check workflow

The dependency action needs the ecosystem toolchain. Set it up before invoking the action:

```yaml
name: Dependency licenses

on:
  push:
    branches: [main]
  pull_request:
    branches: ['*']
  workflow_dispatch:

permissions:
  contents: read

jobs:
  dependency-license:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1

      # Pick the setup steps for the manifests in .licenserc.yaml.
      - uses: actions/setup-go@d8cbb8045d7f43706b9177cceb36c1f91a6644d5 # v7.1.0
        if: hashFiles('go.mod') != ''
        with:
          go-version-file: go.mod

      - uses: actions/setup-java@d1374f8ac74474200d16ef8ecee22c7ae7a51880 # v6.3.0
        if: hashFiles('pom.xml') != ''
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - uses: actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1 # v7.1.0
        if: hashFiles('package.json') != ''
        with:
          node-version: '24'
          cache: npm   # or pnpm / yarn

      - uses: dtolnay/rust-toolchain@39008827e6b4319a349e9a42dfc90579dd25f1ca # stable branch
        if: hashFiles('Cargo.toml') != ''

      - uses: ruby/setup-ruby@be3c7df29b3349ec6bcfc6b4792f1cf9f5237d18 # v1.343.0
        if: hashFiles('Gemfile.lock') != ''
        with:
          bundler-cache: true

      - name: Check dependency licenses
        uses: apache/skywalking-eyes/dependency@a196742f472feaffafea537ce5a2a4c3c53a8de4 # v0.9.0
        with:
          config: .licenserc.yaml
          mode: check
```

Notes:

- `if: hashFiles(...)` is evaluated per step; remove the conditions if the repository has a fixed stack and you prefer explicit setup.
- For Node projects, ensure production dependencies are installed before the action if you want to avoid its interactive install prompt. `npm ci` in a prior step also restores a predictable tree.
- For Maven, the action may run `dependency:resolve` and fall back to `clean install -DskipTests`. Cache `~/.m2` via `setup-java` to keep it fast.
- For Ruby, `bundler-cache: true` installs gems and caches them; the resolver still queries RubyGems for missing license metadata.
- The dependency action does not use a token; it only checks.

## Dependency resolve and LICENSE drift

To keep a committed LICENSE summary in sync:

```yaml
name: LICENSE drift

on:
  pull_request:
    branches: ['*']

permissions:
  contents: read

jobs:
  license-drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1

      # Set up the ecosystem(s) used by dependency.files here.
      - uses: actions/setup-go@d8cbb8045d7f43706b9177cceb36c1f91a6644d5 # v7.1.0
        with:
          go-version-file: go.mod

      - name: Regenerate LICENSE
        uses: apache/skywalking-eyes/dependency@a196742f472feaffafea537ce5a2a4c3c53a8de4 # v0.9.0
        with:
          config: .licenserc.yaml
          mode: resolve
          flags: --summary=dist/LICENSE.tpl

      - name: Check for drift
        run: |
          if [ -n "$(git diff -U0 -- dist/LICENSE)" ]; then
            echo "::error::LICENSE is not updated. Regenerate it with:"
            echo "license-eye dependency resolve --summary dist/LICENSE.tpl"
            git diff -U0 -- dist/LICENSE
            exit 1
          fi
```

- `--summary=<template>` writes `LICENSE` in the template's directory. That output must be tracked by Git for `git diff` to detect drift; if `dist/` is gitignored, pass `--license=<path>` in `flags` and point the diff at a committed path instead.
- Run this on the same OS as the job that produces the committed LICENSE (Linux in most projects). Platform-specific dependencies can make a locally generated file differ.
- If `resolve` exits non-zero because a license is unidentified, the step fails before the diff. Add a verified `dependency.licenses` override or an `excludes` entry.

## Permissions and PR comments

Least privilege for a check-only workflow:

```yaml
permissions:
  contents: read
```

For header comments, add only the permission the comment path needs:

```yaml
permissions:
  contents: read
  pull-requests: write
```

- Comments require a `pull_request` event. `pull_request_target` also works but is dangerous if the workflow checks out and processes fork code; avoid it for license checks.
- On pull requests from forks, the default `GITHUB_TOKEN` is read-only. PR comments from the header action may fail; either accept that, or run a separate trusted workflow on `pull_request_target` that does not execute fork code.
- Set `comment: never` in `.licenserc.yaml` or `token: ""` on the header action to eliminate the need for `pull-requests: write`.
- Dependency checks never comment; they only need `contents: read`.

## Using the CLI instead of the actions

If the repository restricts third-party actions, install a pinned binary and run the CLI. With Go available:

```yaml
      - name: Install license-eye
        run: |
          go install github.com/apache/skywalking-eyes/cmd/license-eye@v0.9.0
          echo "$(go env GOPATH)/bin" >> "$GITHUB_PATH"

      - name: Check headers
        run: license-eye -c .licenserc.yaml header check

      - name: Check dependency licenses
        run: license-eye -c .licenserc.yaml dependency check
```

Or use the container:

```yaml
      - name: Check headers in container
        run: |
          docker run --rm \
            -v "$PWD:/github/workspace" \
            -w /github/workspace \
            apache/skywalking-eyes:0.9.0 header check
```

The container image contains only the binary; dependency checks need a toolchain installed in the container or on the host. For supply-chain-sensitive repositories, pin the action to a commit SHA and verify the release checksum when downloading binaries.

## Monorepos and multiple configs

Run one job per config, or one step per module:

```yaml
      - name: Check module A
        uses: apache/skywalking-eyes/header@a196742f472feaffafea537ce5a2a4c3c53a8de4 # v0.9.0
        with:
          config: modules/a/.licenserc.yaml
          mode: check

      - name: Check module B
        uses: apache/skywalking-eyes/header@a196742f472feaffafea537ce5a2a4c3c53a8de4 # v0.9.0
        with:
          config: modules/b/.licenserc.yaml
          mode: check
```

Remember that `header.paths` is relative to the working directory (the repository root in Actions), while `dependency.files` is relative to the config file. A module config should use paths like `modules/a/**`, not `**`, unless the intent is to check the whole repository.

## Self-hosted runners

The actions run `make install` and copy the binary to `/usr/local/bin`. On a self-hosted runner:

- Ensure the runner user can write to `/usr/local/bin`, or set `INSTALL_DIR` in the environment if you maintain a fork of the action.
- Ensure Go 1.27 (or the version pinned by the action) is available and can download modules.
- Ensure the ecosystem toolchains and network access required by dependency checks are present.
- If the runner has no network access, pre-populate the Go module cache and the language package caches, or use a pinned binary installed by your own provisioning.
