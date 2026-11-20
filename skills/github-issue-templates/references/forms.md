# Issue forms reference

Issue forms are YAML files in `.github/ISSUE_TEMPLATE/` that render as structured web forms. On submission, GitHub converts the responses to a Markdown issue body; the contributor can edit the issue afterward.

Official docs: <https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/syntax-for-issue-forms> and <https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/syntax-for-githubs-form-schema>.

## Contents

- [Top-level keys](#top-level-keys)
- [Body elements](#body-elements)
- [Textarea](#textarea)
- [Input](#input)
- [Dropdown](#dropdown)
- [Checkboxes](#checkboxes)
- [Upload](#upload)
- [Markdown](#markdown)
- [Validation rules](#validation-rules)
- [Complete example](#complete-example)

## Top-level keys

```yaml
name: Bug report
description: Report a reproducible problem
title: "[Bug]: "
labels: ["bug", "status: needs triage"]
assignees: ["maintainer"]
projects: ["owner/1"]
type: bug
body:
  - type: markdown
    attributes:
      value: "Thanks for the report."
```

| Key | Required | Type | Notes |
| --- | --- | --- | --- |
| `name` | Yes | string | Unique across all templates, including Markdown templates. Longer than three characters. |
| `description` | Yes | string | Shown in the template chooser. Keep it short and specific. |
| `body` | Yes | array | Ordered form elements. |
| `title` | No | string | Pre-filled issue title, e.g. `[Bug]: `. |
| `labels` | No | array or comma-delimited string | Labels applied automatically. A label that does not exist is silently ignored. |
| `assignees` | No | array or comma-delimited string | Users must have at least read access to the repository. |
| `projects` | No | array or comma-delimited string | Format `OWNER/PROJECT-NUMBER`; the issue opener needs write access to the project. |
| `type` | No | string | Organization-level issue type, not a label. Leave unset unless the organization defines it. |

## Body elements

`body` is an ordered array. Each element has a `type`, an optional `id`, and type-specific `attributes` and `validations`.

| Type | Purpose | Has `id` |
| --- | --- | --- |
| `markdown` | Display guidance; not submitted | No |
| `textarea` | Multi-line text, optional code rendering | Yes |
| `input` | Single-line text | Yes |
| `dropdown` | Single or multi-select menu | Yes |
| `checkboxes` | One or more checkboxes | Yes |
| `upload` | File upload | Yes |

`id` may contain letters, digits, `-`, and `_`; it must be unique within the form. The ID is used for URL query-parameter prefills.

## Textarea

```yaml
- type: textarea
  id: logs
  attributes:
    label: Relevant logs
    description: Paste logs or a stack trace. Remove secrets first.
    placeholder: "Paste output here"
    value: ""
    render: shell
  validations:
    required: false
    min_length: 20
```

| Attribute | Notes |
| --- | --- |
| `label` | Field label displayed to the contributor. |
| `description` | Guidance under the label. |
| `placeholder` | Grey placeholder text. |
| `value` | Pre-filled text. |
| `render` | Renders submitted text as a fenced code block in the given language (`shell`, `bash`, `json`, ...). Disables attachments and Markdown editing in that field. |

Validations: `required` (public repositories only), `min_length` (non-negative integer).

## Input

```yaml
- type: input
  id: version
  attributes:
    label: Version
    description: Release, commit, or package version.
    placeholder: "v1.2.3"
  validations:
    required: true
    min_length: 3
```

Single-line field. Same `required` and `min_length` validations as `textarea`. Use it for versions, URLs, environment names, and short identifiers—not for logs.

## Dropdown

```yaml
- type: dropdown
  id: area
  attributes:
    label: Area
    description: Which part of the project is affected?
    multiple: false
    options:
      - CLI
      - API
      - UI
      - Documentation
    default: 0
  validations:
    required: true
```

| Attribute | Notes |
| --- | --- |
| `multiple` | Allow more than one selection. |
| `options` | Non-empty array of distinct strings. |
| `default` | Zero-based index of the preselected option. When set, do not include `None` or `n/a` as options. |

Use dropdowns for closed sets (supported versions, OS families, areas). If the list changes every release, an `input` may be cheaper to maintain.

## Checkboxes

```yaml
- type: checkboxes
  id: checks
  attributes:
    label: Checklist
    description: Confirm the basics before submitting.
    options:
      - label: I searched existing issues.
        required: true
      - label: I am using a supported version.
        required: false
  validations:
    required: false
```

Each option has a `label` and optional `required`. Markdown bold, italics, and links are supported in option labels. The element-level `required` is separate from each option's `required`.

Do not gate submission on acknowledgements that contributors cannot verify; keep required checks factual.

## Upload

```yaml
- type: upload
  id: attachments
  attributes:
    label: Attachments
    description: Screenshots, logs, or a small reproduction archive.
  validations:
    required: false
    accept: ".png,.jpg,.jpeg,.gif,.log,.txt,.zip"
```

`accept` is a comma-separated list of extensions. If omitted, all supported types are accepted.

Supported types and limits:

| Category | Extensions | Limit |
| --- | --- | --- |
| Archives | `.zip`, `.gz`, `.tar.gz` | 25 MB |
| Documents | `.pdf`, `.docx`, `.xlsx`, `.pptx` | 25 MB |
| Images | `.png`, `.jpg`, `.jpeg`, `.gif`, `.svg`, `.webp` | 10 MB |
| Videos | `.mp4`, `.mov`, `.webm` | 100 MB |
| Text | `.json`, `.py`, `.js`, `.ts`, `.log`, `.txt`, `.csv` | 25 MB |

For larger artifacts, ask for a link to a gist, paste, or release asset; do not ask contributors to paste secrets.

## Markdown

```yaml
- type: markdown
  attributes:
    value: |
      ## Before you submit

      - Search existing issues.
      - Remove tokens and personal data from logs.
```

`markdown` is display-only and is not included in the submitted issue body. It must not have an `id`.

YAML treats `#` as a comment. Quote Markdown headings:

```yaml
value: "## Before you submit"
```

Or use a block scalar without a leading `#`:

```yaml
value: |
  **Before you submit**
```

## Validation rules

- `required: true` prevents submission until the field is completed. GitHub documents this as enforced only for **public** repositories.
- `min_length` applies to `textarea` and `input`.
- `checkboxes` options can each be required.
- `upload` supports `accept` to restrict extensions.
- YAML must parse. Duplicate `id` values, `id` on `markdown`, unknown element types, and missing `body` are common causes of a form that never appears.

## Complete example

```yaml
name: Bug report
description: Report a reproducible problem
title: "[Bug]: "
labels: ["bug", "status: needs triage"]
body:
  - type: markdown
    attributes:
      value: |
        Thanks for reporting a bug. Search existing issues first and remove
        secrets from logs.

  - type: textarea
    id: summary
    attributes:
      label: Summary
      description: What happened, and what did you expect?
    validations:
      required: true

  - type: input
    id: version
    attributes:
      label: Version
      placeholder: "v1.2.3"
    validations:
      required: true
```
