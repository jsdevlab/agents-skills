---
name: github-issue-templates
description: >-
  Use this skill to create or improve GitHub issue templates from a repository's actual project context: bug report and feature request issue forms, classic Markdown templates, `.github/ISSUE_TEMPLATE/config.yml`, template chooser links, field validation, and project-specific prompts for versions, platforms, runtimes, logs, and screenshots. It also covers template labels that do not apply because the label is missing. Trigger it when the user mentions ISSUE_TEMPLATE, issue forms, `bug_report.yml`, `feature_request.yml`, the issue template chooser, structured bug reports, or contributors filing inconsistent issues. Chinese cues — issue 模板, 问题模板, bug 模板, 功能请求, 表单, 模板选择器, 模板标签不生效. Skip pull request templates, generic GitHub Actions CI, and label taxonomy design (use the `github-labels` skill for labels).
---

# GitHub issue templates

An issue template is a file in `.github/ISSUE_TEMPLATE/` that pre-fills an issue's title, labels, assignees, and body. GitHub supports two formats:

- **Issue forms** (`.yml`): structured web forms with inputs, dropdowns, checkboxes, uploads, and required fields. Prefer these for new bug and feature templates.
- **Classic Markdown templates** (`.md`): YAML front matter plus a Markdown body with HTML comments as guidance. Use them for simple templates or when converting an existing workflow.

The goal is not to ask for everything. It is to ask for the few facts maintainers repeatedly have to request, in a form contributors can complete quickly.

## Workflow

1. **Inspect the project** — README, CONTRIBUTING, SECURITY, package manifests, CI, supported versions/platforms, and existing issues. Do not invent fields the project cannot act on.
2. **Choose the template set** — usually bug report, feature request, and documentation; redirect questions and security reports through `config.yml`.
3. **Write forms** — start with the bug form, then feature request. Keep required fields minimal.
4. **Write `config.yml`** — disable blank issues when the forms cover the common cases; add contact links for Discussions and private security reporting.
5. **Resolve labels** — every label referenced by a template must already exist in the repository. Use the `github-labels` skill if the taxonomy is missing or inconsistent.
6. **Validate** — parse YAML, check IDs and names, preview the form, and verify labels with `gh label list`.

## File layout

```text
.github/
├── ISSUE_TEMPLATE/
│   ├── config.yml
│   ├── bug_report.yml
│   ├── feature_request.yml
│   └── documentation.yml
└── workflows/
```

- Templates in `.github/ISSUE_TEMPLATE/` appear in the chooser after they reach the default branch.
- The chooser sorts filenames alphanumerically, with YAML files before Markdown files. Prefix names (`01-bug.yml`, `02-feature.yml`) only when order matters.
- Template `name` values must be unique across YAML and Markdown templates and longer than three characters.
- Issue forms are for issues, not pull requests.

## Bug report form

```yaml
name: Bug report
description: Report a reproducible problem
title: "[Bug]: "
labels: ["bug", "status: needs triage"]
body:
  - type: markdown
    attributes:
      value: |
        Thanks for reporting a bug. Please search existing issues first.

  - type: textarea
    id: summary
    attributes:
      label: Summary
      description: What happened, and what did you expect?
    validations:
      required: true

  - type: textarea
    id: steps
    attributes:
      label: Steps to reproduce
      placeholder: |
        1. Run ...
        2. Open ...
        3. Observe ...
    validations:
      required: true

  - type: input
    id: version
    attributes:
      label: Version
      description: The release, commit, or package version where this occurs.
      placeholder: "v1.2.3"
    validations:
      required: true

  - type: dropdown
    id: os
    attributes:
      label: Operating system
      options:
        - Linux
        - macOS
        - Windows
        - Other
    validations:
      required: true

  - type: textarea
    id: logs
    attributes:
      label: Relevant logs
      description: Paste logs or stack traces. Remove tokens and personal data.
      render: shell
    validations:
      required: false

  - type: upload
    id: screenshots
    attributes:
      label: Screenshots or traces
      description: Drag and drop images, logs, or a small reproduction archive.
    validations:
      required: false
      accept: ".png,.jpg,.jpeg,.gif,.log,.txt,.zip"

  - type: checkboxes
    id: checks
    attributes:
      label: Checklist
      options:
        - label: I searched existing issues and did not find a duplicate.
          required: true
        - label: I am using a supported version.
          required: true
```

Rules that matter:

- `name`, `description`, and `body` are required at the top level.
- `labels` and `assignees` accept an array or a comma-delimited string. Arrays are clearer in forms.
- A label that does not exist in the repository is **silently not applied** — the issue is still created. Verify labels before merging.
- `required: true` is documented as enforced only for public repositories. Do not treat it as server-side validation for private repositories.
- `id` may contain letters, digits, `-`, and `_`; it must be unique. `markdown` elements must not have an `id`.
- `projects` uses `OWNER/PROJECT-NUMBER` and requires the issue opener to have write access to the project.
- `type` is an organization-level issue type, not a label. Leave it out unless the organization defines one.

