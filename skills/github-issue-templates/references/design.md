# Designing issue templates from a project

A good template encodes what maintainers repeatedly ask for. Start from the repository, not from a generic bug-report template.

## Contents

- [Inspect the project](#inspect-the-project)
- [Choose the template set](#choose-the-template-set)
- [Design the bug form](#design-the-bug-form)
- [Design the feature form](#design-the-feature-form)
- [Project-specific fields](#project-specific-fields)
- [Required vs optional](#required-vs-optional)
- [Privacy and security](#privacy-and-security)
- [Labels contract](#labels-contract)
- [Maintenance](#maintenance)

## Inspect the project

Read these before writing:

| Source | What to learn |
| --- | --- |
| `README.md` | What the project is, supported platforms, install methods, versioning. |
| `CONTRIBUTING.md` | Existing reporting rules, test commands, code of conduct. |
| `SECURITY.md` | Where vulnerabilities should go. Never create a public vulnerability form. |
| Package manifests | Language, runtime, package manager, supported versions, entry points. |
| CI workflows | Test/lint commands, supported OS matrix, toolchain versions. |
| Recent issues and PRs | The questions maintainers ask repeatedly; the fields contributors omit. |
| Existing labels | The vocabulary the project already uses. Do not invent parallel terms. |
| Existing templates | What to keep, merge, or remove; avoid duplicate chooser entries. |
| Releases/tags | How versions are named and which versions are supported. |

If the repository is a monorepo, inspect each package's version and support policy. If it ships a CLI, inspect `--help` and exit codes. If it ships a service, inspect deployment modes and request IDs.

## Choose the template set

A typical set:

| Template | Use when | Format |
| --- | --- | --- |
| Bug report | Users can observe incorrect behavior | Issue form |
| Feature request | Users can describe a problem and a proposed capability | Issue form |
| Documentation | Docs have a specific gap or error | Issue form or Markdown |
| Question / usage | Questions belong in Discussions or a support channel | `contact_links`, not a public template |
| Security | Vulnerabilities must be private | `contact_links` to the security advisory page |
| Blank issue | Maintainers need an escape hatch | Keep `blank_issues_enabled: false`; maintainers still see it |

Do not add a template for every label. Three to five well-designed templates are usually enough. A "question" form that invites account details, logs, and screenshots can create a public support queue you did not ask for.

## Design the bug form

Minimum useful fields:

1. **Summary** — what happened and what was expected. One `textarea`, required.
2. **Reproduction** — exact steps, commands, or a minimal repository. One `textarea`, required.
3. **Version** — release/commit/package version. `input` or `dropdown`, required if supported versions matter.
4. **Environment** — OS, runtime, browser, device, or deployment mode. `dropdown` or `input`, required when the bug is environment-dependent.
5. **Logs** — `textarea` with `render: shell` or `upload`, optional.
6. **Screenshots/traces** — `upload`, optional.
7. **Checklist** — searched issues, latest version, read contributing guide. `checkboxes`, required only in public repos.

Avoid:

- asking for a full config file when the relevant excerpt is enough;
- requiring logs for bugs that are purely visual;
- asking for a version dropdown with every historical release;
- asking contributors to diagnose the root cause;
- putting tokens, cookies, or customer data in the issue body.

## Design the feature form

Ask for:

1. **Problem** — the user need, not a preselected implementation. Required.
2. **Proposed solution** — behavior or API shape. Required.
3. **Alternatives** — workarounds and other designs. Optional.
4. **Scope/area** — dropdown for CLI/API/UI/docs or monorepo packages. Optional.
5. **Breaking-change impact** — checkbox or textarea when the project has a compatibility policy.

Do not require a design document. Link to a RFC/discussion process if the project has one.

## Project-specific fields

| Project | Add | Avoid |
| --- | --- | --- |
| Library | package version, language/runtime, minimal reproduction, expected/actual | full application logs unrelated to the library |
| CLI | command/flags, `--version`, shell, stdout/stderr, exit code | screenshots of a terminal when text is available |
| Web app | deployment mode, browser/version, OS, console/network logs | asking for account credentials |
| Monorepo | package/area dropdown, package version, workspace manager | one giant environment textarea |
| Desktop | app version, OS version, install source, GPU/driver when rendering | assuming all users build from source |
| Mobile | app version/build, device model, OS version, install source, recording | asking for device identifiers |
| Service/API | endpoint, request ID, timestamp, status code, sanitized request/response | full production payloads with personal data |
| Hardware/IoT | model, firmware, connection type, serial-number-free logs | asking for serial numbers in public |

Use `dropdown` for closed sets and `input` for free-form versions. Use `upload` for screenshots, logs, and traces; use a link for files above the upload limit.

## Required vs optional

Make a field required only when triage is impossible without it. Good candidates:

- reproduction steps for a bug;
- version for a versioned product;
- platform where behavior differs;
- the problem statement for a feature request.

Poor candidates:

- logs that many users cannot access;
- screenshots for non-visual bugs;
- a "root cause" or "fix" field;
- acknowledgements that are not factual.

GitHub documents `required` as enforced only in public repositories. In private repositories, do not rely on it for compliance; add triage automation or a maintainer check.

## Privacy and security

- Never ask for passwords, API tokens, private keys, session cookies, or full customer records.
- Ask contributors to redact logs and config.
- Route vulnerabilities to `SECURITY.md` or the private advisory URL in `config.yml`, not a public template.
- Use `contact_links` for support channels that require account details.
- If the project has a privacy policy, link to it instead of restating it.

## Labels contract

Issue templates and labels are one system:

- `labels:` only references labels that exist in the repository.
- A missing label is silently ignored; the issue is created unlabeled.
- The canonical label list belongs in `.github/labels.yml` or the repository's label settings, managed by the `github-labels` skill.
- When renaming a label, update the template and the label manifest in the same change. With `crazy-max/ghaction-github-labeler`, use `from_name` on the label definition to preserve existing issue assignments.
- Do not invent a `priority: urgent` label just for a template if the project already uses GitHub Projects priority fields; choose one source of truth.

## Maintenance

- Review templates after a major release or a support spike.
- Update version dropdowns when support windows change; stale options generate invalid reports.
- Remove templates that duplicate Discussions or the support portal.
- Keep the chooser short; contributors should not have to read five descriptions to file a bug.
- Test the forms on the default branch after merging; feature branches do not update the chooser.
- Keep a short `CONTRIBUTING.md` section that explains which template to use and where to ask questions.
