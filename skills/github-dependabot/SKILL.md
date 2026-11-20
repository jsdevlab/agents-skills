---
name: github-dependabot
description: >-
  Use this skill to configure and operate GitHub Dependabot: `.github/dependabot.yml`, version updates, security updates, grouped and multi-ecosystem updates, schedules, cooldowns, private registries, monorepo directories, labels, commit messages, and Dependabot PR commands. Trigger it when the user mentions Dependabot, `dependabot.yml`, dependency update PRs, grouping dependency bumps, security update PRs, or reducing dependency PR noise. Chinese cues — Dependabot, 依赖更新, 依赖自动升级, 依赖分组, 安全更新, 依赖 PR. Skip general package-manager usage, Renovate, and CI workflows that do not involve Dependabot.
---

# GitHub Dependabot

Dependabot has three related features:

| Feature | What it does | Where configured |
| --- | --- | --- |
| Dependabot alerts | Notifies you about vulnerable dependencies | Repository/organization security settings |
| Dependabot security updates | Opens PRs to fix vulnerable dependencies | Repository/organization security settings |
| Dependabot version updates | Opens PRs to keep dependencies current | `.github/dependabot.yml` |

The YAML file configures **version updates** and some **security update grouping**. Alerts and security updates themselves are enabled in GitHub settings, not in YAML.

## Workflow

1. **Detect ecosystems and manifests** — scan the repository; do not guess from the primary language.
2. **Map directories** — one `directory`, or `directories` for multiple locations; monorepos need globs.
3. **Write minimal entries** — ecosystem, directory/directories, schedule.
4. **Reduce noise** — groups, schedules, cooldowns, labels, commit messages, PR limits.
5. **Handle security updates** — enable them in settings; optionally group them in YAML and disable version updates.
6. **Validate** — parse YAML, check the dependency graph page, and review the first PRs.

## Minimal configuration

```yaml
# yaml-language-server: $schema=https://json.schemastore.org/dependabot-2.0.json
version: 2

updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "monthly"
```

Rules:

- The file is `.github/dependabot.yml` on the default branch. GitHub uses one file per repository; use multiple `updates` entries instead of multiple files.
- `version: 2` and `updates` are required.
- Every `updates` entry needs `package-ecosystem`, `directory` or `directories`, and `schedule.interval`.
- `package-ecosystem: "github-actions"` with `directory: "/"` scans `.github/workflows/` and a root `action.yml`/`action.yaml`.
- YAML is whitespace-sensitive; validate it before committing.

## Ecosystems

Common values:

| Manifest/tool | `package-ecosystem` |
| --- | --- |
| npm, pnpm, yarn, Bun | `npm` |
| pip, Poetry, Pipenv | `pip` |
| uv | `uv` |
| Cargo | `cargo` |
| Go modules | `gomod` |
| Bundler | `bundler` |
| Composer | `composer` |
| Maven / Gradle | `maven` / `gradle` |
| NuGet / .NET SDK | `nuget` / `dotnet-sdk` |
| Docker / Docker Compose | `docker` / `docker-compose` |
| GitHub Actions | `github-actions` |
| Terraform / OpenTofu | `terraform` / `opentofu` |
| Helm | `helm` |
| Mix (Elixir) | `mix` |
| Swift | `swift` |
| Pub (Dart/Flutter) | `pub` |
| Dev Containers | `devcontainers` |
| Git submodules | `gitsubmodule` |
| pre-commit | `pre-commit` |

The full list is in the official options reference; check the current value for newer ecosystems instead of inventing one.

## Monorepos

Use `directories` for multiple manifest locations:

```yaml
- package-ecosystem: "npm"
  directories:
    - "/"
    - "/apps/*"
    - "/packages/*"
  schedule:
    interval: "weekly"
```

- `directory` is a single path and does **not** support globbing.
- `directories` supports globbing and the `*` wildcard. Do not assume every shell glob feature is supported; keep patterns simple.
- For the same ecosystem and target branch, entries must not overlap directories.
- Start with `directory: "/"` when one root lockfile covers the workspace (pnpm workspace, Cargo workspace). Add package directories only when they have independent lockfiles or dependency graphs.
- A directory with its own lockfile and separate dependency graph may need its own entry even if it is inside the monorepo.
- `groups.<name>.group-by: dependency-name` groups the same dependency across multiple directories into one PR (version updates only, same ecosystem, incompatible constraints may split).

## Grouping

```yaml
groups:
  production-minor:
    dependency-type: "production"
    update-types: ["minor", "patch"]

  development:
    dependency-type: "development"
    update-types: ["minor", "patch"]

  testing:
    patterns: ["jest*", "@testing-library*", "vitest*"]
    exclude-patterns: ["@testing-library/react-native"]
```

- A dependency joins the **first** matching group; later groups do not see it.
- `applies-to` defaults to `version-updates`; use `security-updates` for security PRs.
- `update-types` in groups accepts `major`, `minor`, `patch`; `ignore` uses the `version-update:semver-*` form.
- `dependency-type` in groups supports `development` and `production`, and only for package managers that expose that distinction.
- Ungrouped dependencies still get individual PRs.

## Multi-ecosystem groups

Combine Docker, Terraform, and other ecosystems in one PR:

```yaml
version: 2

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

- `patterns` is required in each ecosystem entry assigned to a multi-ecosystem group; use `["*"]` for all dependencies.
- All entries must use the same group name.
- Group schedules and labels/assignees live in `multi-ecosystem-groups`, not in each update entry.
- Grouped PRs get a short digest suffix in the branch name automatically.

## Schedules and cooldowns

```yaml
schedule:
  interval: "weekly"
  day: "monday"
  time: "09:00"
  timezone: "Europe/Berlin"
