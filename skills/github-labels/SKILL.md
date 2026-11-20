---
name: github-labels
description: >-
  Use this skill to design, create, audit, rename, and sync GitHub repository labels: a label taxonomy for type, status, priority, area, size, and contributor labels; consistent colors and descriptions; `gh label` commands; a `.github/labels.yml` source of truth; and a GitHub Actions label-sync workflow with dry-run, skip-delete, exclude rules, and safe renames. Trigger it when the user mentions GitHub labels, label taxonomy, triage labels, `labels.yml`, `gh label`, label sync, or issue templates that reference missing labels. Chinese cues — GitHub 标签, 标签体系, 标签同步, 分类标签, triage, 标签改名. Skip issue template authoring (use the `github-issue-templates` skill) and general project management.
---

# GitHub labels

Labels are repository-scoped tags applied to issues, pull requests, and discussions. They are cheap to create and expensive to keep consistent: a taxonomy with synonyms, missing descriptions, and ad-hoc colors becomes noise that contributors and automation cannot rely on.

This skill treats labels as code: one manifest, one sync workflow, and a stable contract shared with issue templates.

## Workflow

1. **Audit** — export current labels and compare them with the labels actually used by templates, automation, and saved searches.
2. **Design** — choose a naming system, categories, colors, and descriptions. Keep the set small.
3. **Write `.github/labels.yml`** — make it the source of truth, including renames via `from_name`.
4. **Sync** — use `crazy-max/ghaction-github-labeler` on push, with `dry-run` on pull requests and a safe deletion policy.
5. **Validate** — confirm issue templates, workflows, and documentation reference only labels that exist.
6. **Maintain** — review after releases and support spikes; rename instead of delete when possible.

## Quick start

`.github/labels.yml`:

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

- name: "priority: high"
  color: "#b60205"
  description: "Should be addressed in the current cycle"

- name: "area: cli"
  color: "#5319e7"
  description: "Command-line interface"

- name: "good first issue"
  color: "#7057ff"
  description: "Good for first-time contributors"
```

`.github/workflows/labels.yml`:

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

permissions:
  contents: read
  issues: write

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
```

- `actions/checkout` is required: the labeler reads the YAML file from the workspace.
- `dry-run` on pull requests previews changes without applying them; a push to `main` applies them.
- `skip-delete: true` keeps labels that are not in the manifest. Use it for the first rollout, then decide whether the manifest owns the full set.
- Pin third-party actions to a commit SHA; the tags above are informational.

## Taxonomy

Choose **one** naming system and use it everywhere:

| System | Examples | Trade-offs |
| --- | --- | --- |
| Bare GitHub defaults | `bug`, `enhancement`, `documentation` | Familiar; mixes categories; harder to group in a long label list. |
| Prefixed categories | `type: bug`, `status: needs triage`, `priority: high`, `area: cli` | Groups labels alphabetically; explicit automation; requires a rename migration. |

Do not mix `bug` and `type: bug`. Do not create both `enhancement` and `type: feature`.

Common categories:

| Category | Examples | Notes |
| --- | --- | --- |
| Type | `type: bug`, `type: feature`, `type: docs`, `type: chore` | One type per issue. |
| Status | `status: needs triage`, `status: confirmed`, `status: blocked`, `status: waiting for author` | Keep the workflow short. |
| Priority | `priority: high`, `priority: medium`, `priority: low` | Or use a GitHub Projects priority field, not both. |
| Area | `area: cli`, `area: api`, `area: ui`, `area: ci` | Mirror the repository's packages/components. |
| Size | `size: s`, `size: m`, `size: l` | Optional; useful for planning, often stale. |
| Contributor | `good first issue`, `help wanted` | Use these exact names; `good first issue` powers the contribute page. |

A set of 15–30 labels is usually enough. Every label should answer a question triage actually asks.

## Colors and descriptions

