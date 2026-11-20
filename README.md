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
