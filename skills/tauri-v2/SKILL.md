---
name: tauri-v2
description: >-
  Use this skill for Tauri 2 application work across desktop and mobile: scaffolding projects (including split frontend/Tauri-shell layouts), implementing Rust commands and IPC (invoke, events, channels), file access and app-data paths, managing state and windows, configuring `tauri.conf.json`, capabilities/permissions/CSP, using or authoring plugins, packaging and signing installers, app-store distribution, auto-updates, and CI with GitHub Actions. Trigger it when the user mentions Tauri 2, `src-tauri`, `tauri.conf.json`, `#[tauri::command]`, `invoke`, capabilities or permission-denied errors, `tauri-plugin-*`, `tauri build`, bundle/signing, tauri-action, a separate Tauri shell and frontend project, or Tauri on Android/iOS. Chinese cues — Tauri 2 开发, 桌面应用, 移动端, Rust 后端, 权限配置, 打包, 签名, 自动更新, 插件开发, 发版, 壳项目, 前后端分离. Skip generic frontend work, non-Tauri Rust/Cargo projects, and general CI or Docker tasks unrelated to Tauri.
---

# Tauri 2

Tauri 2 builds a desktop or mobile app from a Rust core (`src-tauri`) plus a web frontend rendered by the operating system's WebView. The Rust process has full system access; the WebView is treated as untrusted and reaches Rust through commands. Plugin and core commands are gated by capabilities; app commands are callable by any window unless you opt into the app manifest ACL.

This skill covers development, security, learning recipes, packaging, distribution, CI, and plugin use/authoring. The official documentation is at <https://v2.tauri.app/>; the Chinese mirror is <https://v2.tauri.org.cn/>.

## Quick Start

```sh
# Scaffold (choose frontend, package manager, and template)
npm create tauri-app@latest

# From an existing web project
npm install --save-dev @tauri-apps/cli
npx tauri init

# Run and build
npm run tauri dev
npm run tauri build
```

A generated project looks like this:

```text
.
├── package.json
├── index.html
├── src/                    # frontend
└── src-tauri/
    ├── Cargo.toml
    ├── Cargo.lock          # commit this
    ├── build.rs            # tauri_build::build()
    ├── tauri.conf.json
    ├── capabilities/
    │   └── default.json
    ├── icons/
    └── src/
        ├── main.rs         # thin desktop entry point
        └── lib.rs          # all app logic and the mobile entry point
```

`main.rs` stays a thin passthrough; put logic in `lib.rs` so Android and iOS builds work:

```rust
// src-tauri/src/main.rs
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

fn main() {
    app_lib::run();
}
```

```rust
// src-tauri/src/lib.rs
#[tauri::command]
fn greet(name: String) -> String {
    format!("Hello, {name}!")
}

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![greet])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

```ts
import { invoke } from '@tauri-apps/api/core';

const message = await invoke<string>('greet', { name: 'World' });
```

### Layout choices

The generated layout is only the default. Teams often split the frontend from the Tauri shell:

| Layout | Shape | How to run |
| --- | --- | --- |
| Standard | frontend + `src-tauri` in one repo | `npm run tauri dev` / `build`; hooks own the frontend. |
| Split directories | `web/` (frontend) + `shell/src-tauri` | Run the web dev server separately; run the Tauri CLI from `shell/`. |
| Split repositories | `app-web` repo + `app-shell` repo | Build the web repo, place its output where `frontendDist` points, then build the shell. |
| Cargo monorepo | workspace + `apps/desktop/src-tauri` | Run the Tauri CLI from `apps/desktop`; CI uses `projectPath`. |

In a split layout the shell does not own the frontend build. Omit `beforeDevCommand` and `beforeBuildCommand`, point `devUrl` at the frontend dev server, and point `frontendDist` at the built assets:

```json
{
  "build": {
    "devUrl": "http://localhost:5173",
    "frontendDist": "../../web/dist"
  }
}
```

The Tauri CLI has no `--project-dir`; run it from the directory that contains `src-tauri`. In CI, `tauri-action` takes `projectPath: shell` when the shell is not at the repository root. See `development.md` for the full split workflow and `ci.md` for checking out two repositories.

Every command must be listed in `generate_handler![]`. Forgetting it produces a runtime "command not found" rejection, not a compile error.

Tauri 2 denies plugin and window APIs by default. Grant them in `src-tauri/capabilities/default.json`:

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "Capability for the main window",
  "windows": ["main"],
  "permissions": ["core:default"]
}
```

## Core Model

| Concern | Mechanism | Notes |
| --- | --- | --- |
| Frontend → Rust | `invoke('command', args)` | JSON over IPC; commands are `#[tauri::command]` functions. |
| Rust → frontend | `emit` / `emit_to`, or `Channel` | Events are small, JSON, many-to-many; channels are ordered and fast for streams. |
| Shared state | `app.manage(state)` + `State<T>` | Wrap mutable state in `Mutex`; wrong type fails at runtime. |
| Windows | `WebviewWindowBuilder`, `get_webview_window("label")` | Tauri 2 unified window + webview API. |
| Permissions | capabilities + permissions + scopes | Gated by the runtime authority for plugin/core commands; app commands need `AppManifest::commands` to be ACL-gated. |
| Mobile entry | `#[cfg_attr(mobile, tauri::mobile_entry_point)]` on `run()` | Requires `[lib] crate-type = ["staticlib", "cdylib", "rlib"]`. |

## Critical Rules

**Always**

