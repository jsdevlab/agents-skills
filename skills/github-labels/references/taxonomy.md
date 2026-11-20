# Label taxonomy

A label taxonomy is a small vocabulary for triage, planning, and automation. It should be stable, mutually understandable, and small enough to remember.

## Contents

- [Audit first](#audit-first)
- [Choose a naming system](#choose-a-naming-system)
- [Categories](#categories)
- [Color palette](#color-palette)
- [Descriptions](#descriptions)
- [Migration](#migration)
- [Renames and deletions](#renames-and-deletions)
- [Maintenance](#maintenance)

## Audit first

Export the current labels and where they are used:

```sh
gh label list --limit 200 --json name,color,description \
  --jq '.[] | [.name, .color, .description] | @tsv'
```

Then inspect:

- `.github/ISSUE_TEMPLATE/*.yml` and `*.md` for `labels:` references;
- `.github/workflows/` and scripts for `label:` queries or label names;
- saved issue searches and project automation;
- recent issues for labels that are applied inconsistently or never used.

A label used by no template, query, or maintainer is a candidate for deletion. A label used by a template but missing from the repository is a broken template.

## Choose a naming system

### Bare names

```text
bug
enhancement
documentation
question
good first issue
help wanted
```

Pros: familiar; matches GitHub defaults and the contribute page.
Cons: categories mix; `bug` and `priority` sit next to each other alphabetically; scaling to areas and statuses gets messy.

### Prefixed categories

```text
type: bug
type: feature
type: docs
status: needs triage
status: blocked
priority: high
area: cli
area: api
size: m
good first issue
help wanted
```

Pros: groups labels alphabetically; makes the category explicit in templates and automation; scales to areas and statuses.
Cons: requires migrating default labels; saved searches and templates must be updated.

Pick one. Do not keep both `bug` and `type: bug`. If you migrate, rename the old labels rather than deleting them so existing issues keep their labels.

## Categories

| Category | Purpose | Examples |
| --- | --- | --- |
| Type | What kind of work is this? One per issue. | `type: bug`, `type: feature`, `type: docs`, `type: chore` |
| Status | Where is it in the workflow? | `status: needs triage`, `status: confirmed`, `status: blocked`, `status: waiting for author` |
| Priority | How urgent is it? | `priority: high`, `priority: medium`, `priority: low` |
| Area | Which component/package? | `area: cli`, `area: api`, `area: ui`, `area: ci` |
| Size | Estimated effort. | `size: s`, `size: m`, `size: l` |
| Contributor | Good entry points. | `good first issue`, `help wanted` |

Keep the workflow short. A status set of `needs triage`, `confirmed`, `blocked`, and `waiting for author` is usually enough; adding `in progress`, `ready for review`, and `done` duplicates project boards.

Use priority labels or a GitHub Projects priority field, not both. If the project uses Projects for planning, labels should describe the issue, not its position in a board.

`good first issue` and `help wanted` are special: GitHub uses `good first issue` to populate the repository's contribute page. Use the exact names.

## Color palette

Use color families consistently. A workable palette:

| Family | Color | Use |
| --- | --- | --- |
| Red | `#d73a4a` | Bugs, blockers, high priority |
| Orange | `#d93f0b` | Urgent attention, breaking changes |
| Yellow | `#fbca04` | Needs triage, waiting, caution |
| Green | `#0e8a16` | Confirmed, ready, low priority |
| Blue | `#0075ca` | Documentation, help wanted |
| Purple | `#5319e7` | Areas/components |
| Grey | `#ededed` | Size, chore, metadata |
| White | `#ffffff` | `wontfix` if you keep the default look |

GitHub chooses black or white text based on the background luminance. Check a very light color such as white in the UI; if it is hard to read, use a light grey instead.

Colors are six hexadecimal digits. `gh label` and the REST API expect them **without** `#`; the label-sync YAML accepts `#`.

## Descriptions

Descriptions are capped at 100 characters. Write when to use the label:

Good:

- `Something is broken or behaves incorrectly`
- `New user-facing capability`
- `Awaiting maintainer review`
- `Blocked by another issue or decision`

Weak:

- `Bug` (restates the name)
- `This label is for bugs` (adds nothing)
- `Use this when the thing is not working as expected and you want the team to look at it` (too long)

Keep descriptions stable; they appear in the label picker and in `gh label list`.

## Migration

To move from defaults to prefixed labels:

1. Write `.github/labels.yml` with the new names and `from_name` entries for the old ones:
   ```yaml
   - name: "type: bug"
     color: "#d73a4a"
     description: "Something is broken or behaves incorrectly"
     from_name: "bug"
   ```
2. Update every issue template reference from `bug` to `type: bug`.
3. Update workflows, scripts, saved searches, and docs.
4. Run the sync action in dry-run on a pull request; inspect the plan.
5. Merge; the action renames the label and keeps issue assignments.
6. Remove the temporary compatibility labels only after the migration settles.

Do not delete the old label first. Deleting removes it from issues; a later rename cannot recover the assignments.

## Renames and deletions

- **Rename** when the concept stays the same. `gh label edit old --name new` preserves issue assignments. The sync action's `from_name` does the same.
- **Delete** when the concept is obsolete. `gh label delete` removes the label from all issues and pull requests. Export usage first:
  ```sh
  gh issue list --label "old-label" --state all --limit 1000
  ```
- **Merge** two labels by moving issues to the surviving label, then deleting the duplicate. There is no bulk relabel command in `gh`; use the API or a script, and verify counts.
- **Retire** a label by removing it from templates and automation first, then deleting it after a deprecation window.

## Maintenance

- Review labels after a major release, a support spike, or a reorganization.
- Remove labels that are never used; keep the chooser and label picker short.
- Check that every template label exists after a rename.
- Keep colors and descriptions in the manifest, not only in the GitHub UI; otherwise the next sync overwrites or deletes them.
- If multiple repositories share a taxonomy, keep one source manifest and clone/sync it deliberately; labels are repository-scoped.
- Do not use labels to store secrets, customer names, or sensitive incident details; labels are visible to anyone who can see the issue.
