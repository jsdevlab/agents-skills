# Operating Dependabot

Configuration is only half the job. This reference covers reviewing and merging Dependabot PRs, comment commands, rebasing, auto-merge, and troubleshooting.

## Contents

- [Review workflow](#review-workflow)
- [PR comment commands](#pr-comment-commands)
- [Rebase behavior](#rebase-behavior)
- [Auto-merge](#auto-merge)
- [Grouped PRs](#grouped-prs)
- [Security updates](#security-updates)
- [Troubleshooting](#troubleshooting)
- [Validation checklist](#validation-checklist)

## Review workflow

A practical loop:

1. Let CI run; do not review a red dependency PR.
2. Check the release notes/changelog for major updates.
3. For grouped PRs, inspect the individual commits/versions, not just the title.
4. Merge patch/minor updates that pass CI; hold major updates for migration review.
5. Let the lockfile/manifest change merge before opening follow-up work.
6. If an update is unwanted, add a reviewable `ignore` rule instead of relying on a comment-only ignore.

Dependabot PRs are normal PRs: branch protection, required checks, CODEOWNERS, and merge queues apply.

## PR comment commands

| Command | Effect |
| --- | --- |
| `@dependabot rebase` | Rebase the PR. |
| `@dependabot recreate` | Recreate the PR, discarding edits. |
| `@dependabot ignore this dependency` | Close and stop updates for this dependency. |
| `@dependabot ignore this major version` | Stop updates for this major. |
| `@dependabot ignore this minor version` | Stop updates for this minor. |
| `@dependabot ignore this patch version` | Stop updates for this patch. |
| `@dependabot show DEPENDENCY_NAME ignore conditions` | Show the stored ignore conditions. |

For grouped PRs:

| Command | Effect |
| --- | --- |
| `@dependabot ignore DEPENDENCY_NAME` | Ignore this dependency in the group. |
| `@dependabot ignore DEPENDENCY_NAME major version` | Ignore its major updates. |
| `@dependabot ignore DEPENDENCY_NAME minor version` | Ignore its minor updates. |
| `@dependabot ignore DEPENDENCY_NAME patch version` | Ignore its patch updates. |
| `@dependabot unignore DEPENDENCY_NAME` | Clear the dependency's ignore conditions and open a new PR. |
| `@dependabot unignore *` | Clear all ignore conditions in the group. |
| `@dependabot unignore DEPENDENCY_NAME IGNORE_CONDITION` | Clear one specific condition. |

Dependabot reacts with a thumbs-up and may comment. Ignore commands store repository-level preferences; for a team, put the reason in `dependabot.yml` instead.

The current command reference does **not** include `@dependabot merge`, `close`, or `reopen`. Use the GitHub UI, `gh pr merge`, or auto-merge.

## Rebase behavior

- Dependabot rebases open PRs when it detects changes; by default it stops rebasing a PR after 30 days.
- If someone pushes an extra commit to a Dependabot branch, Dependabot stops rebasing it. Adding `[dependabot skip]`, `[skip dependabot]`, `[dependabot-skip]`, or `[skip-dependabot]` to a commit message lets Dependabot force-push over the commit on the next rebase.
- `rebase-strategy: "disabled"` stops automatic rebasing. Existing PRs may continue for up to 30 days.
- A rebase can invalidate CI results; wait for the new run before merging.

## Auto-merge

Auto-merge is a GitHub PR feature, not a Dependabot command.

```sh
gh pr merge <number> --auto --squash
```

Requirements:

- The repository allows auto-merge.
- Branch protection/required checks are configured.
- The token has permission to enable auto-merge (`pull-requests: write`, and `contents: write` for the merge).

A common workflow is to auto-merge only patch updates:

```yaml
permissions:
  contents: write
  pull-requests: write

steps:
  - uses: dependabot/fetch-metadata@25dd0e34f4fe68f24cc83900b1fe3fe149efef98 # v3.1.0
    id: metadata
  - run: gh pr merge --auto --squash "$PR_URL"
    if: steps.metadata.outputs.update-type == 'version-update:semver-patch'
    env:
      PR_URL: ${{ github.event.pull_request.html_url }}
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Check the action's current README for inputs/outputs and required permissions. Do not auto-merge major updates without a migration review.

## Grouped PRs

- Grouped PRs contain multiple dependency updates; review the commits and the combined changelog.
- If one dependency is problematic, use the group-specific `@dependabot ignore DEPENDENCY_NAME` command or split groups in YAML.
- Cross-directory groups (`group-by: dependency-name`) produce one PR per dependency across directories; incompatible constraints can still split.
- Multi-ecosystem groups combine different package managers; one failing ecosystem can block the whole PR. Keep critical ecosystems in separate groups if that risk is unacceptable.

## Security updates

- Security updates are enabled in repository settings, not by `dependabot.yml`.
- Grouped security updates can be enabled in settings and/or configured with `groups.applies-to: security-updates`.
- `open-pull-requests-limit: 0` disables version updates but leaves security updates enabled.
- Security updates have a separate fixed limit of 10 open PRs.
- `target-branch` affects version updates; security updates still target the default branch.
- Prioritize security PRs by severity and exploitability, not only by semver level.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| No version-update PRs | Schedule, `open-pull-requests-limit`, `ignore`/`allow`, `target-branch`, cooldown, manifest on the configured branch. |
| No security-update PRs | Dependency graph, alerts, and security updates enabled in settings; advisory applies to the manifest; PR limit not reached. |
| Config ignored | File is at `.github/dependabot.yml` on the default branch; YAML parses; one file only. |
| Invalid config | Check **Insights → Dependency graph → Dependabot** for errors; validate YAML; check ecosystem-specific values. |
| Too many PRs | Groups, longer schedule, cooldown, `exclude-paths`, higher `open-pull-requests-limit` doesn't help reduce count; lower it or group. |
| PRs target the wrong branch | `target-branch`; security updates still use the default branch. |
| Private registry 401/404 | Registry type/URL, secret name, `registries` reference, token permissions. |
| External code execution error | Registry access disables it by default; review `insecure-external-code-execution`. |
| Rebase conflicts | Resolve manually or use `@dependabot rebase`; check branch protection and extra commits. |
| Auto-merge not enabling | Repository auto-merge setting, branch protection, token permissions. |

## Validation checklist

- [ ] `.github/dependabot.yml` is on the default branch and is the only Dependabot config file.
- [ ] `version: 2` and at least one `updates` entry.
- [ ] Every entry has an ecosystem, directory/directories, and schedule.
- [ ] Ecosystems match actual manifests; pnpm/yarn use `npm`.
- [ ] Mono-repo directories are non-overlapping for the same ecosystem/target branch.
- [ ] Groups have clear names and first-match behavior is understood.
- [ ] Multi-ecosystem entries all define `patterns` and the same group name.
- [ ] Cooldowns and schedules match the team's risk tolerance.
- [ ] Security updates are enabled in settings; version-only disable uses `open-pull-requests-limit: 0`.
- [ ] Private registry secrets are referenced via `${{secrets.NAME}}`.
- [ ] First run reviewed in the dependency graph page and on a real PR.
