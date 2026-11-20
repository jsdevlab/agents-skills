# Pull request template examples

Adapt these; do not copy every section. Delete anything that does not apply to the project.

## Contents

- [Rust library](#rust-library)
- [Node.js CLI](#nodejs-cli)
- [Web application](#web-application)
- [Monorepo](#monorepo)
- [Documentation](#documentation)
- [Infrastructure](#infrastructure)

## Rust library

```markdown
## Summary

<!-- What changed, and why? -->

## Related issue

Closes #

## Type of change

- [ ] Bug fix
- [ ] New API
- [ ] Breaking change
- [ ] Documentation
- [ ] Internal refactor

## Semver impact

<!-- Delete if none. -->

- [ ] Patch
- [ ] Minor
- [ ] Major
- [ ] No public API change

## Test plan

- `cargo test --all-features`
- `cargo clippy --all-targets --all-features -- -D warnings`
- Manual/example: <!-- command or example crate -->

## Breaking changes and migration

<!-- Delete if none. -->

## Checklist

- [ ] I added tests for the new behavior.
- [ ] I updated the rustdoc and README examples.
- [ ] I added a changelog entry if the project maintains one manually.
```

Notes:

- If the workspace uses release-plz, version bumps and changelogs come from conventional commits; ask for the commit type/scope rather than a manual version field.
- For a 0.x crate, call out breaking changes explicitly even when semver allows them.

## Node.js CLI

```markdown
## Summary

<!-- What changed, and why? -->

## Related issue

Closes #

## Type of change

- [ ] Bug fix
- [ ] New command or flag
- [ ] Output/format change
- [ ] Documentation
- [ ] Refactor

## Test plan

- `pnpm test`
- `pnpm build && node dist/cli.js <command>`
- Manual on: <!-- Linux / macOS / Windows, shell version -->

## Before/after output

<!-- Paste the command and the relevant stdout/stderr. Redact secrets. -->

## Breaking changes

<!-- Deleted/renamed command, flag, output, exit code, config key. Delete if none. -->

## Checklist

- [ ] I updated `--help` and the README usage.
- [ ] I added a test for the new command/flag.
- [ ] I checked exit codes and stderr behavior.
```

## Web application

```markdown
## Summary

<!-- What changed, and why? -->

## Related issue

Closes #

## Type of change

- [ ] Bug fix
- [ ] Feature
- [ ] Visual change
- [ ] Accessibility improvement
- [ ] Refactor

## Test plan

- `pnpm test`
- Browser/device: <!-- Chrome 130 / Safari 18 / Firefox / iOS Safari -->
- Manual flow: <!-- steps -->

## Screenshots or recording

| Before | After |
| --- | --- |
| <!-- image --> | <!-- image --> |

## Accessibility

<!-- Keyboard navigation, focus order, labels, contrast. Delete if not applicable. -->

## Deployment notes

<!-- Feature flag, migration, cache invalidation, rollout order. Delete if not applicable. -->

## Checklist

- [ ] I added or updated tests.
- [ ] I checked the change on a narrow viewport.
- [ ] I removed secrets and customer data from screenshots/logs.
```

## Monorepo

```markdown
## Summary

<!-- What changed, and why? -->

## Related issue

Closes #

## Affected packages

- [ ] `@acme/core`
- [ ] `@acme/cli`
- [ ] `@acme/web`
- [ ] `@acme/config`
- [ ] docs

## Type of change

- [ ] Bug fix
- [ ] Feature
- [ ] Breaking change
- [ ] Dependency update
- [ ] Documentation
- [ ] CI/build

## Test plan

- `pnpm turbo test --filter=...[HEAD^]`
- `pnpm turbo build --filter=...[HEAD^]`
- Manual: <!-- package-specific scenario -->

## Dependency graph impact

<!-- Does a package change require downstream releases? Changeset needed? Delete if none. -->

## Checklist

- [ ] I added a changeset if a published package changed.
- [ ] I ran tests for every affected package.
- [ ] I updated package READMEs or docs.
```

## Documentation

```markdown
## Summary

<!-- What pages or concepts changed? -->

## Related issue

Closes #

## Preview

<!-- Link to the preview deployment or local screenshot. -->

## Scope

- [ ] Typo or formatting
- [ ] New page
- [ ] Correctness/accuracy
- [ ] Restructure/navigation
- [ ] Translation

## Checklist

- [ ] Links resolve.
- [ ] Code samples run or are clearly illustrative.
- [ ] Screenshots match the current UI.
- [ ] I updated the table of contents or redirects if needed.
```

## Infrastructure

```markdown
## Summary

<!-- What infrastructure or deployment behavior changes? -->

## Related issue

Closes #

## Plan output

<!-- Paste the relevant terraform/opentofu plan or deployment diff. Redact secrets. -->

## Blast radius

<!-- Which environments, services, accounts, or regions are affected? -->

## Rollout and rollback

- Rollout:
- Rollback:
- Feature flag or maintenance window:

## Checklist

- [ ] The plan has no unexpected destroy/replace operations.
- [ ] Secrets are referenced from the secret manager, not committed.
- [ ] Dashboards/alerts cover the change.
- [ ] I tested in a non-production environment.
```

## Reusing sections

Common sections worth keeping consistent across templates:

- Summary
- Related issue
- Type of change
- Test plan
- Checklist

Variant sections:

- Semver/breaking change for libraries.
- Before/after output for CLIs.
- Screenshots/accessibility for UI.
- Affected packages/changeset for monorepos.
- Plan/rollback for infrastructure.

Do not make every variant a separate template if a single "Type of change" checklist plus conditional comments is enough.
