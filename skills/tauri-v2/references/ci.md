# CI for Tauri 2

Tauri's official CI guide is GitHub Actions-centric. The same steps work on any runner that can install WebKitGTK (Linux), Xcode (macOS), or the MSVC/WebView2 toolchain (Windows).

Official docs: <https://v2.tauri.app/distribute/pipelines/github/> and <https://v2.tauri.app/develop/tests/webdriver/>.

## Contents

- [GitHub Actions release workflow](#github-actions-release-workflow)
- [Signing secrets](#signing-secrets)
- [Test workflow](#test-workflow)
- [WebDriver end-to-end tests](#webdriver-end-to-end-tests)
- [Android and iOS CI](#android-and-ios-ci)
- [Split repositories and monorepos](#split-repositories-and-monorepos)
- [GitLab CI and other runners](#gitlab-ci-and-other-runners)
- [Caching and performance](#caching-and-performance)
- [Troubleshooting](#troubleshooting)

## GitHub Actions release workflow

```yaml
name: publish

on:
  workflow_dispatch:
  push:
    tags:
      - 'app-v*'

jobs:
  publish-tauri:
    permissions:
      contents: write
    strategy:
      fail-fast: false
      matrix:
        include:
          - platform: macos-latest
            args: '--target aarch64-apple-darwin'
          - platform: macos-latest
            args: '--target x86_64-apple-darwin'
          - platform: ubuntu-22.04
            args: ''
          - platform: windows-latest
            args: ''

    runs-on: ${{ matrix.platform }}
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1

      - name: Install Linux dependencies
        if: matrix.platform == 'ubuntu-22.04'
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            libwebkit2gtk-4.1-dev \
            libayatana-appindicator3-dev \
            librsvg2-dev \
            patchelf

      - uses: actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1 # v7.1.0
        with:
          node-version: lts/*
          cache: npm

      - uses: dtolnay/rust-toolchain@39008827e6b4319a349e9a42dfc90579dd25f1ca # stable branch
        with:
          targets: ${{ matrix.platform == 'macos-latest' && 'aarch64-apple-darwin,x86_64-apple-darwin' || '' }}

      - uses: Swatinem/rust-cache@d1527c8f67ef9af357290475d4af4b0fab8f12e0 # v2.9.3
        with:
          workspaces: './src-tauri -> target'

      - run: npm ci

      - uses: tauri-apps/tauri-action@1deb371b0cd8bd54025b384f1cd735e725c4060f # v1.0.0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          TAURI_SIGNING_PRIVATE_KEY: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY }}
          TAURI_SIGNING_PRIVATE_KEY_PASSWORD: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY_PASSWORD }}
        with:
          tagName: app-v__VERSION__
          releaseName: 'App v__VERSION__'
          releaseBody: 'See the assets below to install this version.'
          releaseDraft: true
          prerelease: false
          args: ${{ matrix.args }}
```

Notes:

- `tauri-action` runs `tauri build`, uploads bundles to a GitHub release, and substitutes `__VERSION__` with the app version.
- Use `releaseDraft: true` for the first runs and publish manually after checking artifacts.
- `fail-fast: false` lets other platforms finish if one fails.
- `permissions: contents: write` is required to create releases. Without it you get `Resource not accessible by integration`.
- Pin third-party actions to a full commit SHA in production. The SHA above is `tauri-action` v1.0.0; replace it with the version you audit.
- `dtolnay/rust-toolchain` is pinned to the `stable` branch commit; its default `toolchain` input remains `stable`, so Rust itself still tracks the latest stable release.
- On private repositories, `ubuntu-22.04-arm` and `ubuntu-24.04-arm` hosted runners may not be available; use QEMU or build ARM on matching hardware.
- `args` is passed through to `tauri build`; add `--bundles nsis` or `--target ...` as needed.
- For multiple package managers, change `cache` and the install command (`pnpm install --frozen-lockfile`, `yarn install --immutable`).
- `npm ci` and `cache: npm` require a committed lockfile. If the repository has none yet, run `npm install` once, commit `package-lock.json`, then switch to `npm ci`.

### Trigger choices

- Tag push (`app-v*`): the tag triggers the workflow; `tauri-action` reads the app version from `tauri.conf.json` to replace `__VERSION__`.
- Push to `release` branch: the action creates the tag and release from the app version.
- `workflow_dispatch`: manual release with the current version.

Do not trigger releases on every `main` push; use tags or a release branch.

### Action inputs worth knowing

The documented core inputs are `tagName`, `releaseName`, `releaseBody`, `releaseDraft`, `prerelease`, `args`, and `projectPath`. Useful extras:

- `uploadUpdaterJson: true` uploads a static updater manifest (`latest.json`) to the release; combine it with signed updater artifacts. `updaterJsonPreferNsis` chooses the NSIS artifact on Windows.
- `mobile: android` or `mobile: ios` switches the action to a mobile build.
- `tauriScript` selects the package-manager command when the action cannot infer it.
- `retryAttempts` retries flaky release uploads.

The action README is the source of truth for the full list: <https://github.com/tauri-apps/tauri-action>.

The examples pin full commit SHAs for `actions/checkout` v7.0.1,
`actions/setup-node` v7.1.0, and `tauri-apps/tauri-action` v1.0.0; keep the
`# vX.Y.Z` comments informational. `actions/setup-node` v7 runs on Node 24;
Node 20 and action majors declaring `runs.using: node20` are not supported on
GitHub-hosted runners after 2026-09-23. For pnpm, use `pnpm/setup` v3 instead
of `pnpm/action-setup`; it installs pnpm 11+ and Node in one step, for example:

```yaml
- uses: pnpm/setup@731dfb6463bbd11c11fd0ee11fe00ed0e36d48a1 # v3.2.0
  with:
    version: 12
    runtime: node@24
    cache: true
    require-lockfile: true
```

It runs `pnpm install` by default, so drop the separate `actions/setup-node`
and `pnpm install` steps, or set `install: false` if a later step handles it.

### Concurrency

```yaml
concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false
```

Never cancel a running release build: it may be uploading assets or signing artifacts. Use `cancel-in-progress: false` for release workflows; a separate PR/test workflow can cancel freely.

## Signing secrets

Set these as repository/environment secrets.

| Secret | Used by | Purpose |
| --- | --- | --- |
| `TAURI_SIGNING_PRIVATE_KEY` | Updater | Private key content or path. |
| `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` | Updater | Key password, if any. |
| `APPLE_CERTIFICATE` | macOS/iOS | Base64 `.p12` signing certificate. |
| `APPLE_CERTIFICATE_PASSWORD` | macOS/iOS | `.p12` password. |
| `APPLE_SIGNING_IDENTITY` | macOS | Keychain identity string. |
| `APPLE_API_ISSUER`, `APPLE_API_KEY`, `APPLE_API_KEY_PATH` | macOS/iOS | App Store Connect API key for notarization/upload. |
| `APPLE_ID`, `APPLE_PASSWORD`, `APPLE_TEAM_ID` | macOS | Apple ID notarization alternative. |
| `WINDOWS_CERTIFICATE` | Windows | Base64 `.pfx`. |
| `WINDOWS_CERTIFICATE_PASSWORD` | Windows | `.pfx` password. |
| `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID` | Windows | Azure Key Vault/Artifact Signing. |
| `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD`, `ANDROID_KEY_BASE64` | Android | Upload keystore. |
| `IOS_CERTIFICATE`, `IOS_CERTIFICATE_PASSWORD`, `IOS_MOBILE_PROVISION` | iOS | Manual signing. |

Tauri's bundler reads the updater variables directly. Windows certificate import and Android `keystore.properties` usually need an explicit step:

```yaml
- name: Import Windows certificate
  if: matrix.platform == 'windows-latest'
  shell: pwsh
  env:
    WINDOWS_CERTIFICATE: ${{ secrets.WINDOWS_CERTIFICATE }}
    WINDOWS_CERTIFICATE_PASSWORD: ${{ secrets.WINDOWS_CERTIFICATE_PASSWORD }}
  run: |
    $pfx = [Convert]::FromBase64String($env:WINDOWS_CERTIFICATE)
    $path = Join-Path $env:RUNNER_TEMP 'certificate.pfx'
    [IO.File]::WriteAllBytes($path, $pfx)
    Import-PfxCertificate -FilePath $path `
      -CertStoreLocation Cert:\CurrentUser\My `
      -Password (ConvertTo-SecureString $env:WINDOWS_CERTIFICATE_PASSWORD -AsPlainText -Force)
```

macOS certificate import is often handled by `tauri-action`; if not, create a temporary keychain, import the base64 `.p12`, and set `APPLE_SIGNING_IDENTITY`. Do not echo secrets.

## Test workflow

```yaml
name: test

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1

      - name: Install Linux dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            libwebkit2gtk-4.1-dev \
            libayatana-appindicator3-dev \
            librsvg2-dev \
            patchelf

      - uses: dtolnay/rust-toolchain@39008827e6b4319a349e9a42dfc90579dd25f1ca # stable branch
        with:
          components: rustfmt, clippy

      - uses: Swatinem/rust-cache@d1527c8f67ef9af357290475d4af4b0fab8f12e0 # v2.9.3
        with:
          workspaces: './src-tauri -> target'

      - uses: actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1 # v7.1.0
        with:
          node-version: lts/*
          cache: npm

      - run: npm ci
      - run: npm run build
      - run: cargo fmt --manifest-path src-tauri/Cargo.toml -- --check
      - run: cargo clippy --manifest-path src-tauri/Cargo.toml --all-targets -- -D warnings
      - run: cargo test --manifest-path src-tauri/Cargo.toml
      - run: npm test
```

Run frontend tests and Rust tests separately; do not require a WebView for unit tests. `tauri::test::mock_builder` and `mock_context` cover commands and state without a window.

## WebDriver end-to-end tests

The recommended stack is WebdriverIO with `@wdio/tauri-service`. It embeds a WebDriver server and supports Windows, Linux, and macOS:

```sh
npm create wdio@latest ./
# choose Desktop Testing → Tauri
```

On Linux, install `webkit2gtk-driver` and `xvfb`, then run under a virtual display:

```yaml
- name: Install WebDriver dependencies
  run: |
    sudo apt-get update
    sudo apt-get install -y libwebkit2gtk-4.1-dev webkit2gtk-driver xvfb

- name: Install tauri-driver
  run: cargo install tauri-driver --locked

- name: Run WebDriver tests
  run: xvfb-run npm test
  working-directory: e2e-tests
```

On Windows, install a matching Edge WebDriver (`msedgedriver-tool`) and run the tests directly. macOS is supported by the embedded WebDriver server; `tauri-driver` alone does not support macOS desktop.

Do not put WebDriver tests in the same job as the release build; they need a built binary and a display.

## Android and iOS CI

Android:

- Runner: `ubuntu-latest` with Java 17+ and the Android SDK/NDK.
- Install Rust targets: `aarch64-linux-android`, `armv7-linux-androideabi`, `i686-linux-android`, `x86_64-linux-android`.
- Set `ANDROID_HOME`, `NDK_HOME`, `JAVA_HOME`.
- Write `src-tauri/gen/android/keystore.properties` from secrets.
- Build: `npm run tauri android build -- --aab`.
- Upload the AAB as an artifact; the first Play upload is manual.

iOS:

- Runner: `macos-latest`.
- Install the distribution certificate/provisioning profile or use App Store Connect API keys.
- Build: `npm run tauri ios build -- --export-method app-store-connect`.
- Upload with `xcrun altool` or Transporter.
- iOS builds cannot run on Linux or Windows.

## Split repositories and monorepos

When the frontend is a separate repository or directory, the build order is always **frontend first, shell second**.

Two-repository workflow:

```yaml
jobs:
  publish:
    runs-on: ubuntu-22.04
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          repository: owner/app-web
          path: app-web
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          repository: owner/app-shell
          path: app-shell
      - uses: actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1 # v7.1.0
        with:
          node-version: lts/*
      - run: npm ci && npm run build
        working-directory: app-web
      - run: npm ci
        working-directory: app-shell
      - uses: tauri-apps/tauri-action@1deb371b0cd8bd54025b384f1cd735e725c4060f # v1.0.0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          projectPath: app-shell
          tagName: app-v__VERSION__
          releaseDraft: true
```

- `projectPath` is the directory containing `src-tauri`, not the repository root.
- `frontendDist` in `app-shell/src-tauri/tauri.conf.json` must resolve to the built web assets, for example `../../app-web/dist` when both checkouts are siblings.
- If the shell repo and frontend repo are private, `actions/checkout` needs a token with access to both (`token: ${{ secrets.CROSS_REPO_TOKEN }}`).
- Pin the web repo checkout to a full commit SHA for reproducible releases; `ref:` defaults to the default branch.
- An alternative is to build the web app in one job, upload `dist/` as an artifact, and download it into the shell job. This avoids a cross-repo checkout at release time and keeps the frontend's own CI authoritative.

Monorepo layout:

```text
repo/
├── apps/desktop/src-tauri/
└── apps/web/
```

```yaml
- run: npm ci && npm run build
  working-directory: apps/web
- run: npm ci
  working-directory: apps/desktop
- uses: tauri-apps/tauri-action@1deb371b0cd8bd54025b384f1cd735e725c4060f # v1.0.0
  with:
    projectPath: apps/desktop
```

`tauri build` itself must run from `apps/desktop`, not the workspace root. For non-GitHub runners, run the frontend build step first, then `tauri build` in the shell directory, and publish the bundles from `apps/desktop/src-tauri/target/release/bundle/`.

## GitLab CI and other runners

Tauri has no official GitLab component, but the build is the same CLI. A minimal Linux job:

```yaml
stages: [build]

tauri-linux:
  stage: build
  image: ubuntu:22.04
  rules:
    - if: $CI_COMMIT_TAG
  before_script:
    - apt-get update
    - apt-get install -y curl wget file build-essential libwebkit2gtk-4.1-dev
      libayatana-appindicator3-dev librsvg2-dev patchelf
    - curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
    - apt-get install -y nodejs
    - curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh -s -- -y
    - . "$HOME/.cargo/env"
  script:
    - npm ci
    - npm run tauri build
  artifacts:
    paths:
      - src-tauri/target/release/bundle/
```

Requirements for any runner:

- Linux: WebKitGTK 4.1, GTK, appindicator, librsvg, patchelf; a display is not needed for `tauri build`, but is needed for WebDriver tests.
- macOS: Xcode and signing credentials.
- Windows: MSVC build tools, WebView2, and optionally NSIS/WiX.
- Cache `src-tauri/target` and the package manager cache; a cold Tauri build is expensive.

## Caching and performance

- `swatinem/rust-cache` with `workspaces: './src-tauri -> target'` matches the default layout.
- Cache npm/pnpm/yarn stores via `actions/setup-node` `cache`.
- Build macOS Intel and Apple Silicon in separate matrix entries; `--target universal-apple-darwin` needs both Rust targets and doubles link time.
- Avoid `--debug` in release workflows.
- Use `npm ci` (or frozen lockfile equivalents) for reproducible installs.
- Keep Linux builds on Ubuntu 22.04 unless you intentionally raise the glibc baseline.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Resource not accessible by integration` | `GITHUB_TOKEN` lacks write access | Add `permissions: contents: write`; check repository Actions settings. |
| Release job has no assets | Build failed earlier in the matrix | Check the failing platform; `fail-fast: false` keeps others running. |
| Linux binary fails on older distro | Built on a newer glibc | Build on Ubuntu 22.04 / Debian 12. |
| macOS app is unsigned or "damaged" | Certificate/notarization secrets missing | Import certificate, set `APPLE_SIGNING_IDENTITY`, notarize. |
| Windows installer unsigned | Certificate not imported or thumbprint mismatch | Import `.pfx`; verify `certificateThumbprint`. |
| Updater artifacts missing | `createUpdaterArtifacts` false or signing key absent | Set it to `true`; set `TAURI_SIGNING_PRIVATE_KEY`. |
| `tauri-action` uses an unexpected CLI | Action resolves the project's CLI and falls back to its own | Use the `tauriScript` input to point at your installed CLI, or pin the action version; align the CLI with the `tauri` crate. |
| WebDriver tests fail on Linux | No display | Run under `xvfb-run`. |
| Cache never hits | Lockfile missing or wrong `workspaces` path | Commit the lockfile; fix the cache path. |
