# Issue template troubleshooting

## Contents

- [Template not in the chooser](#template-not-in-the-chooser)
- [YAML form errors](#yaml-form-errors)
- [Labels not applied](#labels-not-applied)
- [Fields and validation](#fields-and-validation)
- [config.yml problems](#configyml-problems)
- [Upload problems](#upload-problems)
- [Duplicates and ordering](#duplicates-and-ordering)
- [Validation checklist](#validation-checklist)

## Template not in the chooser

| Check | Fix |
| --- | --- |
| File is in `.github/ISSUE_TEMPLATE/` | Move it there; nested folders are not templates. |
| File is on the default branch | The chooser reads the default branch; merge the PR first. |
| File extension is `.yml` or `.md` | `.yaml` is not the documented extension for issue forms; use `.yml`. |
| YAML parses | A syntax error makes the form disappear without a useful chooser error. Run a parser. |
| `name` is unique and longer than 3 characters | Rename it; duplicate names confuse the chooser. |
| Required top-level keys exist | Forms need `name`, `description`, and `body`. |
| `body` is a non-empty array | An empty or missing body is invalid. |
| `id` values are unique | Duplicate IDs invalidate the form. |
| `markdown` elements have no `id` | Remove the `id`. |
| The repository has Issues enabled | Enable Issues in repository settings. |
| The template page was refreshed | The chooser can be cached; reload after merging. |

Classic Markdown templates also need `name` and `about`; a missing `about` can keep the template out of the chooser.

## YAML form errors

Common parser or schema errors:

- **Tabs**: YAML forbids tabs for indentation. Use spaces.
- **`attributes` indentation**: `label`, `description`, `placeholder`, `options`, and `value` must be nested under `attributes`.
- **`validations` indentation**: `required` and `min_length` must be nested under `validations`.
- **Unquoted `#`**: `value: ## Heading` starts a comment. Quote it or use a block scalar without a leading `#`.
- **Unquoted colon**: `description: Error: failed` or `labels: type: bug` can parse as a mapping. Quote the value.
- **Unknown element type**: only `markdown`, `textarea`, `input`, `dropdown`, `checkboxes`, and `upload` are valid.
- **`id` on `markdown`**: remove it.
- **Missing `id`** on any non-markdown element: add one using letters, digits, `-`, or `_`.
- **Empty `options`** on `dropdown` or `checkboxes`: add at least one distinct option.
- **`default` out of range**: it is a zero-based index into `options`.
- **`None`/`n/a` with a default**: remove the default or the placeholder option.

Parse locally before committing:

```sh
npx -y yaml valid < .github/ISSUE_TEMPLATE/bug_report.yml
```

Or with Python:

```sh
python -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" .github/ISSUE_TEMPLATE/bug_report.yml
```

Parsing only proves YAML is valid; it does not prove the GitHub form schema is valid. Preview the form on the default branch for the final check.

## Labels not applied

GitHub silently ignores a label that does not exist. The issue is created without it, and no error is shown to the contributor.

Check:

```sh
gh label list --limit 200 --json name,color,description
```

Then compare each template's `labels:` entries. Common causes:

- typo or different case (`Bug` vs `bug`);
- label renamed or deleted;
- label exists in another repository but not this one;
- the template was merged before the label was created.

Fix by creating the label first (use the `github-labels` skill), then re-check the template. When renaming a label, update the template and use `from_name` in the label-sync manifest to preserve existing issue assignments.

## Fields and validation

| Symptom | Cause | Fix |
| --- | --- | --- |
| Required field can be submitted empty | Private repository | `required` is documented as enforced only for public repositories; add triage automation. |
| `min_length` ignored | Applied to an element type that does not support it | Use `textarea` or `input`; check the schema. |
| Checkbox option cannot be required | `required` placed on the element instead of the option | Put `required: true` inside the option object. |
| Dropdown allows multiple when it should not | `multiple: true` | Set `multiple: false` or remove it. |
| Pre-filled value rejected | `value` on a `dropdown` instead of `default` | Use a zero-based `default` index. |
| `render` has no effect | Used on `input`/`dropdown` | `render` is for `textarea`. |

## config.yml problems

- `blank_issues_enabled: false` does not hide the blank option from maintainers; they see "Maintainers only". This is expected.
- `contact_links` entries need `name`, `url`, and `about`; a missing field can drop the link.
- A security contact link must point to a private channel (security advisory page or policy), never a public issue.
- `config.yml` is not a template; it should not appear in the chooser.
- Invalid YAML in `config.yml` can break the chooser; parse it like any other file.

## Upload problems

- The file extension must be in the supported list or in `accept`.
- Limits: archives/documents/text 25 MB, images 10 MB, videos 100 MB.
- `accept` is a comma-separated extension string, e.g. `".png,.jpg,.log,.zip"`.
- For files over the limit, ask for a gist, paste, or release-asset link.
- Never ask contributors to upload secrets, private keys, or unredacted customer data.

## Duplicates and ordering

- Two templates with the same `name` can produce duplicate or confusing entries. Keep names unique across YAML and Markdown.
- If a Markdown and YAML template share a filename, delete the Markdown file after converting; otherwise the chooser may show both.
- The chooser sorts filenames alphanumerically, YAML before Markdown. Prefix filenames (`01-bug.yml`, `02-feature.yml`) only when the order matters.
- The `config.yml` file is not sorted with templates.

## Validation checklist

- [ ] All template YAML parses.
- [ ] Each form has `name`, `description`, and a non-empty `body`.
- [ ] Names are unique and longer than 3 characters.
- [ ] IDs are unique, valid, and absent on `markdown` elements.
- [ ] Required fields are genuinely necessary.
- [ ] Labels exist in the repository; no typos.
- [ ] `config.yml` links resolve; security link is private.
- [ ] The chooser and each form were tested on the default branch.
- [ ] No template asks for secrets or unredacted personal data.
- [ ] The template set does not duplicate Discussions/support channels.