```

- `interval`: `daily` (weekdays), `weekly` (default Monday), `monthly`, or `cron`. `quarterly`, `semiannually`, and `yearly` are documented for GitHub Enterprise Server 3.19+; verify the target platform.
- `cron` requires `cronjob`.
- Without `time`, Dependabot picks a time; `time`/`timezone` make it deterministic.
- Dependabot applies a **3-day default cooldown** to version updates even when `cooldown` is not configured. Security updates are exempt.
- Customize cooldowns:

```yaml
cooldown:
  default-days: 5
  semver-major-days: 30
  semver-minor-days: 7
  semver-patch-days: 3
  include: ["*"]
  exclude: ["critical-lib"]
```

## Security updates

Enable Dependabot alerts and security updates in repository settings first. YAML can:

- group security updates:
  ```yaml
  groups:
    security-patches:
      applies-to: security-updates
      patterns: ["*"]
      update-types: ["patch", "minor"]
  ```
- disable version updates while leaving security updates on:
  ```yaml
  open-pull-requests-limit: 0
  ```
- define private registries used for both version and security updates.

Security updates are limited to 10 open PRs and cannot be configured with `open-pull-requests-limit`. If `target-branch` is set, version updates target that branch, but security updates still target the default branch.

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

- Default labels are `dependencies` plus an ecosystem label; SemVer labels (`major`, `minor`, `patch`) apply automatically when present.
- `labels: []` disables all labels, including defaults, but SemVer labels may still apply.
- `milestone` is the numeric milestone ID, not the title.
- `pull-request-branch-name.separator` supports `-`, `_`, or `/`. Other branch-name options include `prefix`, `max-length`, `word-separator`, `branch-name-case`, and `template`.
- `rebase-strategy: disabled` stops automatic rebasing; existing PRs may continue for a while.
- `open-pull-requests-limit` defaults to 5 for version updates; security updates have a separate fixed limit of 10.

## Versioning strategy

`versioning-strategy` controls manifest edits for supported ecosystems:

| Value | Behavior |
| --- | --- |
| `auto` | Default: increase for apps, widen for libraries. |
| `increase` | Always increase the minimum requirement. |
| `increase-if-necessary` | Keep the requirement if it already allows the new version. |
| `lockfile-only` | Only update lockfiles. |
| `widen` | Widen the range to include old and new versions. |

Not every ecosystem supports every value; check the official options reference.

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

- Registry credentials must use `${{secrets.NAME}}`, not plaintext.
- Each update block can reference registries from the top-level `registries` section.
- When registry access is configured, Dependabot disables external code execution by default. `insecure-external-code-execution: allow` (only some ecosystems) re-enables it and lets a compromised package access credentials; use only with a clear risk decision.

## PR commands

On a Dependabot PR, comment:

| Command | Effect |
| --- | --- |
| `@dependabot rebase` | Rebase the PR. |
| `@dependabot recreate` | Recreate the PR, discarding edits. |
| `@dependabot ignore this dependency` | Stop updates for this dependency. |
| `@dependabot ignore this major version` | Stop updates for this major. |
| `@dependabot ignore this minor version` | Stop updates for this minor. |
| `@dependabot ignore this patch version` | Stop updates for this patch. |
| `@dependabot show DEPENDENCY_NAME ignore conditions` | Show stored ignore conditions. |

Grouped PRs also support `@dependabot ignore DEPENDENCY_NAME`, `@dependabot unignore DEPENDENCY_NAME`, `@dependabot unignore *`, and `@dependabot unignore DEPENDENCY_NAME IGNORE_CONDITION`.

Ignore commands store repository-level preferences. Prefer explicit `ignore` rules in `dependabot.yml` so the reason is reviewable. Merge/close/reopen via comments is not part of the current command reference; use the GitHub UI, `gh pr merge`, or auto-merge.

## Validation and troubleshooting

- Parse `.github/dependabot.yml`; add the SchemaStore comment for editor autocomplete.
- Confirm the file is on the default branch and is the only `dependabot.yml`.
- Check **Insights → Dependency graph → Dependabot** for the configured ecosystems and errors.
- If no PRs appear: check schedule, `open-pull-requests-limit`, `ignore`/`allow`, `target-branch`, cooldown, and whether the manifest/lockfile is on the configured branch.
- If a PR is noisy: group by dependency type/name, increase `cooldown`, switch low-priority ecosystems to monthly, and use `exclude-paths` for fixtures/vendored code.
- If a security update does not appear: confirm alerts/security updates are enabled in settings; YAML grouping alone does not enable them.
- If a private registry update fails: check the token secret name, registry URL, `registries` reference, and whether external code execution is required.

## Reference map

| Task | Read |
| --- | --- |
| Full option reference and field interactions | [`references/configuration.md`](references/configuration.md) |
| Real-world configurations for npm, Rust, Docker, Actions, monorepos, security-only | [`references/recipes.md`](references/recipes.md) |
| PR commands, grouping operations, rebase/auto-merge, troubleshooting | [`references/operations.md`](references/operations.md) |

## Official links

- Dependabot options reference: <https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file>
- Supported ecosystems: <https://docs.github.com/en/code-security/dependabot/ecosystems-supported-by-dependabot/supported-ecosystems-and-repositories>
- Grouped updates: <https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/optimizing-pr-creation-version-updates>
- Multi-ecosystem updates: <https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/configuring-multi-ecosystem-updates>
- PR comment commands: <https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-pull-request-comment-commands>
- Security updates: <https://docs.github.com/en/code-security/dependabot/dependabot-security-updates/configuring-dependabot-security-updates>