- Register every command in `tauri::generate_handler![]`.
- Return `Result<T, E>` from fallible commands; serialize the error type.
- Use owned types (`String`, `Vec<T>`) in async command parameters.
- Return `Result` when an async command takes a borrowed handle such as `State<'_, T>`.
- Scope capabilities to the windows and platforms that need them.
- Add plugin permissions to a capability, not just the Cargo/npm dependency.
- Keep `tauri`, `tauri-build`, and `@tauri-apps/cli` on compatible 2.x versions.
- Commit `src-tauri/Cargo.lock`; do not commit `src-tauri/target`.
- Build Linux bundles on the oldest baseline you support (Ubuntu 22.04 or Debian 12).

**Never**

- Use `@tauri-apps/api/tauri` (v1) — use `@tauri-apps/api/core`.
- Use `app.get_window()` (v1) — use `app.get_webview_window()`.
- Assume a plugin works on mobile without checking its platform support.
- Put secrets or privileged logic in the frontend; the WebView is the untrusted side.
- Use `std::thread::sleep` inside async tasks; it blocks the runtime.
- Rely on `allowlist` from Tauri 1 — it was replaced by capabilities.

## `tauri.conf.json` Essentials

```json
{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "My App",
  "version": "1.0.0",
  "identifier": "com.example.myapp",
  "build": {
    "devUrl": "http://localhost:5173",
    "frontendDist": "../dist",
    "beforeDevCommand": "npm run dev",
    "beforeBuildCommand": "npm run build"
  },
  "app": {
    "windows": [
      {
        "label": "main",
        "title": "My App",
        "width": 1000,
        "height": 700
      }
    ],
    "security": {
      "csp": "default-src 'self'; connect-src ipc: http://ipc.localhost; img-src 'self' asset: http://asset.localhost data:",
      "capabilities": ["default"]
    }
  },
  "bundle": {
    "active": true,
    "targets": "all",
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/128x128@2x.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ],
    "createUpdaterArtifacts": true
  }
}
```

Platform-specific files (`tauri.windows.conf.json`, `tauri.macos.conf.json`, `tauri.linux.conf.json`, `tauri.android.conf.json`, `tauri.ios.conf.json`) are merged into the base config. Objects merge key by key, but **arrays are replaced**, not concatenated.

## Reference Map

| Task | Read |
| --- | --- |
| Commands, events, channels, state, windows, resources, sidecars, split frontend/shell layouts, mobile dev, testing | [`references/development.md`](references/development.md) |
| `tauri.conf.json`, platform overrides, bundle and security keys | [`references/configuration.md`](references/configuration.md) |
| Capabilities, permissions, scopes, CSP, asset protocol, threat model | [`references/security.md`](references/security.md) |
| Installing and configuring official plugins | [`references/plugin-usage.md`](references/plugin-usage.md) |
| Authoring a plugin, including Android/iOS native code | [`references/plugin-development.md`](references/plugin-development.md) |
| Installers, signing, app stores, updater | [`references/distribution.md`](references/distribution.md) |
| GitHub Actions, split-repo CI, signing secrets, WebDriver tests, other CI | [`references/ci.md`](references/ci.md) |
| Splashscreen, tray, custom titlebar, menus, sidecar recipes | [`references/tutorials.md`](references/tutorials.md) |
| Build/runtime errors and v1→v2 migration | [`references/troubleshooting.md`](references/troubleshooting.md) |

## Version Notes

Versions verified on 2026-11-20:

| Component | Version |
| --- | --- |
| `tauri` crate | 2.12.7 |
| `tauri-build` crate | 2.7.6 |
| `@tauri-apps/cli` | 2.12.6 |
| `@tauri-apps/api` | 2.12.5 |
| `tauri-apps/tauri-action` | v1.0.0 |

Use `^2` or `2` in manifests rather than pinning a patch, unless you have a reproducibility reason. `cargo tauri add <plugin>` picks a compatible plugin version and wires up the Rust dependency, builder registration, npm package, and default permission.

## Common Failures

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `Command greet not found` | Missing from `generate_handler![]` | Add it to the handler macro. |
| `fs.write_text_file not allowed` | Missing capability permission or scope | Add the permission and a scoped object to `capabilities/*.json`. |
| `state not managed for field ...` | `State<T>` type does not match `manage()` exactly | Use `State<'_, Mutex<AppState>>` if you managed `Mutex<AppState>`. |
| White/blank window | `devUrl` or `frontendDist` wrong, or frontend build failed | Check `beforeDevCommand` / `beforeBuildCommand`; open DevTools. |
| Split layout: white screen after build | Frontend was not built, or `frontendDist` points at the wrong repo/path | Build the web project first; resolve `frontendDist` from `src-tauri`; run the Tauri CLI from the shell. |
| Mobile build fails | Missing Rust targets or Android/iOS SDK env | `rustup target add ...`, set `ANDROID_HOME`/`NDK_HOME` or use Xcode. |
| Plugin command denied | Dependency installed but no permission | Add `<plugin>:default` or a specific permission to a capability. |
| Updater refuses to install | Artifacts not signed or endpoint not HTTPS | Set `createUpdaterArtifacts`, sign with `TAURI_SIGNING_PRIVATE_KEY`, use HTTPS. |
| macOS says app is damaged | Unsigned or not notarized Apple Silicon build | Sign and notarize, or use ad-hoc signing for local builds. |

## Official Links

- Start and prerequisites: <https://v2.tauri.app/start/>
- Develop: <https://v2.tauri.app/develop/>
- Security: <https://v2.tauri.app/security/>
- Plugins: <https://v2.tauri.app/plugin/>
- Distribute: <https://v2.tauri.app/distribute/>
- Learn recipes: <https://v2.tauri.app/learn/>
- Configuration reference: <https://v2.tauri.app/reference/config/>
- CLI reference: <https://v2.tauri.app/reference/cli/>
- Chinese docs: <https://v2.tauri.org.cn/learn/>
