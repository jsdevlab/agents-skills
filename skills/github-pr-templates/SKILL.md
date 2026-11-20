---
name: github-pr-templates
description: >-
  Use this skill to create or improve GitHub pull request templates from a repository's actual workflow: a default `pull_request_template.md`, multiple templates in `.github/PULL_REQUEST_TEMPLATE/`, query-parameter selection, project-specific test plans, related-issue links, screenshots, breaking-change notes, and checklists. Trigger it when the user mentions PULL_REQUEST_TEMPLATE, PR template, pull request description template, review checklist, or contributors opening inconsistent pull requests. Chinese cues — PR 模板, 合并请求模板, 拉取请求模板, PR 描述, 审查清单. Skip issue templates and issue forms (use `github-issue-templates`), label taxonomy (use `github-labels`), and generic CI workflow authoring.
---

# GitHub pull request templates

A pull request template pre-fills the PR body when a contributor opens a pull request. It is a Markdown file, not a form: GitHub does not process YAML front matter, does not apply labels or assignees, and does not validate required fields or enforce checkboxes.

The template's job is to make the reviewer's first pass fast: what changed, why, how it was tested, and what could break.

## Workflow

1. **Inspect the project** — README, CONTRIBUTING, CI workflows, issue templates, release tooling, test commands, deployment process, and recent PRs.
2. **Choose one template or a small set** — most repositories need one default; use multiple only when bug fixes, features, docs, and releases genuinely need different questions.
3. **Write short sections** — summary, related issue, type of change, test plan, risk/breaking changes, screenshots when relevant.
4. **Place the file** — `.github/pull_request_template.md` (default) or `.github/PULL_REQUEST_TEMPLATE/<name>.md` (multiple).
5. **Validate** — check the default-branch chooser, test a real PR body, and make sure links/commands are current.

## Default template

```markdown
## Summary

<!-- What changed, and why? Keep it to a few sentences. -->

## Related issue

Closes #

## Type of change

- [ ] Bug fix
- [ ] New feature
- [ ] Documentation
- [ ] Refactor or maintenance
- [ ] Breaking change

## Test plan

<!-- Exact commands, manual steps, or a link to CI. -->

    npm test

## Screenshots

<!-- For UI changes, add before/after screenshots or a short recording. -->

## Breaking changes and migration

<!-- Delete this section if not applicable. -->

## Checklist

- [ ] I added or updated tests.
- [ ] I updated the documentation.
- [ ] I removed secrets and personal data from logs/screenshots.
```

Rules:

- Keep instructions in HTML comments; they are hidden in the rendered PR.
- Delete sections that do not apply to the project. A twelve-section template gets ignored.
- Do not require checklist items that CI already enforces; report them as "CI covers this" instead of asking contributors to duplicate work.
- Ask for a test plan that a reviewer can reproduce, not "tests pass".
- Link the related issue with `Closes #123` or `Refs #123`; this drives GitHub's issue linking and closing behavior.
- For UI changes, ask for screenshots or a recording; for CLI changes, ask for command/output; for libraries, ask for API and semver impact.

## File locations

| Goal | Path |
| --- | --- |
| Default template in the hidden directory | `.github/pull_request_template.md` |
| Default template in the repository root | `pull_request_template.md` |
| Default template in docs | `docs/pull_request_template.md` |
| Multiple templates | `.github/PULL_REQUEST_TEMPLATE/<name>.md` |
| Multiple templates in root or docs | `PULL_REQUEST_TEMPLATE/<name>.md` |

- Templates become available after they are merged into the **default branch**.
- Keep exactly one default template. If files exist in multiple default locations, which one GitHub picks is easy to get wrong; use `.github/pull_request_template.md`.
- Multiple templates are selected with a query parameter:

```text
https://github.com/OWNER/REPO/compare/main...my-branch?quick_pull=1&template=bug_fix.md
```

- Organization or personal-account defaults can live in a `.github` repository as community health files. Use this only when every repository should share the same template.

## Multiple templates

Use multiple templates only when the questions differ materially:

```text
.github/
└── PULL_REQUEST_TEMPLATE/
    ├── bug_fix.md
    ├── feature.md
    ├── documentation.md
    └── release.md
```

- The `template` query parameter takes the filename, including `.md`.
- A PR opened without the parameter uses the default template (if one exists) or no template.
- Document the links in `CONTRIBUTING.md` so contributors can find them.
- Do not create a template per package in a monorepo; use one template with an "Affected packages" checklist instead.

## Match sections to the project

| Project type | Ask for |
| --- | --- |
| Library | public API changes, semver impact, minimal reproduction, tests, docs/changelog |
| CLI | command and flags, before/after output, exit codes, shell/platform, tests |
| Web app | screenshots/recording, browser/device, accessibility notes, deployment impact |
| Monorepo | affected packages, dependency graph impact, per-package tests |
| Desktop/mobile | platform/device, install source, screenshots, packaging impact |
| Service/API | endpoint/contract changes, migration, rollout/rollback, observability |
| Infrastructure | plan output, blast radius, rollback, secrets handling |
| Documentation | preview link, affected pages, screenshots if visual |

## Cooperation with other skills

- **Issue templates** (`github-issue-templates`): link the issue with `Closes #...`; do not duplicate the entire bug form in the PR template.
- **Labels** (`github-labels`): PR templates cannot apply labels. Use the `labels` query parameter for a prefilled link, or a workflow such as `actions/labeler` for path-based labels.
- **Changesets** (`changesets`): in an npm package that uses Changesets, ask whether a changeset was added and which packages it covers.
- **release-plz** (`release-plz`): in a Rust workspace, version bumps and changelogs are generated from conventional commits; ask for the commit type/scope rather than a manual version bump.

## Reference map

| Task | Read |
| --- | --- |
| Analyze the project and design sections | [`references/design.md`](references/design.md) |
| Multiple templates, locations, query parameters, organization defaults | [`references/layout-and-selection.md`](references/layout-and-selection.md) |
| Complete examples for libraries, CLIs, web apps, monorepos, docs, infra | [`references/examples.md`](references/examples.md) |
| Template not showing, duplicate defaults, checkboxes, labels | [`references/troubleshooting.md`](references/troubleshooting.md) |

## Common failures

| Symptom | Cause | Fix |
| --- | --- | --- |
| Template does not appear | File is not on the default branch, or the path/name is wrong | Merge to the default branch; use `.github/pull_request_template.md`. |
| Wrong template wins | Multiple default templates in root/docs/.github | Keep one default; move the rest under `PULL_REQUEST_TEMPLATE/`. |
| Multiple template link does nothing | Missing or wrong `template=` filename | Use `?template=bug_fix.md`; include the extension. |
| Labels are not applied | PR templates do not set labels | Use the `labels` query parameter or a labeling workflow. |
| Contributors ignore the checklist | Template is too long or asks for redundant work | Remove sections; let CI enforce what it can. |
| HTML instructions appear in the PR | Instructions are outside comments | Wrap guidance in `<!-- ... -->`. |

## Official links

- Creating a pull request template: <https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/creating-a-pull-request-template-for-your-repository>
- Query parameters for pull requests: <https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/using-query-parameters-to-create-a-pull-request>
- Default community health files: <https://docs.github.com/en/communities/setting-up-your-project-for-healthy-contributions/creating-a-default-community-health-file>
