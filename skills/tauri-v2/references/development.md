# Tauri 2 development

Covers project setup, commands and IPC, state, windows, resources, sidecars, mobile development, debugging, and testing. Official docs: <https://v2.tauri.app/develop/>.

## Contents

- [Project setup](#project-setup)
- [Split frontend and shell layouts](#split-frontend-and-shell-layouts)
- [Commands](#commands)
- [Events and channels](#events-and-channels)
- [State](#state)
- [Windows and menus](#windows-and-menus)
- [Resources](#resources)
- [Sidecars](#sidecars)
- [Mobile development](#mobile-development)
- [Debugging](#debugging)
- [Testing](#testing)
- [Pitfalls](#pitfalls)

## Project setup

Prerequisites:

- **Rust** via `rustup`; on Windows use the MSVC toolchain (`rustup default stable-msvc`).
- **Node.js LTS** only if the frontend uses a JavaScript toolchain.
- **Linux**: WebKitGTK 4.1 and GTK development packages. Debian/Ubuntu:
  ```sh
  sudo apt install libwebkit2gtk-4.1-dev build-essential curl wget file \
    libxdo-dev libssl-dev libayatana-appindicator3-dev librsvg2-dev
  ```
- **Windows**: Microsoft C++ Build Tools with "Desktop development with C++", plus the WebView2 runtime.
- **macOS**: Xcode.
- **Mobile**: Android Studio + SDK/NDK and `JAVA_HOME`/`ANDROID_HOME`/`NDK_HOME`; iOS needs Xcode, CocoaPods, and Xcode command line tools.

The Rust crate is in `src-tauri`. `tauri.conf.json` marks the project root for the CLI. `src/lib.rs` holds the shared entry point and all logic; `src/main.rs` only calls `app_lib::run()`.

`src-tauri/Cargo.toml` for a mobile-capable app:

```toml
[package]
name = "app"
version = "0.1.0"
edition = "2021"

[lib]
name = "app_lib"
crate-type = ["staticlib", "cdylib", "rlib"]

[build-dependencies]
tauri-build = { version = "2", features = [] }

[dependencies]
tauri = { version = "2", features = [] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

## Split frontend and shell layouts

Not every project uses `create-tauri-app`. A common production layout separates the web frontend from the Rust shell. The shell still contains `src-tauri`, because the Tauri CLI has no `--project-dir` and must run from the directory that contains it.

### Same repository, separate directories

```text
repo/
├── web/                     # React/Vue/Svelte/etc.
│   ├── package.json
│   └── dist/                # built assets
└── shell/
    ├── package.json         # optional; @tauri-apps/cli
    └── src-tauri/
        ├── tauri.conf.json
        ├── Cargo.toml
        └── src/
```

```json
{
  "build": {
    "devUrl": "http://localhost:5173",
    "frontendDist": "../../web/dist"
  }
}
```

- No `beforeDevCommand` / `beforeBuildCommand`: the web project owns its dev server and build.
- Start the web dev server in one terminal: `npm --prefix web run dev`.
- Start the shell from `shell/`: `npm run tauri dev` (or `cargo tauri dev`).
- Build: `npm --prefix web run build`, then `npm run tauri build` from `shell/`.
- `frontendDist` is resolved relative to `src-tauri`, so `../../web/dist` means `repo/web/dist`.

If you want the shell to orchestrate the frontend, keep it explicit:

```json
{
  "build": {
    "devUrl": "http://localhost:5173",
    "frontendDist": "../../web/dist",
    "beforeDevCommand": "npm --prefix ../web run dev",
    "beforeBuildCommand": "npm --prefix ../web run build"
  }
}
```

The object form is clearer when the frontend lives elsewhere:

```json
{
  "build": {
    "devUrl": "http://localhost:5173",
    "frontendDist": "../../web/dist",
    "beforeDevCommand": { "script": "npm run dev", "cwd": "../web", "wait": false },
    "beforeBuildCommand": { "script": "npm run build", "cwd": "../web" }
  }
}
```

Hooks run with the **frontend directory** as cwd: the nearest directory containing `package.json`, or the parent of `src-tauri` when none is found. In this layout that is `shell/`, so `cwd: "../web"` resolves to `repo/web`. Use the object form, `--prefix`, or an absolute path rather than assuming `src-tauri` or the repository root.

### Two repositories

```text
app-web/          # frontend repo, built in its own CI job
app-shell/        # Tauri shell repo; src-tauri lives here
```

Options for getting the frontend into the shell build:

1. **Path outside the shell checkout** — check out both repos side by side and set `frontendDist` to `../../app-web/dist`. Works locally and in CI, but the path must exist at build time.
2. **Copy into the shell** — build the web repo, copy `dist/` into `app-shell/dist/`, and set `frontendDist: "../dist"`. This makes the shell build self-contained and is easier to cache/artifact.
3. **Remote URL** — `frontendDist` may be a URL, but then the packaged app loads remote content. Avoid this for offline desktop apps; it changes the CSP, remote-access, and update story.

For CI, check out both repositories and use `tauri-action`'s `projectPath`:

```yaml
- uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
  with:
    repository: owner/app-web
    path: app-web
- uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
  with:
    repository: owner/app-shell
    path: app-shell
- run: npm ci && npm run build
  working-directory: app-web
- run: npm ci
  working-directory: app-shell
- uses: tauri-apps/tauri-action@1deb371b0cd8bd54025b384f1cd735e725c4060f # v1.0.0
  with:
    projectPath: app-shell
```

### Cargo workspace / monorepo

```text
repo/
├── Cargo.toml               # [workspace]
├── apps/desktop/
│   └── src-tauri/
└── web/
```

Run the Tauri CLI from `apps/desktop` (the directory containing `src-tauri`), not the workspace root. In CI, set `projectPath: apps/desktop`.

### Keeping the two sides compatible

- The frontend installs `@tauri-apps/api` and plugin JS packages; the shell owns the `tauri` crate, CLI, capabilities, and plugins. Keep major/minor versions compatible and upgrade both sides together.
- Command names, argument casing, and event names are the contract. Generate TypeScript bindings with `tauri-specta`/`specta`, or keep a shared types package; do not hand-maintain duplicate definitions.
- Capabilities and permissions live in the shell. The frontend cannot grant itself access; a missing permission is a shell-side fix.
- On mobile, the web dev server must be reachable from the device. Use `TAURI_DEV_HOST` in the frontend dev-server config, bind to `0.0.0.0`, and keep the HMR host/port aligned. Tauri's mobile dev command sets `TAURI_DEV_HOST`.
- A split layout does not change packaging: the final `frontendDist` is bundled into the app. Test the packaged app, not only the dev server.

## Commands

```rust
#[tauri::command]
fn my_custom_command(invoke_message: String) -> String {
    format!("Rust received: {invoke_message}")
}

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![my_custom_command])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

```ts
import { invoke } from '@tauri-apps/api/core';

const result = await invoke<string>('my_custom_command', {
  invokeMessage: 'Hello',
});
```

Rules:

- Command names must be unique across the whole app, including separate modules.
- Commands defined directly in `lib.rs` must not be `pub`; commands in a module must be `pub`.
- Arguments are camelCase in JavaScript by default. Use `#[tauri::command(rename_all = "snake_case")]` to change that.
- Arguments must implement `serde::Deserialize`; return values and errors must implement `serde::Serialize`.
- For binary data, return `tauri::ipc::Response` instead of a `Vec<u8>` to avoid JSON overhead:

```rust
use tauri::ipc::Response;

#[tauri::command]
fn read_file() -> Response {
    Response::new(std::fs::read("path/to/file").unwrap())
}
```

### Errors

```rust
#[derive(Debug, thiserror::Error)]
enum Error {
    #[error(transparent)]
    Io(#[from] std::io::Error),
    #[error("not found: {0}")]
    NotFound(String),
}

impl serde::Serialize for Error {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::ser::Serializer,
    {
        serializer.serialize_str(&self.to_string())
    }
}

#[tauri::command]
fn read(path: String) -> Result<String, Error> {
    std::fs::read_to_string(&path).map_err(Into::into)
}
```

A rejected promise on the frontend means the command returned `Err`.

### Async commands

```rust
#[tauri::command]
async fn fetch(url: String) -> Result<String, String> {
    reqwest::get(&url)
        .await
        .map_err(|e| e.to_string())?
        .text()
        .await
        .map_err(|e| e.to_string())
}
```

- Async commands cannot borrow arguments across `.await`; take owned types (`String`, `Vec<T>`), not `&str`.
- If an async command takes any borrowed argument (including `State<'_, T>`), it must return `Result`.
- Do not block the async runtime with `std::thread::sleep` or synchronous I/O; use `tokio::time::sleep` or spawn blocking work.

### Separate modules

```rust
// src-tauri/src/commands.rs
#[tauri::command]
pub fn ping() -> String {
    "pong".into()
}
```

```rust
mod commands;

tauri::Builder::default()
    .invoke_handler(tauri::generate_handler![commands::ping])
```

The JavaScript name is still `ping`; the module path is only for Rust.

## Events and channels

Use **commands** for request/response. Use **events** for small notifications or many-to-many fan-out. Use **channels** for ordered, high-throughput streams such as progress or logs.

### Events from Rust

```rust
use tauri::{AppHandle, Emitter};

#[tauri::command]
fn start_download(app: AppHandle, url: String) {
    app.emit("download-started", &url).unwrap();
    app.emit_to("main", "download-progress", 50).unwrap();
}
```

- `emit` sends to all listeners.
- `emit_to(label, event, payload)` targets one window/webview.
- `emit_filter` targets a predicate.
- Payload must implement `Serialize + Clone`.

### Listen from the frontend

```ts
import { listen } from '@tauri-apps/api/event';

const unlisten = await listen<number>('download-progress', (event) => {
  console.log(event.payload);
});

// Later, when the component unmounts:
unlisten();
```

Pitfalls:

- `listen()` returns a promise. Await it before calling the unlisten handle; calling it synchronously does nothing useful.
- Always clean up listeners in component teardown paths. Page reloads and navigations clear them, but SPA route changes do not.
- Event handlers are not ordered when async. Use a channel for ordered streams.
- Events are not covered by capabilities, so never put secrets in event payloads.

### Channels

```rust
use tauri::ipc::Channel;
use serde::Serialize;

#[derive(Clone, Serialize)]
#[serde(tag = "event", content = "data", rename_all = "camelCase")]
enum DownloadEvent {
    Progress { percent: u32 },
    Finished { path: String },
}

#[tauri::command]
async fn download(url: String, on_event: Channel<DownloadEvent>) {
    for percent in [10, 50, 100] {
        on_event.send(DownloadEvent::Progress { percent }).unwrap();
    }
    on_event
        .send(DownloadEvent::Finished { path: "/tmp/file".into() })
        .unwrap();
}
```

```ts
import { invoke, Channel } from '@tauri-apps/api/core';

const onEvent = new Channel<DownloadEvent>();
onEvent.onmessage = (message) => console.log(message);

await invoke('download', { url: 'https://example.com/file', onEvent });
```

## State

```rust
use std::sync::Mutex;
use tauri::{Builder, Manager, State};

#[derive(Default)]
struct AppState {
    counter: u32,
}

#[tauri::command]
fn increment(state: State<'_, Mutex<AppState>>) -> u32 {
    let mut guard = state.lock().unwrap();
    guard.counter += 1;
    guard.counter
}

fn main() {
    Builder::default()
        .manage(Mutex::new(AppState::default()))
        .invoke_handler(tauri::generate_handler![increment])
        .run(tauri::generate_context!())
        .unwrap();
}
```

- Tauri stores state behind `Arc`; do not wrap managed state in your own `Arc`.
- The type passed to `manage()` must match the `State<T>` type exactly. `State<'_, AppState>` does not match `manage(Mutex::new(AppState::default()))`.
- A wrong state type is a runtime error: the command rejects with `state not managed for field ...`, while `app.state::<T>()` panics.
- Access state outside commands with `app_handle.state::<T>()`.
- Prefer `std::sync::Mutex` for short critical sections. Use `tokio::sync::Mutex` only when you must hold the guard across `.await`.
- Avoid holding a mutex guard across an `.await`; it can deadlock the app.

## Windows and menus

Tauri 2 uses `WebviewWindow` for both windows and webviews.

```rust
use tauri::{Manager, WebviewUrl, WebviewWindowBuilder};

tauri::Builder::default().setup(|app| {
    let window = WebviewWindowBuilder::new(app, "main", WebviewUrl::default())
        .title("My App")
        .inner_size(1000.0, 700.0)
        .build()?;

    window.set_focus()?;
    Ok(())
});
```

```rust
#[tauri::command]
fn focus_main(app: tauri::AppHandle) {
    if let Some(window) = app.get_webview_window("main") {
        let _ = window.set_focus();
    }
}
```

- Window labels are the identity used by capabilities and `emit_to`.
- Create windows from config (`app.windows`) or at runtime (`WebviewWindowBuilder`).
- Use `data-tauri-drag-region` on custom titlebars and grant the needed `core:window:*` permissions.
- Tray icons need the `tray-icon` Cargo feature; see `tutorials.md`.
- Menu APIs are in `tauri::menu`; on macOS top-level menu entries must be submenus.

## Resources

Bundle extra files with `bundle.resources`:

```json
{
  "bundle": {
    "resources": ["lang/*", "resources/**/*.md"]
  }
}
```

Resolve paths instead of hardcoding them:

```rust
use tauri::path::BaseDirectory;

#[tauri::command]
fn read_lang(app: tauri::AppHandle) -> Result<String, String> {
    let path = app
        .path()
        .resolve("lang/de.json", BaseDirectory::Resource)
        .map_err(|e| e.to_string())?;
    std::fs::read_to_string(path).map_err(|e| e.to_string())
}
```

```ts
import { resolveResource } from '@tauri-apps/api/path';

const path = await resolveResource('lang/de.json');
```

- `../` becomes `_up_` and an absolute root becomes `_root_` inside the resource directory.
- To read resources from JavaScript, grant `fs:allow-read-text-file` plus a `$RESOURCE` scope (for example `fs:allow-resource-read-recursive`).
- On Android resources live inside the APK; use the `fs` plugin rather than `std::fs` for them.

## Sidecars

Bundle an external executable with `bundle.externalBin`:

```json
{
  "bundle": {
    "externalBin": ["binaries/my-sidecar"]
  }
}
```

The actual file must carry the target triple suffix, for example `my-sidecar-x86_64-pc-windows-msvc.exe`. Get the suffix with `rustc --print host-tuple`.

Run it from Rust after initializing the shell plugin:

```rust
use tauri_plugin_shell::ShellExt;

let command = app.shell().sidecar("my-sidecar").unwrap();
let (mut rx, mut child) = command.spawn().unwrap();
```

From JavaScript:

```ts
import { Command } from '@tauri-apps/plugin-shell';

const command = Command.sidecar('binaries/my-sidecar', ['--flag']);
const output = await command.execute();
```

Grant the sidecar in a capability:

```json
{
  "identifier": "shell:allow-execute",
  "allow": [
    {
      "name": "binaries/my-sidecar",
      "sidecar": true,
      "args": ["--flag", { "validator": "\\S+" }]
    }
  ]
}
```

`Command.sidecar` takes the same string as `externalBin`. `app.shell().sidecar()` takes only the file name (`"my-sidecar"`). Missing the shell permission produces a denied command at runtime.

## Mobile development

```sh
npm run tauri android dev
npm run tauri ios dev 'iPhone 15'
npm run tauri android build -- --aab
npm run tauri ios build -- --export-method app-store-connect
```

Requirements:

- `[lib] crate-type = ["staticlib", "cdylib", "rlib"]` and `#[cfg_attr(mobile, tauri::mobile_entry_point)]` on `run()`.
- Android: SDK, NDK, `JAVA_HOME`, `ANDROID_HOME`, `NDK_HOME`, and Rust targets:
  ```sh
  rustup target add aarch64-linux-android armv7-linux-androideabi \
    i686-linux-android x86_64-linux-android
  ```
- iOS: macOS with Xcode and CocoaPods; targets:
  ```sh
  rustup target add aarch64-apple-ios x86_64-apple-ios aarch64-apple-ios-sim
  ```
- For a physical iOS device, the dev server must listen on `process.env.TAURI_DEV_HOST`; run `tauri ios dev --force-ip-prompt` to select the device address.
- Not every plugin or API supports mobile. Check the plugin page's platform table and guard desktop-only code with `#[cfg(desktop)]`.

## Debugging

- Rust output appears in the terminal running `tauri dev`.
- Open the WebView inspector with right-click → Inspect, or `Ctrl+Shift+I` / `Cmd+Option+I`.
- `tauri::is_dev()`, `cfg!(dev)`, and `cfg!(debug_assertions)` distinguish dev and debug builds.
- `tauri build --debug` produces a debuggable build in `src-tauri/target/debug/bundle`.
- On Linux, `RUST_BACKTRACE=1 tauri dev` (or `$env:RUST_BACKTRACE=1` in PowerShell) gives a stack trace.
- `tauri info` prints versions, targets, and environment details.
- `.taurignore` excludes files from the dev watcher, using `.gitignore` syntax.
- For production WebView debugging, add the `devtools` Cargo feature; do not ship it to the App Store on macOS (private API).

## Testing

Frontend unit tests can mock IPC without a running Rust backend:

```ts
import { beforeEach, expect, test, vi } from 'vitest';
import { clearMocks, mockIPC } from '@tauri-apps/api/mocks';
import { invoke } from '@tauri-apps/api/core';

beforeEach(() => {
  clearMocks();
});

test('calls add', async () => {
  mockIPC((cmd, args) => {
    if (cmd === 'add') return (args.a as number) + (args.b as number);
  });

  await expect(invoke('add', { a: 2, b: 3 })).resolves.toBe(5);
});
```

- `mockWindows('main', 'settings')` fakes window labels.
- `mockIPC(..., { shouldMockEvents: true })` enables partial event mocking (Tauri 2.7+); `emitTo` and `emit_filter` are not covered.
- Call `clearMocks()` after each test.
- Rust-side tests can use `tauri::test::mock_builder` and `mock_context`; the test runtime does not create a real webview.
- End-to-end tests use WebDriver + WebdriverIO (`@wdio/tauri-service`), which supports Windows, Linux, and macOS. See `ci.md`.

## Pitfalls

| Problem | Cause | Fix |
| --- | --- | --- |
| Command not found | Not registered in `generate_handler![]` | Add it. |
| `pub fn` command in `lib.rs` fails to compile | Glue-code name collision | Remove `pub`; use `pub` only in separate modules. |
| Async command with `&str` fails | Borrowed data cannot cross `.await` | Take `String`. |
| State rejection | `State<T>` type mismatch | Match `manage()` exactly. |
| Event listener never fires | `listen()` result not awaited, or event name/target mismatch | Await the promise; check `emit_to` label. |
| High CPU / frozen UI | Blocking work on the main thread | Use async or `spawn_blocking`. |
| Mobile build cannot find library | Missing `crate-type` or mobile entry point | Add all three crate types and the `mobile_entry_point` attribute. |
| Sidecar not found | Binary lacks target-triple suffix | Rename with `rustc --print host-tuple`. |
