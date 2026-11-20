# Pull request template troubleshooting

## Contents

- [Template not showing](#template-not-showing)
- [Multiple templates](#multiple-templates)
- [Content issues](#content-issues)
- [Checkboxes and enforcement](#checkboxes-and-enforcement)
- [Labels and assignees](#labels-and-assignees)
- [Issue linking](#issue-linking)
- [Validation checklist](#validation-checklist)

## Template not showing

| Check | Fix |
| --- | --- |
| File is on the default branch | Merge the template first; feature branches do not affect the default-branch PR flow. |
| Path is one of the supported locations | `.github/pull_request_template.md`, `pull_request_template.md`, or `docs/pull_request_template.md`. |
| Filename matches the documented spelling | Use `pull_request_template.md` exactly; avoid ad-hoc names for the default template. |
| Only one default template exists | Remove duplicates from root, `docs/`, and `.github/`; named variants go under `PULL_REQUEST_TEMPLATE/`. |
| Repository has PR templates enabled | They are on by default; check repository settings if a template still does not appear. |
| Cached page | Reload the compare page or open a new PR. |

Templates are merged into the default branch; they are not read from the branch being merged.

## Multiple templates

- Named templates live under `.github/PULL_REQUEST_TEMPLATE/` (or `PULL_REQUEST_TEMPLATE/` in root/docs).
- Select one with `?template=name.md`; include the `.md` extension.
- `quick_pull=1` opens the compare page; without it the query may be applied to a different GitHub page.
- There is no automatic chooser UI. Link to the variants from `CONTRIBUTING.md` or a PR comment.
- A PR opened without `template=` uses the default template if one exists.
- If a named template behaves like the default, check for duplicate filenames or a stale default file.

Example:

```text
https://github.com/OWNER/REPO/compare/main...my-branch?quick_pull=1&template=bug_fix.md
```

## Content issues

| Symptom | Cause | Fix |
| --- | --- | --- |
| YAML front matter appears in the PR | PR templates do not process front matter | Remove it; PR templates are Markdown only. |
| HTML instructions appear | Comments are malformed or outside `<!-- ... -->` | Wrap guidance in comments and close them. |
| The template is a wall of text | Too many sections/required questions | Cut to summary, issue, type, test plan, checklist; add variants only when needed. |
| Every PR says "N/A" | Sections do not apply to most changes | Delete those sections or move them to a named template. |
| Commands are stale | Project scripts changed | Update test/build commands after tooling changes. |
| A screenshot request appears for backend changes | UI-only section is in the default template | Move screenshots to the UI variant or mark it clearly optional. |

## Checkboxes and enforcement

- GitHub renders Markdown task lists, but the PR template itself does not enforce them.
- A contributor can submit a PR with every box unchecked.
- If a checkbox matters, enforce it with a workflow that reads the PR body or with a required status check. Do not assume the template is a gate.
- Avoid duplicating CI checks as manual boxes; ask for the test that CI cannot cover.
- For a review checklist, a `PULL_REQUEST_TEMPLATE` plus branch protection and CODEOWNERS is more reliable than a long Markdown list.

## Labels and assignees

PR templates cannot set labels, assignees, milestones, or projects. Options:

- Pre-fill them with query parameters:
  ```text
  ...?quick_pull=1&template=bug_fix.md&labels=bug,needs+review
  ```
- Use a path-based labeler workflow such as `actions/labeler` v7 (pin commit `bf12e9b00b37c5c0ca2b87b79b2daf7891dbda13`) to apply labels from changed files.
- Use a workflow that reads the PR body or title and applies labels through the API.
- Document the limitation so contributors do not expect the template to categorize the PR.

Do not put fake YAML front matter in the template hoping GitHub will apply labels; it will render as text.

## Issue linking

- Use a closing keyword: `Closes #123`, `Fixes #123`, or `Resolves #123`.
- The issue is closed when the PR merges into the default branch, not when the PR is opened.
- Use `Refs #123` if the PR should not close the issue.
- The issue must be in the same repository, or the link must use the full `OWNER/REPO#123` form.
- If the link does not work, check the issue number and whether the PR targets the default branch.

## Validation checklist

- [ ] The template is on the default branch.
- [ ] There is exactly one default template.
- [ ] Named templates are under `PULL_REQUEST_TEMPLATE/`.
- [ ] Query-parameter links use the correct filename including `.md`.
- [ ] Instructions are in HTML comments and do not leak into the rendered PR.
- [ ] Test commands match the current package scripts.
- [ ] Checklist items are verifiable and not redundant with CI.
- [ ] Labels/assignees are handled by query parameters or automation, not the template.
- [ ] `Closes #...` uses a valid issue.
- [ ] Screenshots/secrets guidance is present for UI and log-heavy projects.