- Color is six hexadecimal digits; the REST API and `gh label` take it **without** `#`. The label-sync YAML accepts `#`.
- Descriptions are capped at 100 characters. Write an imperative description of when to use the label, not a restatement of its name.
- Use color families consistently: red for bugs/blockers, orange/yellow for attention, green for ready/approved, blue for documentation, purple for areas, grey for size/chore.
- GitHub chooses the text color based on the background; check contrast for light colors such as `#ffffff`.
- Avoid emoji in labels that automation or saved searches must match; if you use them, use them consistently.

## CLI and API

Audit:

```sh
gh label list --limit 200 --json name,color,description \
  --jq '.[] | [.name, .color, .description] | @tsv'
```

Create or update:

```sh
gh label create "type: bug" --color d73a4a --description "Something is broken"
gh label create "type: bug" --color d73a4a --description "Something is broken" --force
```

Rename (preserves the label's issue assignments):

```sh
gh label edit "bug" --name "type: bug" --color d73a4a --description "Something is broken"
```

Delete:

```sh
gh label delete "obsolete" --yes
```

Clone labels from another repository:

```sh
gh label clone owner/source-repo
```

`gh label create --force` updates the color and description if the label exists; it does not rename. Use `gh label edit --name` or the sync action's `from_name` for renames.

## Cooperation with `github-issue-templates`

The issue-template skill consumes this taxonomy. The contract:

1. `.github/labels.yml` (or the repository's label settings) is the source of truth for names, colors, and descriptions.
2. Every `labels:` entry in `.github/ISSUE_TEMPLATE/*.yml` and classic `*.md` templates must exist. GitHub **silently ignores** a missing label; the issue is created unlabeled.
3. Renaming a label requires a coordinated change:
   - add `from_name: "old-name"` to the manifest entry so the sync action renames instead of delete+create;
   - update every template reference (`.github/ISSUE_TEMPLATE/*.yml`, `*.md`);
   - grep workflows, scripts, and docs for `label:old-name` / `"old-name"`;
   - run the sync action in dry-run on the PR, then apply on merge.
4. Do not create a label only because a template references it; either add it to the taxonomy with a clear purpose or remove the reference.
5. For triage, prefer `status: needs triage` plus one `type:` label. Remove the status label once triage is complete.

## Reference map

| Task | Read |
| --- | --- |
| Taxonomy, naming, colors, descriptions, migration | [`references/taxonomy.md`](references/taxonomy.md) |
| `gh label` commands, REST API, audits, renames, deletes | [`references/cli-and-api.md`](references/cli-and-api.md) |
| `.github/labels.yml`, sync workflow, permissions, dry-run, skip-delete | [`references/sync-action.md`](references/sync-action.md) |
| Issue-template compatibility, renames, validation | [`references/template-integration.md`](references/template-integration.md) |

## Common failures

| Symptom | Cause | Fix |
| --- | --- | --- |
| Sync action deletes unexpected labels | `skip-delete` defaults to `false`, so labels absent from the manifest are removed | Set `skip-delete: true` during rollout, include all managed labels, or use `exclude`. |
| Label color rejected | `#` included in `gh label`/REST color | Pass six hex digits without `#`. |
| Rename creates a new label and loses assignments | Deleted and recreated instead of renamed | Use `gh label edit --name` or `from_name` in the manifest. |
| Template issue arrives unlabeled | Label does not exist or has a typo | Add it to the taxonomy; verify with `gh label list`. |
| Workflow cannot create labels | Token lacks `issues: write` | Add the permission; `contents: read` is enough for checkout. |
| Labels differ between repositories | Labels are repository-scoped | Use the manifest/action per repository, or clone a source repo. |
| Saved searches break after a rename | Searches still use the old label name | Update saved filters and automation in the same change. |

## Official links

- Managing labels: <https://docs.github.com/en/issues/using-labels-and-milestones-to-track-work/managing-labels>
- REST labels API: <https://docs.github.com/en/rest/issues/labels>
- `gh label`: <https://cli.github.com/manual/gh_label>
- Label sync action: <https://github.com/crazy-max/ghaction-github-labeler>
- Issue templates: <https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/configuring-issue-templates-for-your-repository>
