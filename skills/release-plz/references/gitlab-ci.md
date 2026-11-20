# release-plz GitLab CI

GitLab support is slightly more involved than GitHub because release-plz needs a token that can both push branches/tags and create merge requests. The official path is the **release-plz CI component**; a manual `.gitlab-ci.yml` job is the fallback.

Current component version: **0.1.0**. The component path is:

```yaml
include:
  - component: $CI_SERVER_FQDN/release-plz/components/release-plz@0.1.0
```

The component creates two jobs:

- `release-plz:pr` — runs `release-plz release-pr --forge gitlab --git-token "$RELEASE_PLZ_TOKEN"`.
- `release-plz:release` — runs `release-plz release --forge gitlab --git-token "$RELEASE_PLZ_TOKEN"`.

Both run on the default branch and skip tag pipelines.

## Contents

- [Token and bot email setup](#token-and-bot-email-setup)
- [Component inputs](#component-inputs)
- [Minimal component setup](#minimal-component-setup)
- [What the component does](#what-the-component-does)
- [Manual CLI fallback](#manual-cli-fallback)
- [Submodules hosted elsewhere](#submodules-hosted-elsewhere)
- [Registry tokens](#registry-tokens)
- [Caveats](#caveats)
- [Troubleshooting](#troubleshooting)

## Token and bot email setup

release-plz authenticates to GitLab with an access token. The token needs:

- **Role:** Maintainer or higher.
- **Scopes:** `api` (read/create/update merge requests and releases) and `write_repository` (push the release branch and tags).

Create a project or group access token:

1. **Settings → Access tokens → Add new token** (group or project).
2. Give it the Maintainer role and `api` + `write_repository` scopes.
3. Store it as a **masked** CI/CD variable named `RELEASE_PLZ_TOKEN`:
   **Settings → CI/CD → Variables → Add variable**.
4. Find the bot user created for the token under **Manage → Members**, and get its username.
5. Derive the bot email as `<username>@noreply.<your-gitlab-host>`. For gitlab.com that is `<username>@noreply.gitlab.com`.
6. Store it as a CI/CD variable named `RELEASE_PLZ_BOT_EMAIL`.

The bot email is required because release-plz creates commits, and Git needs an author identity. If it is missing or wrong, the release PR job fails when committing.

For a private cargo registry, add its token separately as `CARGO_REGISTRIES_<NAME>_TOKEN` (uppercase registry name). For crates.io, the variable is `CARGO_REGISTRY_TOKEN`.

## Component inputs

| Input | Default | Meaning |
| --- | --- | --- |
| `stage` | `deploy` | Stage for both release-plz jobs. |
| `image` | `rust` | Image with Cargo installed. |
| `cwd` | `.` | Directory to run release-plz in, relative to `$CI_PROJECT_DIR`. |
| `tags` | `[]` | Runner tags. |
| `git_credentials` | `[]` | Extra git credentials as `"https://user:pass@host"` strings; the default credential for `$CI_SERVER_HOST` can be overridden. |
| `git_submodule_recursive` | `false` | Recursively clone submodules and update them before running release-plz. |
| `bot_email` | `$RELEASE_PLZ_BOT_EMAIL` | Author email for release commits. |
| `bot_name` | `release-plz` | Author name for release commits. |
| `release_args` | empty | Extra args for `release-plz release`. |
| `pr_args` | empty | Extra args for `release-plz release-pr`. |

Example with inputs:

```yaml
include:
  - component: $CI_SERVER_FQDN/release-plz/components/release-plz@0.1.0
    inputs:
      stage: deploy
      image: rust:1.86
      cwd: rust-crates
      bot_name: "Release Bot"
      release_args: "--dry-run"
      pr_args: "-v --no-changelog"
```

`release_args` / `pr_args` are appended to the commands. Keep them to simple flags; do not pass values that require shell quoting unless you understand how the component interpolates them.

## Minimal component setup

```yaml
# .gitlab-ci.yml
include:
  - component: $CI_SERVER_FQDN/release-plz/components/release-plz@0.1.0
```

```text
CI/CD variables (masked):
  RELEASE_PLZ_TOKEN       = <project/group access token>
  RELEASE_PLZ_BOT_EMAIL   = <bot-username>@noreply.<gitlab-host>
  CARGO_REGISTRY_TOKEN    = <crates.io token>        # or CARGO_REGISTRIES_<NAME>_TOKEN
```

That is enough for the standard flow: the PR job opens/updates the release MR, and the release job publishes versions already present in `Cargo.toml`.

If your project uses shallow clones by default, set full history explicitly:

```yaml
variables:
  GIT_DEPTH: "0"
```

release-plz needs tags and commit history to compute versions and changelogs. A shallow clone can make the PR job miss releases or produce empty changelogs.

## What the component does

The component's template is worth understanding because most GitLab failures come from one of these steps:

- Uses `GIT_STRATEGY: clone` and skips tag pipelines.
- Runs only when `$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH`.
- Caches the crates.io index and download cache.
- Sets `git config --global credential.helper store` and writes the token into `~/.git-credentials`.
- Checks out `$CI_COMMIT_BRANCH` and verifies `HEAD == $CI_COMMIT_SHA`; if the branch moved, the job tells you to restart CI on the latest commit.
- Sets the git author to `bot_name` / `bot_email`.
- Rewrites the origin URL to include `RELEASE_PLZ_TOKEN`, because release-plz does not use `--git-token` for repository authentication when pushing the release branch.
- Installs `release-plz` and `cargo-semver-checks` with `cargo-binstall` (compiling from source is too slow for CI).
- Runs the two commands with `--forge gitlab`.

Do not duplicate the token by putting it in `GIT_STRATEGY` or the clone URL; the component handles the remote.

## Manual CLI fallback

Use this only when the component is unavailable (for example, an older GitLab instance without the CI catalog). The manual job must reproduce the component's token, identity, history, and install steps.

```yaml
stages:
  - deploy

.release-plz-base:
  stage: deploy
  image: rust:1.86
  rules:
    - if: $CI_COMMIT_TAG
      when: never
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  variables:
    GIT_STRATEGY: clone
    GIT_DEPTH: "0"
    CARGO_HOME: "$CI_PROJECT_DIR/.cargo"
  before_script:
    - git config --global user.email "$RELEASE_PLZ_BOT_EMAIL"
    - git config --global user.name "release-plz"
    - git remote set-url origin "$CI_SERVER_PROTOCOL://anything:$RELEASE_PLZ_TOKEN@$CI_SERVER_HOST/$CI_PROJECT_PATH.git"
    - curl -L --proto '=https' --tlsv1.2 -sSf https://raw.githubusercontent.com/cargo-bins/cargo-binstall/main/install-from-binstall-release.sh | bash
    - cargo binstall release-plz cargo-semver-checks --no-confirm

release-plz:pr:
  extends: .release-plz-base
  script:
    - release-plz release-pr --forge gitlab --git-token "$RELEASE_PLZ_TOKEN"

release-plz:release:
  extends: .release-plz-base
  script:
    - release-plz release --forge gitlab --git-token "$RELEASE_PLZ_TOKEN"
```

Notes:

- `GIT_DEPTH: "0"` avoids shallow-history failures.
- The remote URL uses `anything` as the username; GitLab accepts the token as the password. Do not echo the URL in logs.
- `CARGO_HOME` inside the project directory lets GitLab cache the cargo registry; add cache paths if you want faster runs.
- `cargo binstall` installs `cargo-semver-checks`; without it, release-plz's semver check is skipped or fails depending on configuration.
- The registry token is separate from `RELEASE_PLZ_TOKEN`. Set `CARGO_REGISTRY_TOKEN` (crates.io) or `CARGO_REGISTRIES_<NAME>_TOKEN` (private registry).

## Submodules hosted elsewhere

GitLab can clone submodules from GitLab repositories automatically, but a submodule hosted on GitHub (or another forge) needs credentials. Use the component inputs:

```yaml
include:
  - component: $CI_SERVER_FQDN/release-plz/components/release-plz@0.1.0
    inputs:
      git_submodule_recursive: true
      git_credentials:
        - "https://${GH_USERNAME}:${GH_TOKEN}@github.com"
```

- `git_submodule_recursive: true` makes the component clone recursively and run `git submodule update --init`.
- `git_credentials` entries are written to `~/.git-credentials` in the `.git-credentials` format.
- The default credential for `$CI_SERVER_HOST` is added after the custom ones so you can override it if needed.

For the manual job, add the same `git submodule update --init --recursive` and credential file setup before running release-plz.

## Registry tokens

- crates.io: `CARGO_REGISTRY_TOKEN`.
- Private registry: `CARGO_REGISTRIES_<NAME>_TOKEN`, where `<NAME>` is the registry name from the cargo config, uppercase. The component/CLI's `registry` must match.
- Do not confuse the registry token with `RELEASE_PLZ_TOKEN`; the former publishes crates, the latter pushes git branches and opens merge requests.
- If the project uses trusted publishing on crates.io, do not set `CARGO_REGISTRY_TOKEN`; but note that the GitLab component does not configure OIDC trusted publishing the way the GitHub Action does. Use a registry token for GitLab unless you have verified an OIDC setup.

## Caveats

- **`release_always = false` is GitHub-only.** The release command's release-PR association check is not supported on GitLab. Use `release_always = true` (the default) on GitLab.
- **The component runs only on the default branch and skips tag pipelines.** If you need a release from a tag pipeline, use a manual CLI job with explicit rules.
- **The component's default stage is `deploy`.** If your pipeline has no `deploy` stage, pass `stage:` with an existing stage.
- **The token must be masked; protected branches may require a protected token.** If the variable is protected, it is only available on protected branches/tags; make sure the default branch is protected in your project.
- **Merge request pipelines** are not the component's target. It reacts to branch pipelines on the default branch. Ensure branch pipelines are enabled for the project.
- **The release job may run on every default-branch commit.** With `release_always = true`, it checks the registry each time; versions already published are a no-op. Do not disable the job with a canceling concurrency rule; a canceled publish can be skipped.

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `release-plz:pr` cannot push | Token lacks `write_repository` or Maintainer role; check the remote URL and the token variable. |
| `release-plz:pr` cannot create an MR | Token lacks `api`; check project permissions. |
| Commit author error / empty email | `RELEASE_PLZ_BOT_EMAIL` missing or wrong; derive it from the bot username. |
| Shallow clone / missing tags | Set `GIT_DEPTH: "0"` or avoid shallow clones. |
| Pipeline runs on tags or non-default branches | The component's rules skip tags and run only on the default branch; check `CI_DEFAULT_BRANCH`. |
| Registry auth failure | Wrong variable name: `CARGO_REGISTRY_TOKEN` for crates.io, `CARGO_REGISTRIES_<NAME>_TOKEN` for private registries. |
| `cargo binstall` failure | The runner needs network access to GitHub and the component's image needs Cargo; pin `image:` or pre-install. |
| Release skipped after merging the release MR | `release_always = false` is GitHub-only; set it to `true` on GitLab. |
| Merge request pipeline does not trigger | Enable branch pipelines; the component targets default-branch branch pipelines. |
