# Agent Skills

A collection of [skills.sh](https://skills.sh)-compatible skills for AI coding
agents. Each skill lives in `skills/<name>/` and can be installed individually or
as a group.

## Install

### All skills

```bash
npx skills add jsdevlab/agents-skills
```

### A specific skill

```bash
npx skills add jsdevlab/agents-skills --skill changesets
npx skills add jsdevlab/agents-skills --skill editorconfig
npx skills add jsdevlab/agents-skills --skill github-dependabot
npx skills add jsdevlab/agents-skills --skill github-issue-templates
npx skills add jsdevlab/agents-skills --skill github-labels
npx skills add jsdevlab/agents-skills --skill github-pr-templates
npx skills add jsdevlab/agents-skills --skill license
npx skills add jsdevlab/agents-skills --skill release-plz
npx skills add jsdevlab/agents-skills --skill tauri-v2
```

The CLI discovers every skill under `skills/`, then lets you choose the skills and
target agents (Claude Code, Cursor, Codex, and others).

### From a local checkout

```bash
npx skills add ./skills                    # all skills
npx skills add ./skills --skill changesets # one skill
```

### Manual install

Copy a skill directory into one of:

- `~/.claude/skills/<name>/` — available in all projects
- `<your-project>/.claude/skills/<name>/` — available in that project

## Skills

| Skill | What it does | Requirements |
| --- | --- | --- |
| [`changesets`](skills/changesets/SKILL.md) | Create, version, and publish npm packages with Changesets: semver bumps, changelogs, `changeset version` / `changeset publish`, `changesets/action@v2` CI with npm trusted publishing, prereleases, and snapshots. | Node.js 22.11+, 24, or 26+; `@changesets/cli` v3; pnpm `>=10`, npm `>=10.9`, or yarn `>=4.5.2` |
| [`editorconfig`](skills/editorconfig/SKILL.md) | Create and maintain `.editorconfig` files that match a repository's languages and formatters: indentation, line endings, charset, trailing whitespace, final newlines, glob sections, monorepo overrides, Markdown/Makefile/Go exceptions, and alignment with Prettier, rustfmt, gofmt, and Black. Includes `editorconfig-checker` CI/pre-commit validation. | A repository whose editors or formatters need a shared baseline; optional `editorconfig-checker` v4 and Go/Docker for CI |
| [`github-dependabot`](skills/github-dependabot/SKILL.md) | Configure and operate GitHub Dependabot: `.github/dependabot.yml`, version and security updates, grouped and multi-ecosystem updates, schedules and cooldowns, monorepo directories, private registries, PR customization, and comment commands. | A GitHub repository with dependency manifests; repository settings access to enable alerts and security updates |
| [`github-issue-templates`](skills/github-issue-templates/SKILL.md) | Build GitHub issue forms and classic templates from project context: `bug_report.yml`, `feature_request.yml`, `config.yml`, validation, project-specific version/platform/log fields, and label references that match the repository. Pairs with `github-labels`. | A GitHub repository with Issues enabled; YAML parser for local validation |
| [`github-labels`](skills/github-labels/SKILL.md) | Design and sync GitHub labels as code: type/status/priority/area taxonomy, colors and descriptions, `gh label` commands, `.github/labels.yml`, and a pinned label-sync workflow with dry-run and safe deletion rules. Pairs with `github-issue-templates`. | GitHub repository write access; optional `gh` CLI; GitHub Actions for automatic sync |
| [`github-pr-templates`](skills/github-pr-templates/SKILL.md) | Design GitHub pull request templates from the repository's review workflow: a default `pull_request_template.md`, multiple templates under `.github/PULL_REQUEST_TEMPLATE/`, query-parameter links, project-specific test plans, issue links, breaking-change notes, and honest handling of labels and checkboxes. Pairs with `github-issue-templates`, `github-labels`, `changesets`, and `release-plz`. | A GitHub repository with pull requests enabled |
| [`license`](skills/license/SKILL.md) | Check and fix license headers and audit dependency licenses with Apache SkyWalking Eyes (`license-eye`): `.licenserc.yaml` configuration, `header check/fix/diff`, `dependency resolve/check`, LICENSE summaries, and GitHub Actions integration. | `license-eye` binary (release archive, `go install`, Homebrew, or Docker); Go 1.27+ if installing from source; Go, Maven, Node/npm/pnpm, Cargo, or Ruby toolchains for dependency checks |
| [`release-plz`](skills/release-plz/SKILL.md) | Automate Rust releases with release-plz: release PRs from conventional commits, version bumps, changelogs, `cargo publish`, git tags, and GitHub/GitLab/Gitea releases. Covers `release-plz.toml`, release strategies, and CI setup with the GitHub Action or GitLab CI component. | release-plz 0.4.3+; Rust toolchain; GitHub Actions or GitLab CI; a cargo registry token for publishing |
| [`tauri-v2`](skills/tauri-v2/SKILL.md) | Build Tauri 2 desktop and mobile apps: Rust commands and IPC, state, windows, `tauri.conf.json`, capabilities/permissions/CSP, official and custom plugins, installers, signing, app stores, auto-updates, GitHub Actions CI, and Learn recipes such as splashscreen, tray, menus, and sidecars. Includes split frontend/Tauri-shell layouts for teams that do not use the official template. | Rust 1.90+ for the updater plugin; Node.js LTS for JavaScript frontends; platform WebView/build toolchain (WebKitGTK 4.1, MSVC/WebView2, or Xcode); Android Studio/Xcode for mobile |

## Repository layout

```text
skills/
├── changesets/
│   ├── SKILL.md
│   └── references/
│       ├── cli.md
│       ├── config.md
│       ├── ci-automation.md
│       ├── prereleases-and-snapshots.md
│       └── troubleshooting.md
├── editorconfig/
│   ├── SKILL.md
│   ├── evals/
│   │   └── evals.json
│   └── references/
│       ├── syntax.md
│       ├── recipes.md
│       └── validation.md
├── github-dependabot/
│   ├── SKILL.md
│   ├── evals/
│   │   └── evals.json
│   └── references/
│       ├── configuration.md
│       ├── recipes.md
│       └── operations.md
├── github-issue-templates/
│   ├── SKILL.md
│   ├── evals/
│   │   └── evals.json
│   └── references/
│       ├── forms.md
│       ├── classic-and-config.md
│       ├── design.md
│       └── troubleshooting.md
├── github-labels/
│   ├── SKILL.md
│   ├── evals/
│   │   └── evals.json
│   └── references/
│       ├── taxonomy.md
│       ├── cli-and-api.md
│       ├── sync-action.md
│       └── template-integration.md
├── github-pr-templates/
│   ├── SKILL.md
│   ├── evals/
│   │   └── evals.json
│   └── references/
│       ├── design.md
│       ├── layout-and-selection.md
│       ├── examples.md
│       └── troubleshooting.md
├── license/
│   ├── SKILL.md
│   ├── evals/
│   │   └── evals.json
│   └── references/
│       ├── configuration.md
│       ├── cli.md
│       ├── dependency-licenses.md
│       ├── ci.md
│       └── troubleshooting.md
├── release-plz/
│   ├── SKILL.md
│   ├── evals/
│   │   └── evals.json
│   └── references/
│       ├── configuration.md
│       ├── cli.md
│       ├── github-actions.md
│       ├── gitlab-ci.md
│       ├── changelog-and-semver.md
│       └── troubleshooting.md
└── tauri-v2/
    ├── SKILL.md
    ├── evals/
    │   └── evals.json
    └── references/
        ├── development.md
        ├── configuration.md
        ├── security.md
        ├── plugin-usage.md
        ├── plugin-development.md
        ├── distribution.md
        ├── ci.md
        ├── tutorials.md
        └── troubleshooting.md
```

Packaged `.skill` artifacts and other build output under `dist/` are local only and
are git-ignored.

## Adding a new skill

1. Create `skills/<name>/SKILL.md` with the required frontmatter:

   ```markdown
   ---
   name: <name>
   description: What this skill does and when to use it
   ---
   ```

2. Put deeper documentation in `skills/<name>/references/` so `SKILL.md` stays focused.
3. Keep the skill name lowercase and hyphenated.
4. Verify discovery:

   ```bash
   npx skills add ./skills --list
   ```

## Publishing checklist

- [ ] Push to `jsdevlab/agents-skills` on GitHub
- [ ] Verify `npx skills add ./skills --list` lists every skill
- [x] Include the MIT `LICENSE`
- [ ] Optionally add the skills.sh badge:
      `[![skills.sh](https://skills.sh/b/jsdevlab/agents-skills)](https://skills.sh/jsdevlab/agents-skills)`

## License

[MIT](LICENSE) © 2026 jsdevlab
