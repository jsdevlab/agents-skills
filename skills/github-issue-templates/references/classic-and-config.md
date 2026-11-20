# Classic templates and the chooser

GitHub supports classic Markdown issue templates alongside YAML issue forms. The chooser is customized by `.github/ISSUE_TEMPLATE/config.yml`.

Official docs: <https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/configuring-issue-templates-for-your-repository>.

## Contents

- [Classic Markdown template](#classic-markdown-template)
- [Front matter keys](#front-matter-keys)
- [Writing the body](#writing-the-body)
- [config.yml](#configyml)
- [Chooser behavior](#chooser-behavior)
- [Converting to a form](#converting-to-a-form)
- [Testing](#testing)

## Classic Markdown template

```markdown
---
name: Bug report
about: Report a reproducible problem
title: "[Bug]: "
labels: bug, needs triage
assignees: ''
---

### Summary

<!-- What happened, and what did you expect? -->

### Steps to reproduce

<!-- 1. ... 2. ... 3. ... -->

### Version and environment

<!-- Release, commit, OS, runtime. Remove tokens and personal data. -->

### Relevant logs

<!-- Paste logs or stack traces. -->
```

The contributor sees the headings and comments in the issue editor. HTML comments are hidden when the issue is rendered, which makes them useful as writing prompts.

## Front matter keys

| Key | Required | Notes |
| --- | --- | --- |
| `name` | Yes | Chooser label. Must be unique across all templates and longer than three characters. |
| `about` | Yes | Short description shown in the chooser. |
| `title` | No | Pre-filled title. |
| `labels` | No | Comma-delimited string, e.g. `bug, needs triage`. Labels must already exist or they are silently ignored. |
| `assignees` | No | Comma-delimited usernames with at least read access. |

Classic templates cannot enforce required fields. If you need validation, use an issue form.

If label names contain `: ` (for example `type: bug`), quote the entire value:

```yaml
labels: "type: bug, status: needs triage"
```

Without quotes, YAML reads the colon as a nested mapping and the template fails to load.

## Writing the body

- Use `###` headings for the facts you need: summary, reproduction, expected behavior, environment, logs.
- Put instructions in HTML comments next to each heading.
- Keep the body short. A wall of headings discourages reports.
- Do not ask for secrets, tokens, passwords, or unredacted customer data.
- Link to `CONTRIBUTING.md`, `SECURITY.md`, and the support channel instead of duplicating long policies.
- For logs, ask for a fenced code block or a file upload; for large files, ask for a link.

## `config.yml`

```yaml
blank_issues_enabled: false
contact_links:
  - name: Questions and usage help
    url: https://github.com/OWNER/REPO/discussions
    about: Ask usage questions and share ideas in Discussions.
  - name: Report a security vulnerability
    url: https://github.com/OWNER/REPO/security/advisories/new
    about: Please report vulnerabilities privately; do not open a public issue.
```

| Key | Meaning |
| --- | --- |
| `blank_issues_enabled` | `true` shows the blank issue option to everyone with access. `false` hides it from Read/Triage roles; Write/Maintain/Admin still see "Maintainers only". |
| `contact_links` | External links shown in the chooser. Each needs `name`, `url`, and `about`. |

Use `contact_links` for:

- usage questions that belong in Discussions, Discord, Stack Overflow, or a support portal;
- security vulnerabilities that must be reported privately;
- documentation sites or status pages;
- commercial support channels.

Do not use a public issue template to collect vulnerability details.

## Chooser behavior

- Templates are read from `.github/ISSUE_TEMPLATE/` on the **default branch**.
- Files are listed alphanumerically, with YAML forms before Markdown templates.
- `config.yml` itself is not shown as a template.
- If two files define the same `name`, the chooser can show duplicates or reject the form; keep names unique.
- If a YAML form is invalid, it is omitted from the chooser; check the repository's template page after merging.
- Issue forms apply to issues, not pull requests. Pull request templates live in `.github/PULL_REQUEST_TEMPLATE.md` or `.github/PULL_REQUEST_TEMPLATE/` and are a separate feature.

## Converting to a form

To convert a classic template:

1. Create a new `.yml` form with the same `name` and `about`/`description`.
2. Map each Markdown heading to an element: summary/reproduction → `textarea`, version/environment → `input` or `dropdown`, logs → `textarea` with `render`, screenshots → `upload`, acknowledgements → `checkboxes`.
3. Move writing guidance from HTML comments into `attributes.description` and `markdown` elements.
4. Add `validations.required` only for facts maintainers truly cannot triage without.
5. Delete the old `.md` file if you want the same filename; otherwise keep both temporarily and accept that the chooser shows two entries.
6. Verify labels and preview the form before merging.

Do not mechanically copy every heading. A form with twelve required fields is worse than a short Markdown template.

## Testing

- Parse every YAML file with a YAML parser (for example `npx yaml valid < file` or `python -c` with PyYAML).
- Open the repository's **New issue** page after the files reach the default branch; the chooser is not rendered from a feature branch.
- Submit a test issue with each form, then delete it. Verify the title prefix, labels, assignees, and body formatting.
- Check that labels exist with `gh label list --json name`.
- Confirm that `config.yml` links resolve and that security links point to a private channel.
- Check the form on a narrow screen; long dropdowns and many required fields are painful on mobile.
