# Locations, multiple templates, and selection

GitHub looks for pull request templates in a few fixed locations and selects multiple templates with a query parameter. The details are easy to get wrong because a file can exist in the repository and still never appear.

## Contents

- [Default locations](#default-locations)
- [Multiple templates](#multiple-templates)
- [Query parameters](#query-parameters)
- [Default community health files](#default-community-health-files)
- [Monorepos](#monorepos)
- [Migration](#migration)

## Default locations

Use one of these for a single repository-wide template:

| Location | Path |
| --- | --- |
| Hidden directory (recommended) | `.github/pull_request_template.md` |
| Repository root | `pull_request_template.md` |
| Docs directory | `docs/pull_request_template.md` |

- Use the exact lower-case filename from the docs. Keep one default file; do not duplicate it in root, `docs/`, and `.github/`.
- Templates are read from the **default branch**. A template on a feature branch does not affect pull requests opened against the default branch.
- The file is plain Markdown. YAML front matter is not interpreted; it will show up in the PR body.
- HTML comments (`<!-- ... -->`) are hidden in the rendered PR but visible while editing. Use them for instructions.

## Multiple templates

Create a `PULL_REQUEST_TEMPLATE` subdirectory in `.github/`, the repository root, or `docs/`:

```text
.github/
├── pull_request_template.md          # default
└── PULL_REQUEST_TEMPLATE/
    ├── bug_fix.md
    ├── feature.md
    ├── documentation.md
    └── release.md
```

- A PR opened without a `template` query parameter uses the default template if one exists.
- Named templates are selected explicitly. They are not a chooser UI like issue forms; the contributor follows a link or adds the parameter.
- Document the links in `CONTRIBUTING.md`:

```markdown
- [Bug fix PR](https://github.com/OWNER/REPO/compare/main...HEAD?quick_pull=1&template=bug_fix.md)
- [Feature PR](https://github.com/OWNER/REPO/compare/main...HEAD?quick_pull=1&template=feature.md)
- [Docs PR](https://github.com/OWNER/REPO/compare/main...HEAD?quick_pull=1&template=documentation.md)
```

- Keep the set small. Four templates is usually the maximum before contributors guess.
- Do not create a template per package; use one template with an "Affected packages" checklist.

## Query parameters

The `template` parameter fills the PR body from a file under `PULL_REQUEST_TEMPLATE/`:

```text
https://github.com/OWNER/REPO/compare/main...my-branch?quick_pull=1&template=bug_fix.md
```

The value is the filename, including `.md`. If the file is in a nested path, use the path relative to `PULL_REQUEST_TEMPLATE/`.

Other useful parameters:

| Parameter | Effect |
| --- | --- |
| `quick_pull=1` | Opens the "Open a pull request" page for the compared branches. |
| `title=...` | Pre-fills the title; use `+` for spaces. |
| `body=...` | Pre-fills the body; use `+` for spaces. |
| `labels=bug,help+wanted` | Pre-fills labels (comma-separated). |
| `assignees=octocat` | Pre-fills assignees. |
| `milestone=...` | Pre-fills a milestone. |
| `projects=OWNER/NUMBER` | Adds the PR to a project. |

Query parameters are a convenience, not enforcement. A contributor can open a PR without them, and a PR template cannot apply labels or assignees by itself. The user needs permission for the equivalent action; invalid parameters or insufficient permissions return `404 Not Found`.

## Default community health files

Organizations and personal accounts can put default community health files in a public `.github` repository. A default `pull_request_template.md` there applies to repositories that do not have their own.

Use this for:

- a shared review checklist across many repositories;
- organization-wide security and contribution links;
- a consistent PR summary format.

Do not use it to enforce repository-specific test commands; those belong in the repository's own template.

## Monorepos

One template with an "Affected packages" section is usually better than one template per package:

```markdown
## Affected packages

- [ ] `@acme/core`
- [ ] `@acme/cli`
- [ ] `@acme/web`
- [ ] docs
```

Ask for:

- the packages changed;
- whether a dependency graph change affects downstream packages;
- the tests run per package;
- whether a changeset (Changesets) or conventional commit/release-plz flow is required.

If a package has a genuinely different review process (for example, an infrastructure package with a plan output), add a named template for that case.

## Migration

To move from a root template to the hidden directory:

1. Create `.github/pull_request_template.md`.
2. Copy the content and update links/commands.
3. Delete the old root/docs template.
4. Test with a new PR against the default branch.
5. Update `CONTRIBUTING.md` links.

To split one template into multiple:

1. Keep the common sections in the default template.
2. Move the variant-specific sections into named templates.
3. Add query-parameter links to `CONTRIBUTING.md`.
4. Test each link on the default branch.
5. Remove sections from the default that no longer apply.
