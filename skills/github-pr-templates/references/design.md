# Designing a pull request template

A PR template is a review contract. It should capture the facts a reviewer cannot infer from the diff, without turning every pull request into a form.

## Contents

- [Inspect the project](#inspect-the-project)
- [Choose the sections](#choose-the-sections)
- [Write a useful summary](#write-a-useful-summary)
- [Link the issue](#link-the-issue)
- [Design the test plan](#design-the-test-plan)
- [Design the checklist](#design-the-checklist)
- [Handle risk and breaking changes](#handle-risk-and-breaking-changes)
- [Screenshots and recordings](#screenshots-and-recordings)
- [Length and tone](#length-and-tone)
- [PR templates vs issue templates](#pr-templates-vs-issue-templates)

## Inspect the project

| Source | What to learn |
| --- | --- |
| `README.md` | What the project is, supported platforms, install and test commands. |
| `CONTRIBUTING.md` | Existing review expectations, commit conventions, branch policy. |
| CI workflows | What CI already checks (tests, lint, typecheck, build, security). |
| Issue templates | How contributors describe bugs/features; what to link. |
| Release tooling | Changesets, release-plz, semantic-release, manual changelog. |
| Recent PRs | What reviewers repeatedly ask for; which sections are ignored. |
| `SECURITY.md` | Where vulnerability reports go; do not put secrets in PRs. |

If CI already runs `npm test`, `cargo test`, `go test`, lint, and build, do not ask the contributor to attest to all of them. Ask for the test that is not covered by CI: a manual scenario, a device/browser, a migration rehearsal, a screenshot.

## Choose the sections

A useful default set:

1. **Summary** — what changed and why.
2. **Related issue** — `Closes #123` or `Refs #123`.
3. **Type of change** — bug fix, feature, docs, refactor, breaking change.
4. **Test plan** — exact commands or manual steps; CI link if useful.
5. **Screenshots/recording** — only for UI changes.
6. **Risk / breaking changes / migration** — only when applicable.
7. **Checklist** — a few items that are not automated.

Add project-specific sections:

| Project | Add |
| --- | --- |
| Library | public API diff, semver impact, docs/changelog |
| CLI | command, flags, before/after output, exit code, shell/platform |
| Web app | browser/device, accessibility, screenshots, deployment impact |
| Monorepo | affected packages, dependency graph, per-package tests |
| Desktop/mobile | platform, device, install source, packaging impact |
| Service/API | contract change, migration, rollout/rollback, dashboards |
| Infrastructure | plan output, blast radius, rollback, secret handling |
| Documentation | preview link, affected pages |

Do not add a section just because another repository has it. A section that is always "N/A" trains contributors to ignore the template.

## Write a useful summary

The summary should answer:

- What behavior changed?
- Why is this the right change?
- What is deliberately out of scope?

Weak:

```markdown
## Summary

Fixes stuff.
```

Better:

```markdown
## Summary

Make the CSV exporter stream rows instead of buffering the whole file.
Large exports no longer double peak memory; the output format is unchanged.
```

Keep it short. The diff explains *how*; the summary explains *what* and *why*.

## Link the issue

Use GitHub's closing keywords:

```markdown
Closes #123
```

- `Closes`, `Fixes`, and `Resolves` close the issue when the PR merges into the default branch.
- Use `Refs #123` when the PR is related but should not close the issue.
- Link the issue, not a private tracker ID that reviewers cannot access.
- If there is no issue, explain why the change is small enough to skip one.

## Design the test plan

A good test plan is reproducible:

```markdown
## Test plan

- `pnpm test --filter @acme/csv`
- Manual: export 2 GB of rows with `--stream`; peak RSS stayed below 300 MB.
- Manual: exported file diffed against the previous release for a 10k-row fixture.
```

Bad test plans:

- "Tests pass" — which tests? CI already says that.
- "Manually tested" — what scenario?
- A checklist of every CI job — redundant and easy to fake.

If the change cannot be tested automatically, say exactly what was run, on which platform/device, and what was observed.

## Design the checklist

Keep the checklist short and verifiable:

```markdown
- [ ] I added or updated tests.
- [ ] I updated the documentation.
- [ ] I removed secrets and personal data from logs and screenshots.
```

Avoid:

- items CI already enforces;
- "I followed the code style" (formatters enforce it);
- "I ran the tests" when CI runs them;
- items nobody can verify ("code is clean");
- mandatory boxes that contributors tick without reading.

A checklist is not a gate unless a workflow enforces it. If a check matters, automate it.

## Handle risk and breaking changes

For libraries and APIs, ask:

```markdown
## Breaking changes

<!-- Delete if none. -->

- `Client::new` now returns `Result`.
- Migration: wrap construction in `?` or handle the configuration error.
```

For services/infrastructure, ask for:

- rollout order;
- rollback steps;
- feature flag or migration gate;
- dashboards/alerts to watch;
- blast radius.

For a pure docs or test-only change, delete the section.

## Screenshots and recordings

Ask for screenshots only when the change is visual:

```markdown
## Screenshots

| Before | After |
| --- | --- |
| ... | ... |
```

- Include the platform/browser/device when rendering differs.
- Use a short recording for interaction changes.
- Redact tokens, customer data, and internal URLs.
- Do not attach a 100 MB video; link to an artifact or a short clip.

## Length and tone

- One screen is a good target; two screens is the maximum for a complex project.
- Use HTML comments for guidance; they disappear in the rendered PR.
- Use second person and direct instructions: "Add the command you ran", not "The author should perhaps consider...".
- Do not require a novel. Reviewers can ask follow-up questions.
- Delete sections that do not apply; a template that forces "N/A" everywhere is noise.

## PR templates vs issue templates

| Issue templates | PR templates |
| --- | --- |
| Markdown or YAML issue forms | Markdown only |
| Can set labels, assignees, title, projects | Cannot set labels/assignees; only prefill via query parameters |
| Can require fields in public repos | No validation; checkboxes are not enforced |
| Stored in `.github/ISSUE_TEMPLATE/` | Stored in `.github/`, `docs/`, root, or `PULL_REQUEST_TEMPLATE/` |

Do not paste an issue form's YAML into a PR template. It will render as text, not a form.

If the project uses the `github-issue-templates` skill, link to the issue instead of duplicating its fields. If it uses `github-labels`, do not assume a PR template can apply labels; use query parameters or a labeler workflow.
