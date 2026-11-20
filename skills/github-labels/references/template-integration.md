# Labels and issue-template integration

Issue templates and labels are one contract. The template says which labels an issue should receive; GitHub only applies labels that already exist. A missing label is silently ignored, so the failure is invisible until triage notices the issue is unlabeled.

This reference covers the shared contract, validation, and safe renames. The template authoring side lives in the `github-issue-templates` skill.

## Contents

- [Contract](#contract)
- [Audit templates against labels](#audit-templates-against-labels)
- [Validation script](#validation-script)
- [Adding a label for a template](#adding-a-label-for-a-template)
- [Renaming a label used by templates](#renaming-a-label-used-by-templates)
- [Deleting or merging labels](#deleting-or-merging-labels)
- [Triage labels](#triage-labels)
- [Checklist](#checklist)

## Contract

1. `.github/labels.yml` is the source of truth for managed label names, colors, and descriptions.
2. Every `labels:` entry in `.github/ISSUE_TEMPLATE/*.yml` and `*.md` must exist in the repository.
3. Templates use the exact label name as defined, including spaces and prefixes (`type: bug`, not `type:bug`); avoid case-only variants such as `Bug` and `bug`.
4. The labels manifest is reviewed in the same PR as template changes when the two depend on each other.
5. A label is added only if it has a triage purpose. Do not create labels just to satisfy a template.

Form `labels` can be an array:

```yaml
labels: ["type: bug", "status: needs triage"]
```

Or a comma-delimited string:

```yaml
labels: "type: bug, status: needs triage"
```

Classic Markdown templates use the comma-delimited string form:

```markdown
---
name: Bug report
about: Report a reproducible problem
labels: "type: bug, status: needs triage"
---
```

YAML arrays are less error-prone when labels contain commas; prefer arrays in forms.

## Audit templates against labels

List repository labels:

```sh
gh label list --limit 200 --json name --jq '.[].name' | sort
```

Extract template labels:

```sh
rg -n '^\s*labels:' .github/ISSUE_TEMPLATE
```

Compare manually or with the script below. Common mismatches:

- `bug` vs `type: bug`;
- `needs triage` vs `status: needs triage`;
- `documentation` vs `type: docs`;
- trailing spaces after commas in classic templates;
- a label that was renamed in the manifest but not in the template.

## Validation script

Commit this as `.github/scripts/check-template-labels.py` and run it in CI or locally. It reads `.github/labels.yml`, YAML issue forms, and classic Markdown front matter.

```python
#!/usr/bin/env python3
import glob
import sys

import yaml

LABELS_FILE = ".github/labels.yml"
TEMPLATES = ".github/ISSUE_TEMPLATE"


def label_names(value):
    if value is None:
        return []
    if isinstance(value, list):
        return [str(item).strip() for item in value if str(item).strip()]
    return [item.strip() for item in str(value).split(",") if item.strip()]


def main():
    with open(LABELS_FILE, encoding="utf-8") as handle:
        labels = {item["name"] for item in yaml.safe_load(handle) if "name" in item}

    missing = []
    for path in sorted(glob.glob(f"{TEMPLATES}/*.yml")):
        with open(path, encoding="utf-8") as handle:
            data = yaml.safe_load(handle) or {}
        for name in label_names(data.get("labels")):
            if name not in labels:
                missing.append((path, name))

    for path in sorted(glob.glob(f"{TEMPLATES}/*.md")):
        with open(path, encoding="utf-8") as handle:
            text = handle.read()
        if not text.startswith("---"):
            continue
        front = text.split("---", 2)[1]
        data = yaml.safe_load(front) or {}
        for name in label_names(data.get("labels")):
            if name not in labels:
                missing.append((path, name))

    if missing:
        for path, name in missing:
            print(f"{path}: label not in {LABELS_FILE}: {name!r}")
        return 1

    print(f"OK: all template labels exist in {LABELS_FILE}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Run locally:

```sh
python -m pip install pyyaml
python .github/scripts/check-template-labels.py
```

CI job:

```yaml
name: template-labels

on:
  pull_request:
    paths:
      - ".github/labels.yml"
      - ".github/ISSUE_TEMPLATE/**"
      - ".github/scripts/check-template-labels.py"
      - ".github/workflows/template-labels.yml"

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - uses: actions/setup-python@6d4fd22b20465ccc0f7354476588e8df7c9ca0e0 # v7.1.0
        with:
          python-version: "3.x"
      - run: python -m pip install pyyaml
      - run: python .github/scripts/check-template-labels.py
```

This job validates references against the manifest, not against the live repository. If labels are managed outside the manifest, compare with `gh label list` instead or add those labels to the manifest.

## Adding a label for a template

1. Define the label in `.github/labels.yml` with a description and color.
2. Add the exact name to the template's `labels:`.
3. Open one PR containing both changes.
4. The label-sync action runs on merge; the dry-run on the PR shows the plan.
5. After merge, open a test issue with the template and confirm the label is applied.

Do not create the label only in the GitHub UI; the next sync may delete it if `skip-delete` is false.

## Renaming a label used by templates

1. Update the manifest entry:
   ```yaml
   - name: "type: bug"
     color: "#d73a4a"
     description: "Something is broken or behaves incorrectly"
     from_name: "bug"
   ```
2. Update every template reference from `bug` to `type: bug`.
3. Update workflows, scripts, saved searches, and docs that use `label:bug`.
4. Open one PR with the manifest and template changes.
5. Inspect the label-sync dry-run. It should show a rename, not a delete/create.
6. Merge and verify a sample issue still has the label.

If the template is updated before the label exists, the merge window can create unlabeled issues. Merge the manifest and templates together; the sync runs on the same push.

## Deleting or merging labels

- Remove the label from templates first, then delete it after the deprecation window.
- To merge two labels, move issues to the surviving label, verify counts, then delete the duplicate.
- Check saved searches and automation before deleting; a deleted label disappears from issues and queries.
- Do not delete a label merely because it is absent from the manifest while `skip-delete: true`; it may be managed by another team.

## Triage labels

A simple triage contract:

| Stage | Labels |
| --- | --- |
| New issue | `status: needs triage` (set by template) |
| Triaged | remove `status: needs triage`; add `type: bug` / `type: feature` and an `area:` |
| Blocked | add `status: blocked` |
| Waiting on reporter | add `status: waiting for author` |
| Closed as duplicate/invalid | use GitHub's close reason where possible; keep labels minimal |

Avoid setting both a `priority:` label and a Projects priority field. Pick the system the team actually reviews.

## Checklist

- [ ] Every template label exists in `.github/labels.yml` or the live repository.
- [ ] Label names match exactly, including case and prefix.
- [ ] Manifest and template changes are in the same PR.
- [ ] Renames use `from_name` and preserve assignments.
- [ ] Dry-run shows no unintended deletions.
- [ ] A test issue receives the expected labels.
- [ ] Saved searches and automation use the new names after a rename.
- [ ] Security reports do not go through a public label/template path.
