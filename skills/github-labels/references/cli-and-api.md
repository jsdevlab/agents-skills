# `gh label` and the REST API

Use `gh label` for interactive maintenance and the REST API for bulk audits, migrations, or automation that does not run in a workflow.

## Contents

- [Audit](#audit)
- [Create and update](#create-and-update)
- [Rename](#rename)
- [Delete](#delete)
- [Clone](#clone)
- [REST API](#rest-api)
- [Permissions](#permissions)
- [Bulk operations](#bulk-operations)
- [Pitfalls](#pitfalls)

## Audit

```sh
gh label list --limit 200 --json name,color,description \
  --jq '.[] | [.name, .color, .description] | @tsv'
```

- Default limit is 30; set `--limit 200` or page with the API for large repositories.
- `--search` matches names and descriptions.
- `--sort name` gives a stable alphabetical audit; default sort is creation time.
- To find labels used by issue templates:
  ```sh
  rg -n '^\s*labels:' .github/ISSUE_TEMPLATE
  ```

## Create and update

```sh
gh label create "type: bug" --color d73a4a --description "Something is broken"
```

- Color is six hexadecimal digits **without** `#`.
- Description is optional but strongly recommended; keep it under 100 characters.
- `--force` updates an existing label's color and description:
  ```sh
  gh label create "type: bug" --color d73a4a --description "Something is broken" --force
  ```
- `--force` does not rename; use `gh label edit --name` or the sync action's `from_name`.

Update without creating:

```sh
gh label edit "type: bug" --color b60205 --description "Broken behavior"
```

## Rename

```sh
gh label edit "bug" --name "type: bug" --color d73a4a --description "Something is broken"
```

Renaming preserves the label's issue and pull-request assignments. Update issue templates and saved searches in the same change.

If the new name already exists, GitHub rejects the rename. Merge or delete the duplicate first.

## Delete

```sh
gh label delete "obsolete" --yes
```

Deleting removes the label from every issue, pull request, and discussion in the repository. There is no undo. Before deleting:

```sh
gh issue list --label "obsolete" --state all --limit 1000
```

If the concept is still valid, rename instead of deleting.

## Clone

```sh
gh label clone owner/source-repo
gh label clone owner/source-repo --force
```

- Copies every label from the source into the current repository (or `--repo` destination).
- Existing labels are skipped unless `--force`.
- Labels in the destination that are not in the source are left untouched; clone is not a sync.

## REST API

Base path: `/repos/{owner}/{repo}/labels`.

```sh
# List, with pagination
gh api repos/{owner}/{repo}/labels --paginate \
  --jq '.[] | [.name, .color, .description] | @tsv'

# Create
gh api -X POST repos/{owner}/{repo}/labels \
  -f name='type: bug' \
  -f color=d73a4a \
  -f description='Something is broken'

# Rename and update
gh api -X PATCH repos/{owner}/{repo}/labels/bug \
  -f new_name='type: bug' \
  -f color=d73a4a \
  -f description='Something is broken'

# Delete
gh api -X DELETE repos/{owner}/{repo}/labels/obsolete
```

Fields:

| Field | Notes |
| --- | --- |
| `name` | Required on create. Rename with `new_name` on update. |
| `color` | Six hex digits, no leading `#`. |
| `description` | 100 characters or fewer. |

Issue-label operations are easier through `gh issue edit`:

```sh
gh issue edit 123 --add-label "type: bug"
gh issue edit 123 --remove-label "type: bug"
gh issue view 123 --json labels
```

The REST equivalent uses `POST /issues/{issue_number}/labels`, `PUT` to replace the entire label set, and `DELETE /issues/{issue_number}/labels/{name}`. URL-encode spaces and special characters in path segments; for one-off edits, prefer `gh issue edit`.

## Permissions

- Fine-grained token: **Issues: write** on the repository.
- Classic token: `repo` for private repositories; `public_repo` may be enough for public repositories, depending on the operation.
- GitHub Actions: `issues: write`; `contents: read` is enough to check out a label manifest.
- Anyone with write access can create, edit, and delete labels; triage access can apply labels but not edit the label definitions.

Use the least privilege that covers the operation. A label-sync workflow does not need `contents: write`.

## Bulk operations

There is no `gh label` bulk relabel command. For migrations:

1. Export labels and usage.
2. Create the new label (or rename the old one).
3. Page through issues with the old label and add the new one.
4. Remove the old label only after counts match.

Use `gh api --paginate` and check rate limits. Test on a fork or a small repository first.

## Pitfalls

| Problem | Cause | Fix |
| --- | --- | --- |
| `Validation Failed` on create | Color includes `#`, or description is over 100 characters | Pass six hex digits; shorten the description. |
| Rename fails | New name already exists | Merge or delete the duplicate first. |
| Assignments lost | Deleted and recreated instead of renamed | Use `gh label edit --name` or `from_name`. |
| Clone does not remove labels | Clone is not sync | Use the label-sync action for a full manifest. |
| `gh api` 404 on a label name with spaces | Path not encoded | URL-encode the name, or use the label's numeric ID where supported. |
| Rate limited during bulk work | Too many API calls | Batch, cache, and honor `Retry-After`; use conditional requests where possible. |
