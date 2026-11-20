# Dependabot recipes

Adapt these to the repository's actual manifests. Remove ecosystems that are not present.

## Contents

- [npm/pnpm monorepo](#npmpnpm-monorepo)
- [Rust workspace](#rust-workspace)
- [Python with uv](#python-with-uv)
- [Go modules](#go-modules)
- [GitHub Actions, Docker, and Terraform](#github-actions-docker-and-terraform)
- [Security updates only](#security-updates-only)
- [Private registry](#private-registry)
- [Low-noise maintenance](#low-noise-maintenance)

## npm/pnpm monorepo

```yaml
version: 2

updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Europe/Berlin"
    groups:
      production:
        dependency-type: "production"
        update-types: ["minor", "patch"]
      development:
        dependency-type: "development"
        update-types: ["minor", "patch"]
      linting:
        patterns: ["eslint*", "prettier*", "@typescript-eslint*"]
    labels:
      - "dependencies"
      - "javascript"
    commit-message:
      prefix: "deps"
      include: "scope"
    open-pull-requests-limit: 10
```

- pnpm and yarn use the `npm` ecosystem.
- With a pnpm workspace, the root `pnpm-lock.yaml` covers `apps/*` and `packages/*`; start with `directory: "/"`.
- Use `directories` only when packages have independent lockfiles or dependency graphs outside the workspace root. Avoid overlapping entries.
- `groups.<name>.group-by: dependency-name` can combine the same dependency across directories when you do use multiple directories.
- Major updates remain ungrouped so they get individual review.

## Rust workspace

```yaml
version: 2

updates:
  - package-ecosystem: "cargo"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      patch-updates:
        update-types: ["patch"]
      dev-dependencies:
        dependency-type: "development"
        update-types: ["minor", "patch"]
    cooldown:
      semver-major-days: 14
      semver-minor-days: 5
      semver-patch-days: 3
    labels:
      - "dependencies"
      - "rust"
    commit-message:
      prefix: "deps"
      include: "scope"
```

- `dependency-type` group support for Cargo may be limited; verify against the current options reference.
- A workspace with `Cargo.lock` at the root needs only `/`; add crate directories only if they have independent manifests/lockfiles.
- If the project uses release-plz, dependency PRs are regular PRs; release-plz handles the crate release separately.

## Python with uv

```yaml
version: 2

updates:
  - package-ecosystem: "uv"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      runtime:
        dependency-type: "production"
        update-types: ["minor", "patch"]
      dev:
        dependency-type: "development"
        update-types: ["minor", "patch"]
    labels: ["dependencies", "python"]
```

Use `pip` when the project uses `requirements.txt`, Poetry, or Pipenv without `uv.lock`. Use `uv` when `uv.lock` is present.

## Go modules

```yaml
version: 2

updates:
  - package-ecosystem: "gomod"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      golang-x:
        patterns: ["golang.org/x/*"]
      minor-and-patch:
        update-types: ["minor", "patch"]
    labels: ["dependencies", "go"]
```

For a multi-module repository, add each `go.mod` directory explicitly or use `directories` with `*` paths supported by the current Dependabot docs.

## GitHub Actions, Docker, and Terraform

```yaml
version: 2

multi-ecosystem-groups:
  infrastructure:
    schedule:
      interval: "weekly"
    labels: ["infrastructure", "dependencies"]
    assignees: ["platform-team"]

updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      actions:
        patterns: ["*"]

  - package-ecosystem: "docker"
    directory: "/"
    patterns: ["nginx", "redis", "postgres"]
    multi-ecosystem-group: "infrastructure"

  - package-ecosystem: "terraform"
    directory: "/infra"
    patterns: ["aws*", "terraform-*"]
    multi-ecosystem-group: "infrastructure"
```

- `patterns` is required for every update assigned to a multi-ecosystem group.
- `github-actions` is deliberately not in the multi-ecosystem group in this example; actions often need separate review/CI behavior. Put it in the group only if you want one infra PR for everything.
- Use `["*"]` if all dependencies in an ecosystem should be grouped.

## Security updates only

```yaml
version: 2

updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "daily"
    open-pull-requests-limit: 0
    groups:
      security:
        applies-to: "security-updates"
        patterns: ["*"]
        update-types: ["patch", "minor"]
```

- `open-pull-requests-limit: 0` disables version-update PRs for npm; security updates still open (separate fixed limit of 10).
- Enable Dependabot alerts and security updates in repository settings; YAML alone does not enable them.
- Grouping security updates is optional; some teams prefer one PR per advisory for auditability.

## Private registry

```yaml
version: 2

registries:
  npm-private:
    type: npm-registry
    url: https://npm.example.com
    token: ${{secrets.NPM_TOKEN}}

  docker-private:
    type: docker-registry
    url: https://registry.example.com
    username: ${{secrets.REGISTRY_USER}}
    password: ${{secrets.REGISTRY_PASSWORD}}

updates:
  - package-ecosystem: "npm"
    directory: "/"
    registries: ["npm-private"]
    schedule:
      interval: "weekly"

  - package-ecosystem: "docker"
    directory: "/"
    registries: ["docker-private"]
    schedule:
      interval: "weekly"
```

- Store tokens/passwords as Actions secrets, never in the YAML.
- The registry URL/type must match the package manager.
- If the update process needs manifest code execution, review `insecure-external-code-execution` carefully; it can expose registry credentials to a compromised package.

## Low-noise maintenance

```yaml
version: 2

updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "monthly"
    cooldown:
      default-days: 7
      semver-major-days: 30
    groups:
      all-minor-patch:
        update-types: ["minor", "patch"]
    open-pull-requests-limit: 5
    labels: ["dependencies"]
```

- Monthly scheduling plus grouping is the biggest noise reduction.
- Cooldowns delay fresh releases; major-version cooldowns reduce early breakage.
- Keep major updates ungrouped if they need individual migration.
- Do not set `open-pull-requests-limit: 0` unless you intentionally disable version updates.

## Multiple manifests in one repository

Use one update entry per ecosystem/directory combination that has an independent dependency graph:

```yaml
version: 2

updates:
  - package-ecosystem: "npm"
    directory: "/frontend"
    schedule: { interval: "weekly" }

  - package-ecosystem: "npm"
    directory: "/docs"
    schedule: { interval: "monthly" }

  - package-ecosystem: "cargo"
    directory: "/src-tauri"
    schedule: { interval: "weekly" }
```

For a directory with its own lockfile outside the workspace, add it separately. Do not create overlapping entries for the same ecosystem and target branch.
