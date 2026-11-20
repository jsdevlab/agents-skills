# Label sync with GitHub Actions

A label manifest plus a sync workflow makes labels reviewable, reproducible, and consistent across maintainers. The recommended action is [`crazy-max/ghaction-github-labeler`](https://github.com/crazy-max/ghaction-github-labeler): it reads a YAML file, creates/updates/renames/deletes labels, and supports dry-run and exclusions.

## Contents

- [Manifest](#manifest)
- [Workflow](#workflow)
- [Permissions](#permissions)
- [Dry-run and rollout](#dry-run-and-rollout)
- [skip-delete](#skip-delete)
- [exclude](#exclude)
- [Renames](#renames)
- [Validation](#validation)
- [Alternatives](#alternatives)
- [Pitfalls](#pitfalls)

## Manifest

`.github/labels.yml` is a list of label objects:

```yaml
- name: "type: bug"
  color: "#d73a4a"
  description: "Something is broken or behaves incorrectly"

- name: "type: feature"
  color: "#a2eeef"
  description: "New user-facing capability"

- name: "status: needs triage"
  color: "#fbca04"
  description: "Awaiting maintainer review"

- name: "type: docs"
  color: "#0075ca"
  description: "Documentation improvements"
  from_name: "documentation"
```

| Key | Required | Notes |
| --- | --- | --- |
| `name` | Yes | New label name. |
| `color` | Yes | Six hex digits; the action accepts a leading `#`. |
| `description` | No | Up to 100 characters. If omitted, the action keeps the current description. |
| `from_name` | No | Rename an existing label instead of creating a new one; preserves issue assignments. |

Keep the manifest sorted by category/name so diffs are reviewable. Do not mix colors with and without `#` within the same file unless you have a reason; the action normalizes, but consistency helps review.

## Workflow

```yaml
name: labels

on:
  push:
    branches: [main]
    paths:
      - ".github/labels.yml"
      - ".github/workflows/labels.yml"
  pull_request:
    paths:
      - ".github/labels.yml"
      - ".github/workflows/labels.yml"
  workflow_dispatch:

permissions:
  contents: read
  issues: write

concurrency:
  group: labels
  cancel-in-progress: false

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - uses: crazy-max/ghaction-github-labeler@548a7c3603594ec17c819e1239f281a3b801ab4d # v6.0.0
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          yaml-file: .github/labels.yml
          dry-run: ${{ github.event_name == 'pull_request' }}
          skip-delete: true
          exclude: |
            good first issue
            help wanted
```

- `actions/checkout` is required because the action reads the YAML file from the workspace.
- `dry-run` on pull requests produces a plan without changing labels. A push to `main` applies it.
- `workflow_dispatch` lets a maintainer run the sync manually after a token/permission fix.
- Pin third-party actions to a commit SHA. The comments above name the version; update both together.

## Permissions

Minimum:

```yaml
permissions:
  contents: read
  issues: write
```

- `contents: read` is for `actions/checkout`.
- `issues: write` lets the token create, edit, and delete labels.
- Do not grant `contents: write` for a label sync.
- On `pull_request` from a fork, `GITHUB_TOKEN` is read-only; the dry-run still works, but a real sync must run on a trusted branch or via `workflow_dispatch`.
- Do not use `pull_request_target` just to get a write token for untrusted PR code.

## Dry-run and rollout

First rollout:

1. Export current labels with `gh label list`.
2. Write the manifest and workflow.
3. Open a PR. The `pull_request` run executes with `dry-run: true`; inspect the action log for creates, updates, renames, and deletes.
4. Adjust the manifest until the plan is boring.
5. Merge. The push run applies the changes.
6. Verify with `gh label list --limit 200`.

If the action log is unavailable or truncated, run the same manifest locally with `gh label` commands or add a temporary `workflow_dispatch` run with `dry-run: true`.

## skip-delete

`skip-delete` defaults to `false`, which means **labels not present in the manifest are deleted**. This is the action's most dangerous default.

- During initial adoption, set `skip-delete: true`. Existing labels, including GitHub defaults, survive.
- Once the manifest owns the full taxonomy, set `skip-delete: false` so stale labels are removed.
- If some labels are managed elsewhere, keep `skip-delete: false` and list those labels under `exclude`.

A full-ownership manifest must include every label you want to keep, including `good first issue`, `help wanted`, and any default labels you still use.

## exclude

`exclude` is a newline-delimited list of label names or glob patterns:

```yaml
exclude: |
  good first issue
  help wanted
  dependencies
  renovate*
  *bot
```

Use it for:

- labels managed by another automation (dependency bots, release tooling);
- GitHub defaults you do not want to own yet;
- labels whose descriptions are maintained by an organization policy.

Exclusions are not a substitute for a complete manifest; they are a boundary between owners.

## Renames

To rename `bug` to `type: bug`:

```yaml
- name: "type: bug"
  color: "#d73a4a"
  description: "Something is broken or behaves incorrectly"
  from_name: "bug"
```

The action renames the existing label, preserving issue assignments. Do not also leave a `- name: "bug"` entry; that can create a duplicate or confuse the plan.

Update these in the same PR:

- `.github/ISSUE_TEMPLATE/*.yml` and `*.md` `labels:`;
- workflows/scripts that query `label:bug`;
- saved searches and project automation;
- documentation that tells contributors which label to use.

After the rename applies, verify a sample issue still has the label.

## Validation

- Parse the manifest:
  ```sh
  npx -y yaml valid < .github/labels.yml
  ```
- Check for duplicate names and duplicate `from_name` values.
- Check that every `name` and `from_name` is unique.
- Check descriptions are 100 characters or fewer.
- Check colors are six hex digits (with or without `#`).
- Check every issue-template label exists in the manifest or repository.
- Run the workflow in dry-run and read the plan; do not merge a plan that deletes labels you did not intend to delete.

## Alternatives

- [`EndBug/label-sync`](https://github.com/EndBug/label-sync): YAML/JSON config, delete/force options, widely used.
- `micnncim/action-label-syncer`: older, simple manifest sync.
- `gh label` commands: fine for a one-off change, not for reviewed, repeatable state.

Pick one tool per repository. Running two label syncers with different manifests causes flapping and confusing diffs.

## Pitfalls

| Problem | Cause | Fix |
| --- | --- | --- |
| Labels deleted unexpectedly | `skip-delete` defaults to false | Set `skip-delete: true` or include every label in the manifest. |
| Action cannot find YAML | No checkout step, or wrong `yaml-file` path | Add `actions/checkout`; use a repository-relative path. |
| Permission denied | Missing `issues: write`, or fork PR token is read-only | Add permission; apply on push/manual dispatch. |
| Rename fails | Both old and new names exist | Merge duplicates first, then rename. |
| Dry-run never applies | Workflow only runs on PR | Add a push trigger on the default branch. |
| Labels flap between runs | Two syncers/manifests | Use one source of truth; remove the other workflow. |
| Descriptions truncated/rejected | Over 100 characters | Shorten the description. |
| Color rejected | Wrong format (e.g. named color, 3-digit hex) | Use six hex digits. |
