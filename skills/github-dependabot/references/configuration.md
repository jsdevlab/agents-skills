# `dependabot.yml` configuration reference

The file lives at `.github/dependabot.yml` on the default branch. It has a top-level `version`, optional `registries` and `multi-ecosystem-groups`, and a required `updates` list.

Official reference: <https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file>.

## Contents

- [Skeleton](#skeleton)
- [Required keys](#required-keys)
- [Ecosystems](#ecosystems)
- [Directories](#directories)
- [Schedule](#schedule)
- [Groups](#groups)
- [Multi-ecosystem groups](#multi-ecosystem-groups)
- [Cooldown](#cooldown)
- [Allow and ignore](#allow-and-ignore)
- [PR customization](#pr-customization)
- [Versioning strategy](#versioning-strategy)
- [Private registries](#private-registries)
- [Security updates](#security-updates)
- [Exclude paths and external code](#exclude-paths-and-external-code)
- [Field interactions](#field-interactions)

## Skeleton

```yaml
# yaml-language-server: $schema=https://json.schemastore.org/dependabot-2.0.json
version: 2

registries:
  example:
    type: npm-registry
    url: https://npm.example.com
    token: ${{secrets.NPM_TOKEN}}

multi-ecosystem-groups:
  infrastructure:
    schedule:
      interval: "weekly"

updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
```

## Required keys

| Key | Location | Meaning |
| --- | --- | --- |
| `version` | top level | Always `2`. |
| `updates` | top level | List of ecosystem configurations. |
| `package-ecosystem` | each update | Package manager value. |
| `directory` or `directories` | each update | Manifest location(s). |
| `schedule.interval` | each update | `daily`, `weekly`, `monthly`, `cron` (plus GHES-only values). |

## Ecosystems

| Ecosystem | YAML value | Typical manifests |
| --- | --- | --- |
| npm/pnpm/yarn/Bun | `npm` | `package.json`, lockfiles |
| pip/Poetry/Pipenv | `pip` | `requirements.txt`, `pyproject.toml`, `Pipfile` |
| uv | `uv` | `pyproject.toml`, `uv.lock` |
| Cargo | `cargo` | `Cargo.toml`, `Cargo.lock` |
| Go modules | `gomod` | `go.mod`, `go.sum` |
| Bundler | `bundler` | `Gemfile`, `Gemfile.lock` |
| Composer | `composer` | `composer.json`, `composer.lock` |
| Maven | `maven` | `pom.xml` |
| Gradle | `gradle` | `build.gradle`, `build.gradle.kts` |
| NuGet | `nuget` | `*.csproj`, `packages.config` |
| .NET SDK | `dotnet-sdk` | `global.json` |
| Docker | `docker` | `Dockerfile` |
| Docker Compose | `docker-compose` | `docker-compose.yml` |
| GitHub Actions | `github-actions` | `.github/workflows/*.yml`, action files |
| Terraform | `terraform` | `*.tf` |
| OpenTofu | `opentofu` | `*.tf` |
| Helm | `helm` | `Chart.yaml` |
| Mix | `mix` | `mix.exs` |
| Swift | `swift` | `Package.swift` |
| Pub | `pub` | `pubspec.yaml` |
| Dev Containers | `devcontainers` | `devcontainer.json` |
| Git submodules | `gitsubmodule` | `.gitmodules` |
| pre-commit | `pre-commit` | `.pre-commit-config.yaml` |

Check the official supported-ecosystems page for newer values. Do not invent an ecosystem string.

## Directories

```yaml
- package-ecosystem: "npm"
  directory: "/"
```

```yaml
- package-ecosystem: "npm"
  directories:
    - "/"
    - "/apps/*"
    - "/packages/*"
```

- `directory` is a single path; no globbing.
- `directories` supports globbing and `*`; keep patterns simple and avoid overlapping entries for the same ecosystem/target branch.
- With one root lockfile that covers a workspace (pnpm workspace, Cargo workspace), `directory: "/"` may be enough. Add workspace directories only for independent lockfiles/dependency graphs.
- For GitHub Actions, use `/`; Dependabot scans `.github/workflows/` and root action files.
- For Dev Containers, use `/`; Dependabot scans the standard devcontainer locations.
- If two update blocks for the same ecosystem and target branch overlap, the configuration is ambiguous; make directories unique.

## Schedule

```yaml
schedule:
  interval: "weekly"
  day: "tuesday"
  time: "09:00"
  timezone: "Europe/London"
```

| Key | Notes |
| --- | --- |
| `interval` | `daily` (weekdays), `weekly` (default Monday), `monthly`, or `cron`. `quarterly`, `semiannually`, `yearly` are documented for GHES 3.19+. |
| `day` | Weekly only: `monday`...`sunday`. |
| `time` | `HH:MM`; without it, Dependabot chooses a time. |
| `timezone` | IANA timezone for `time`. |
| `cronjob` | Required when `interval: cron`. |

Example cron:

```yaml
schedule:
  interval: "cron"
  cronjob: "0 9 * * 1"
```

## Groups

```yaml
groups:
  production:
    dependency-type: "production"
    update-types: ["minor", "patch"]

  testing:
    patterns: ["jest*", "vitest*", "@testing-library*"]
    exclude-patterns: ["@testing-library/react-native"]

  security:
    applies-to: "security-updates"
    patterns: ["*"]
    update-types: ["patch", "minor"]
```

| Key | Meaning |
| --- | --- |
| group name | Identifier used in branch/PR titles; starts/ends with a letter, may contain letters, `|`, `_`, `-`. |
| `applies-to` | `version-updates` (default) or `security-updates`. |
| `dependency-type` | `development` or `production` (where supported). |
| `patterns` | Dependency name patterns with `*`. |
| `exclude-patterns` | Exclude matches; wins over `patterns`. |
| `update-types` | `major`, `minor`, `patch`; group-level values. |
| `group-by` | `dependency-name` to group the same dependency across directories. |

First matching group wins. Ungrouped dependencies get individual PRs.

## Multi-ecosystem groups

```yaml
multi-ecosystem-groups:
  infrastructure:
    schedule:
      interval: "weekly"
    labels: ["infrastructure", "dependencies"]
    assignees: ["platform-team"]

updates:
  - package-ecosystem: "docker"
    directory: "/"
    patterns: ["nginx", "redis"]
    multi-ecosystem-group: "infrastructure"

  - package-ecosystem: "terraform"
    directory: "/"
    patterns: ["aws*"]
    multi-ecosystem-group: "infrastructure"
```

- `patterns` is required in every assigned update entry; `["*"]` includes all.
- The group name must match the top-level key.
- Group schedule/labels/assignees live at the group level.
- Grouped branches get an automatic digest suffix.

## Cooldown

```yaml
cooldown:
  default-days: 5
  semver-major-days: 30
  semver-minor-days: 7
  semver-patch-days: 3
  include: ["*"]
  exclude: ["critical-lib"]
```

- Version updates have a **3-day default cooldown** even without this block.
- Security updates are exempt.
- `include`/`exclude` support `*`; exclude wins.
- SemVer-specific days apply only to ecosystems that support SemVer.
- Not every ecosystem supports SemVer-specific cooldowns; `default-days` is the fallback.

## Allow and ignore

```yaml
allow:
  - dependency-type: "production"
  - dependency-name: "express"

ignore:
  - dependency-name: "lodash"
  - dependency-name: "@types/node"
    update-types: ["version-update:semver-patch"]
  - dependency-name: "express"
    versions: ["5.x"]
```

- `allow` narrows the set; `ignore` removes matches.
- If a dependency matches both, it is ignored.
- `ignore.update-types` uses `version-update:semver-patch|minor|major`.
- `ignore.versions` accepts versions/ranges such as `5.x` or `>=2.0.0`.
- `allow.dependency-type` supports `direct`, `indirect`, `all`, `production`, `development` as documented for the ecosystem.
- Prefer `ignore` in the config over `@dependabot ignore` comments so the reason is reviewable.

## PR customization

```yaml
labels: ["dependencies", "npm"]
assignees: ["maintainer"]
milestone: 4
commit-message:
  prefix: "deps"
  prefix-development: "deps-dev"
  include: "scope"
pull-request-branch-name:
  separator: "-"
open-pull-requests-limit: 10
rebase-strategy: "disabled"
```

| Key | Notes |
| --- | --- |
| `labels` | Defaults to `dependencies` plus an ecosystem label; `[]` disables defaults. SemVer labels may still apply. |
| `assignees` | GitHub usernames; must have access. |
| `milestone` | Numeric milestone ID. |
| `commit-message` | `prefix`, `prefix-development`, `include: scope`. |
| `pull-request-branch-name.separator` | Default `/`; supported values are `-`, `_`, and `/`. Related options: `prefix`, `max-length`, `word-separator`, `branch-name-case`, and `template`. |
| `open-pull-requests-limit` | Default 5 for version updates; `0` disables version updates. Security has a separate fixed 10. |
| `rebase-strategy` | `auto` (default) or `disabled`. |

## Versioning strategy

| Value | Behavior |
| --- | --- |
| `auto` | Default: increase for apps, widen for libraries. |
| `increase` | Always increase the minimum version. |
| `increase-if-necessary` | Keep the constraint if it already allows the new version. |
| `lockfile-only` | Only update lockfiles. |
| `widen` | Widen the range to include both versions. |

Supported ecosystems vary; check the options reference before relying on a value.

## Private registries

```yaml
registries:
  npm-private:
    type: npm-registry
    url: https://npm.example.com
    token: ${{secrets.NPM_TOKEN}}

updates:
  - package-ecosystem: "npm"
    directory: "/"
    registries: ["npm-private"]
    schedule:
      interval: "weekly"
```

- Use `${{secrets.NAME}}`, never plaintext.
- Each update block can reference registries defined at the top level.
- Registry types include npm, Docker, Maven, Python, Ruby, NuGet, Composer, and others; see the docs.
- When registries are configured, external code execution is disabled by default. `insecure-external-code-execution: allow` is only supported by some ecosystems and allows a compromised package to access credentials; treat it as a security decision.

## Security updates

Security updates are enabled in repository settings:

1. Enable the dependency graph.
2. Enable Dependabot alerts.
3. Enable Dependabot security updates.
4. Optionally enable grouped security updates.

YAML can group them:

```yaml
groups:
  security:
    applies-to: "security-updates"
    patterns: ["*"]
    update-types: ["patch", "minor"]
```

and disable version updates while keeping security updates:

```yaml
open-pull-requests-limit: 0
```

`target-branch` does not redirect security updates; they still use the default branch.

## Exclude paths and external code

```yaml
exclude-paths:
  - "vendor/**"
  - "test/fixtures/**"
```

- `exclude-paths` skips manifests/files during update scans.
- Do not exclude a directory that contains the only lockfile for an ecosystem you want to update.
- `insecure-external-code-execution: allow` is only for ecosystems whose update process runs manifest code (for example pip, Bundler, Mix); use it only when required and after reviewing the risk.

## Field interactions

| Combination | Behavior |
| --- | --- |
| `directory` + glob | Not supported; use `directories`. |
| `directories` overlap for same ecosystem/target branch | Ambiguous; make them unique. |
| `target-branch` + security updates | Security updates still target the default branch. |
| `open-pull-requests-limit: 0` | Disables version updates, not security updates. |
| `groups` + ungrouped deps | Ungrouped deps get individual PRs. |
| Multiple matching groups | First group wins. |
| `registry` + external code | External code execution disabled unless explicitly allowed. |
| `cooldown` + security updates | Cooldown does not apply to security updates. |
| Multiple `dependabot.yml` files | Not supported; use one file with multiple updates entries. |