## Feature request form

```yaml
name: Feature request
description: Suggest a new capability or improvement
title: "[Feature]: "
labels: ["enhancement", "status: needs triage"]
body:
  - type: textarea
    id: problem
    attributes:
      label: Problem
      description: What problem are you trying to solve?
    validations:
      required: true

  - type: textarea
    id: proposal
    attributes:
      label: Proposed solution
      description: Describe the behavior or API you want.
    validations:
      required: true

  - type: textarea
    id: alternatives
    attributes:
      label: Alternatives considered
      description: Workarounds, other tools, or designs you considered.
    validations:
      required: false

  - type: dropdown
    id: scope
    attributes:
      label: Area
      options:
        - CLI
        - API
        - UI
        - Documentation
        - Other
    validations:
      required: false

  - type: checkboxes
    id: checks
    attributes:
      label: Checklist
      options:
        - label: I searched existing issues and discussions.
          required: true
```

Feature requests often need less environment data than bug reports. Ask for the problem first, not a preselected implementation.

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

- `blank_issues_enabled: false` hides the blank issue option from most contributors. Users with Write, Maintain, or Admin roles still see it as "Maintainers only".
- `contact_links` requires `name`, `url`, and `about`.
- Use contact links for questions, security reports, and external support instead of creating a public template that invites sensitive data.

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

<!-- Release, commit, OS, runtime. Remove secrets. -->

### Relevant logs

<!-- Paste logs or stack traces. -->
```

The front matter keys are `name`, `about`, `title`, `labels`, and `assignees`. `about` is the chooser description. `labels` is a comma-delimited string here. HTML comments guide the contributor without appearing in the rendered issue.

## Match fields to the project

| Project type | Useful fields |
| --- | --- |
| Library | package version, language/runtime version, OS, minimal reproduction, expected/actual, logs |
| CLI | command and flags, `--version`, OS/shell, config excerpt, stdout/stderr, exit code |
| Web app | deployment (hosted/self-hosted), browser and version, OS, account plan if relevant, console/network logs |
| Monorepo | package/area dropdown, package version, affected command, workspace manager |
| Desktop app | app version, OS and version, install source, GPU/driver if rendering, logs |
| Mobile app | app version/build, device model, OS version, install source, screenshots/screen recording |
| Service/API | endpoint, request ID, timestamp, status code, sanitized request/response, deployment region |

Do not ask for passwords, tokens, private keys, full customer data, or unredacted production dumps. Provide a private security contact instead.

## Cooperation with `github-labels`

The label taxonomy is owned by the `github-labels` skill. This skill consumes it:

- Before writing `labels:`, read `.github/labels.yml` if it exists, or run `gh label list --json name`.
- Reference only labels that exist. A missing label is silently ignored by GitHub, so the template looks correct but the issue arrives unlabeled.
- When a label is renamed, update the template and the labels manifest in the same change. If the repo uses the label-sync action, use `from_name` to preserve existing issue assignments.
- Do not put priority, status, and type all into free-form labels if the project already uses GitHub Projects fields or issue types; pick one source of truth.

## Reference map

| Task | Read |
| --- | --- |
| Full issue-form schema, field types, validation, uploads | [`references/forms.md`](references/forms.md) |
| Classic templates, `config.yml`, chooser behavior | [`references/classic-and-config.md`](references/classic-and-config.md) |
| Analyze a project and choose templates/fields | [`references/design.md`](references/design.md) |
| Errors, missing labels, forms not appearing | [`references/troubleshooting.md`](references/troubleshooting.md) |

## Common failures

| Symptom | Cause | Fix |
| --- | --- | --- |
| Template does not appear | Wrong directory, not on the default branch, or invalid YAML | Use `.github/ISSUE_TEMPLATE/`, merge to the default branch, parse the YAML. |
| Issue is created without labels | Label does not exist | Create it with the `github-labels` workflow before merging. |
| Form rejects valid YAML | `body` missing, duplicate `id`, or `id` on `markdown` | Fix the top-level keys and IDs; parse the file locally. |
| Required field is not enforced | Private repository | Do not rely on `required` for private repos; validate in triage automation. |
| Project not added | Opener lacks project write access | Use a project auto-add workflow or drop `projects`. |
| Upload rejected | Unsupported extension or size | Check the accepted types and limits; use a link for larger files. |
| Chooser shows no blank option | `blank_issues_enabled: false` | Expected; maintainers still see a "Maintainers only" option. |

## Official links

- Configuring issue templates: <https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/configuring-issue-templates-for-your-repository>
- Issue forms syntax: <https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/syntax-for-issue-forms>
- Form schema: <https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/syntax-for-githubs-form-schema>
- Labels: <https://docs.github.com/en/issues/using-labels-and-milestones-to-track-work/managing-labels>
