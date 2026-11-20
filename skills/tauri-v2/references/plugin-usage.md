# Using Tauri 2 plugins

Tauri core stays small; most system access lives in plugins. A plugin can ship a Rust crate, an npm package, permissions, and mobile native code. Installing the dependency is only half the job — the plugin must be registered in the builder and granted in a capability.

Official plugin catalog: <https://v2.tauri.app/plugin/>.

## Contents

- [Install flow](#install-flow)
- [Official plugins](#official-plugins)
- [Plugin configuration](#plugin-configuration)
- [Recipes](#recipes)
- [Mobile support](#mobile-support)
- [Community plugins](#community-plugins)
- [Pitfalls](#pitfalls)

## Install flow

The fast path:

```sh
npm run tauri add fs
```

This runs the CLI's plugin installer, which normally:

1. Adds `tauri-plugin-fs` to `src-tauri/Cargo.toml`.
2. Registers `.plugin(tauri_plugin_fs::init())` in `src-tauri/src/lib.rs`.
3. Adds `@tauri-apps/plugin-fs` to `package.json`.
4. Adds `fs:default` to `src-tauri/capabilities/default.json`.

Verify all four. If the CLI could not edit a file, finish the step manually.

Manual install:

```sh
cargo add tauri-plugin-fs
npm install @tauri-apps/plugin-fs
```

```rust
tauri::Builder::default()
    .plugin(tauri_plugin_fs::init())
```

```json
{
  "identifier": "default",
  "windows": ["main"],
  "permissions": ["core:default", "fs:default"]
}
```

Some plugins do not use `init()`; they take a builder, a callback, or config:

```rust
.plugin(tauri_plugin_single_instance::init(|app, argv, cwd| {
    // handle the second instance
}))
```

```rust
.plugin(tauri_plugin_updater::Builder::new().build())
```

Always check the plugin page for the exact registration call.

## Official plugins

`tauri add <name>` accepts the short name below. Permission identifiers use the same name (`fs:default`, `dialog:allow-open`, ...).

| Plugin | Short name | What it provides | Notes |
| --- | --- | --- | --- |
| Autostart | `autostart` | Launch on login/startup | Desktop. |
| Barcode Scanner | `barcode-scanner` | Camera barcode/QR scanning | Android/iOS. |
| Biometric | `biometric` | Face/fingerprint auth | Android/iOS. |
| CLI | `cli` | Parse command-line arguments | Desktop; config under `plugins.cli`. |
| Clipboard | `clipboard-manager` | Read/write clipboard | Desktop and mobile. |
| Deep Link | `deep-link` | Custom URL schemes and app links | Config under `plugins.deep-link`; mobile needs intent/associated-domain setup. |
| Dialog | `dialog` | Native open/save/message dialogs | Desktop and mobile; pairs with fs scopes. |
| File System | `fs` | Read/write files and directories | Scoped; default allows reads in app dirs only. |
| Geolocation | `geolocation` | Device location | Android/iOS; needs OS permission. |
| Global Shortcut | `global-shortcut` | System-wide keyboard shortcuts | Desktop. |
| Haptics | `haptics` | Vibration feedback | Android/iOS. |
| HTTP | `http` | HTTP client that bypasses WebView CORS | Scoped by URL; desktop and mobile. |
| Localhost | `localhost` | Serve app assets over localhost instead of the custom protocol | Rust-only; security caution. |
| Log | `log` | Structured logging to stdout/file/WebView | Config under `plugins.log`. |
| NFC | `nfc` | NFC read/write | Android/iOS. |
| Notification | `notification` | Native notifications | Desktop and mobile; request permission. |
| Opener | `opener` | Open URLs and files with the OS | Prefer over shell for links. |
| OS | `os` | OS name, version, arch, locale | Desktop and mobile. |
| Persisted Scope | `persisted-scope` | Persist fs/asset scopes across restarts | Rust-only; register after `fs`. |
| Positioner | `positioner` | Position windows relative to tray/status bar | Desktop; JS and Rust APIs. |
| Process | `process` | Relaunch/exit the app | Used after updater install. |
| Shell | `shell` | Execute commands, open URLs, sidecars | High risk; scope binaries and args. |
| Single Instance | `single-instance` | Ensure one running instance | Desktop; init with callback. |
| SQL | `sql` | SQLite/MySQL/PostgreSQL client | Desktop and mobile; migrations via builder. |
| Store | `store` | Persistent key-value store | Desktop and mobile. |
| Stronghold | `stronghold` | Encrypted secrets vault | Desktop and mobile. |
| Updater | `updater` | Signed in-app updates | Desktop; requires signer keys. |
| Upload | `upload` | Streaming uploads with progress | Uses a custom protocol. |
| WebSocket | `websocket` | WebSocket client with channels | Desktop and mobile. |
| Window State | `window-state` | Save/restore size and position | Desktop. |

Exact npm package names follow `@tauri-apps/plugin-<short-name>`. Rust-only plugins such as `persisted-scope`, `positioner`, and `single-instance` may not have a JavaScript API; check the plugin page.

## Plugin configuration

Plugin settings go under `plugins` in `tauri.conf.json`, keyed by the short plugin name:

```json
{
  "plugins": {
    "log": {
      "level": "info",
      "targets": ["stdout", "webview"]
    },
    "updater": {
      "pubkey": "CONTENT OF PUBLIC KEY",
      "endpoints": ["https://releases.example.com/{{target}}/{{arch}}/{{current_version}}"]
    }
  }
}
```

The plugin parses its own config. Unknown fields may be rejected depending on the plugin, so copy the field names from the plugin page.

## Recipes

### File system

```rust
// Rust side
use tauri_plugin_fs::FsExt;

let contents = app.fs().read_to_string("path")?;
```

```ts
import { readTextFile, writeTextFile, BaseDirectory } from '@tauri-apps/plugin-fs';

const text = await readTextFile('config.json', { baseDir: BaseDirectory.AppConfig });
await writeTextFile('config.json', '{}', { baseDir: BaseDirectory.AppConfig });
```

```json
{
  "permissions": [
    "fs:default",
    {
      "identifier": "fs:allow-write-text-file",
      "allow": [{ "path": "$APPCONFIG/config.json" }]
    }
  ]
}
```

`fs:default` grants read commands and access to app directories. A path picked through the dialog plugin is added to the runtime fs scope for the current session; install `persisted-scope` if it must remain allowed after a restart.

### Dialog

```ts
import { open, save, message } from '@tauri-apps/plugin-dialog';

const file = await open({ multiple: false, directory: false });
const target = await save({ defaultPath: 'export.csv' });
await message('Done', { title: 'Export', kind: 'info' });
```

```json
{ "permissions": ["dialog:default"] }
```

Use the returned path with a scoped `fs` command. On mobile, `open` can return content URIs.

### Shell and sidecars

```ts
import { Command, open } from '@tauri-apps/plugin-shell';

await open('https://v2.tauri.app/');
const result = await Command.create('git', ['status']).execute();
```

```json
{
  "identifier": "shell:allow-execute",
  "allow": [{ "name": "git", "args": ["status"] }]
}
```

Prefer the `opener` plugin for URLs and files. Shell commands are high risk; list exact binaries and arguments, never a wildcard.

### HTTP

```ts
import { fetch } from '@tauri-apps/plugin-http';

const response = await fetch('https://api.example.com/data');
```

```json
{
  "permissions": [
    {
      "identifier": "http:default",
      "allow": [{ "url": "https://api.example.com/*" }],
      "deny": [{ "url": "https://api.example.com/private/*" }]
    }
  ]
}
```

The HTTP plugin bypasses WebView CORS but is still scoped by URL. Do not grant `https://*`.

### Store

```ts
import { load } from '@tauri-apps/plugin-store';

const store = await load('settings.json', { autoSave: true });
await store.set('theme', 'dark');
const theme = await store.get<string>('theme');
```

```json
{ "permissions": ["store:default"] }
```

### Updater and process

```ts
import { check } from '@tauri-apps/plugin-updater';
import { relaunch } from '@tauri-apps/plugin-process';

const update = await check();
if (update) {
  await update.downloadAndInstall();
  await relaunch();
}
```

```json
{ "permissions": ["updater:default", "process:allow-restart"] }
```

The updater requires a public key, HTTPS endpoints, and signed artifacts; see `distribution.md`.

### Single instance

```rust
tauri::Builder::default()
    .plugin(tauri_plugin_single_instance::init(|app, argv, cwd| {
        if let Some(window) = app.get_webview_window("main") {
            let _ = window.set_focus();
        }
    }))
```

Deep links and file associations often need single-instance so a second launch forwards its arguments to the running app.

### Deep link

```json
{
  "plugins": {
    "deep-link": {
      "desktop": {
        "schemes": ["myapp"]
      }
    }
  }
}
```

```json
{ "permissions": ["deep-link:default"] }
```

Mobile also requires Android intent filters and iOS associated domains; run `tauri android init` / `tauri ios init` and follow the plugin page.

### Window state

```rust
tauri::Builder::default().plugin(tauri_plugin_window_state::Builder::default().build())
```

```json
{ "permissions": ["window-state:default"] }
```

Save/restore is automatic for windows created after the plugin is registered; check the plugin's `StateFlags` options if you want partial persistence.

### Global shortcut

```rust
use tauri_plugin_global_shortcut::GlobalShortcutExt;

app.global_shortcut().register("CommandOrControl+Shift+K")?;
```

```json
{ "permissions": ["global-shortcut:allow-register", "global-shortcut:allow-unregister"] }
```

Desktop only; shortcuts are global to the OS.

## Mobile support

- Desktop-only: autostart, CLI, global shortcut, positioner, single instance, updater, window state.
- Mobile-only: barcode scanner, biometric, geolocation, haptics, NFC.
- Both, with platform caveats: clipboard, deep link, dialog, fs, HTTP, log, notification, opener, OS, process, shell, SQL, store, stronghold, websocket.
- Rust-only: persisted-scope, localhost.

Check the support table on <https://v2.tauri.app/plugin/> before promising mobile support. Guard desktop-only code:

```rust
#[cfg(desktop)]
{
    use tauri_plugin_global_shortcut::GlobalShortcutExt;
    app.global_shortcut().register("CommandOrControl+Shift+K")?;
}
```

## Community plugins

Search crates.io for `tauri-plugin-<name>` and npm for `@scope/plugin-<name>`. Before adopting one:

- Check Tauri 2 support, last release, open issues, license, and platform coverage.
- Read its permissions and scopes; a plugin with a broad default scope can undo your capability design.
- Prefer plugins that use `tauri_plugin::Builder` and ship `permissions/default.toml`.
- Pin a version or commit; do not follow a moving branch in production.

## Pitfalls

| Problem | Cause | Fix |
| --- | --- | --- |
| Plugin command denied | `tauri add` did not add the permission, or the capability is not enabled | Add `<plugin>:default`; check `app.security.capabilities`. |
| Plugin not initialized | Cargo dependency added but no `.plugin(...)` | Register it in `lib.rs`. |
| npm import fails | Rust crate installed, npm package missing | `npm install @tauri-apps/plugin-<name>`. |
| Mobile build fails | Plugin is desktop-only or missing mobile targets | Check the platform table; guard with `#[cfg(desktop)]`. |
| FS access denied after dialog selection | Selection does not grant persistent scope | Add a scope or install `persisted-scope`. |
| Shell executable denied | Scope missing or name mismatch | Allow the exact binary and arguments. |
| Updater check fails | Signing key, endpoint, or artifact mismatch | See `distribution.md`. |
