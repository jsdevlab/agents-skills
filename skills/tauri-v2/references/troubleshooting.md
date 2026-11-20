# Tauri 2 troubleshooting

Start with `tauri info`; it prints CLI, Rust, Node, OS, WebView, and target details. Then reproduce with `RUST_BACKTRACE=1` (PowerShell: `$env:RUST_BACKTRACE=1`) and open the WebView inspector.

Official docs: <https://v2.tauri.app/develop/debug/>.

## Contents

- [Diagnostic checklist](#diagnostic-checklist)
- [Build failures](#build-failures)
- [Blank window and frontend issues](#blank-window-and-frontend-issues)
- [Command and permission errors](#command-and-permission-errors)
- [Mobile](#mobile)
- [Linux graphics](#linux-graphics)
- [Windows](#windows)
- [macOS](#macos)
- [Updater and signing](#updater-and-signing)
- [Migrating from Tauri 1](#migrating-from-tauri-1)

## Diagnostic checklist

1. `npm run tauri info` — versions and environment.
2. `npm run tauri dev -- --verbose` — CLI and build detail.
3. Rust terminal output — panics and command errors.
4. WebView DevTools (`Ctrl+Shift+I` / `Cmd+Option+I`) — frontend, CSP, IPC errors.
5. `cargo check --manifest-path src-tauri/Cargo.toml` — Rust-only compile errors.
6. `git status` and `tauri.conf.json` diff — accidental config or version changes.
7. On mobile, run `tauri android dev --open` or `tauri ios dev --open` and inspect Logcat/Xcode.

## Build failures

| Error | Cause | Fix |
| --- | --- | --- |
| `failed to run custom build command for tauri` | Missing system libraries, schema generation failure, or invalid config | Install WebKitGTK/GTK deps; run `cargo check` for the real error. |
| `glib-2.0` / `webkit2gtk` not found | Linux development packages missing | Install `libwebkit2gtk-4.1-dev`, `libgtk-3-dev`, `libayatana-appindicator3-dev`, `librsvg2-dev`. |
| `linker 'cc' not found` | No C toolchain | Install `build-essential` / Xcode Command Line Tools / MSVC Build Tools. |
| `light.exe` failed | MSI build without VBSCRIPT | Enable Windows optional feature VBSCRIPT, or build NSIS only. |
| `failed to bundle project` on Linux | Missing `patchelf`, `librsvg`, or distro package | Install the Linux dependencies from the CI reference. |
| `GLIBC_2.xx not found` at runtime | Built on a newer Linux than the target | Build on Ubuntu 22.04 / Debian 12. |
| `error: linker 'x86_64-w64-mingw32-gcc' not found` | Cross-compiling Windows without toolchain | Use `cargo-xwin`, NSIS, and the MSVC target. |
| Feature not found (`tray-icon`, `devtools`) | Cargo feature not enabled | Add it to the `tauri` dependency. |
| `identifier` invalid | Underscore or another invalid character; the `.app` suffix triggers a warning | Use reverse-DNS with alphanumerics, hyphens, and periods; avoid `.app`. |

## Blank window and frontend issues

- `devUrl` must match the dev server port. `tauri dev` waits for the URL; a mismatch shows a blank window or times out.
- `frontendDist` must point at the built assets, not the source directory, unless you are using the no-framework mode.
- `beforeDevCommand` / `beforeBuildCommand` must actually start/build the frontend. Test them directly outside Tauri.
- For a Windows absolute `frontendDist`, use `\\\\?\\D:\\dist`; otherwise `D:` is parsed as a URL scheme.
- A restrictive CSP can block scripts, styles, fonts, or `connect-src`. Check the DevTools console; add only the required source, not `unsafe-eval`.
- If `withGlobalTauri` is false, `window.__TAURI__` is unavailable; use the npm package.
- On macOS, a transparent window with no background can look blank; check the HTML body background and `transparent` config.
- If the window exists but is hidden, check `visible` and splashscreen logic.

### Split frontend and shell

| Symptom | Cause | Fix |
| --- | --- | --- |
| `tauri dev` opens a blank window | The frontend dev server is not running; with no `beforeDevCommand`, Tauri does not start it | Start the web dev server first; check `devUrl` host/port. |
| Packaged app is blank | `frontendDist` points at a source directory, a stale build, or the wrong repo | Build the web project; resolve `frontendDist` from `src-tauri`; verify the directory contains `index.html`. |
| `frontendDist` not found in CI | The web repo/dir was not checked out or built before the shell | Build the frontend first; check the path and job order. |
| Release assets missing | `projectPath` points at the repo root instead of the directory containing `src-tauri` | Set `projectPath: shell` (or `apps/desktop`). |
| `tauri build` cannot find the project | Running from the repo/workspace root | Run it from the directory containing `src-tauri`; the CLI has no `--project-dir`. |
| Frontend calls fail after a shell upgrade | `@tauri-apps/api` / plugin JS versions are older than the shell | Upgrade the frontend packages together with the shell. |
| Mobile dev loads desktop localhost | The web dev server binds only to `127.0.0.1` | Bind to `0.0.0.0` and use `TAURI_DEV_HOST` in the Vite/framework config. |

## Command and permission errors

| Error | Fix |
| --- | --- |
| `Command <name> not found` | Add the command to `tauri::generate_handler![]`; check the exact name (no `plugin:` prefix for app commands). |
| `Command plugin:<name>|<cmd> not found` | Register the plugin and its `invoke_handler`; verify the plugin name. |
| `... not allowed. Permissions associated with this command: ...` | Add the listed permission to the capability for that window. |
| Scope still denies the path | Add a scope with the resolved absolute path or `$VAR`; `deny` overrides `allow`. |
| Capability has no effect | `app.security.capabilities` is set and omits it, or the window label does not match. |
| `state not managed for field ...` | `State<T>` does not match `manage()` exactly; use the same wrapper (`Mutex<T>`). |
| `state not managed` panic | `app.state::<T>()` for a type never managed. |
| Async command borrow error | Use owned parameters (`String`, `Vec<T>`); return `Result` if borrowing state. |
| `AppError` does not implement `Serialize` | Implement `Serialize` for the error or return `String`. |

Permission example:

```text
fs.write_text_file not allowed. Permissions associated with this command:
fs:allow-app-write, fs:allow-app-write-recursive, ...
```

Add `fs:allow-write-text-file` **and** a scope for the target path. The command name in the error maps to the permission name with `allow-`.

## Mobile

| Symptom | Cause | Fix |
| --- | --- | --- |
| `failed to find target with hash string 'aarch64-linux-android'` | Rust target missing | `rustup target add aarch64-linux-android ...`. |
| `SDK location not found` | `ANDROID_HOME`/`ANDROID_SDK_ROOT` missing | Set `ANDROID_HOME`; install SDK/NDK/build-tools. |
| `NDK not configured` | `NDK_HOME` missing or wrong | Set `NDK_HOME` to the NDK version directory. |
| `JAVA_HOME is not set` | Android Studio JBR not configured | Set `JAVA_HOME` to Android Studio's `jbr`. |
| iOS build script cannot find `node`/`cargo` | Xcode build phase does not inherit shell PATH | Avoid version managers; use absolute paths or a script that sources the environment. |
| iOS first run asks for local network access | The dev server is exposed on the LAN | Allow it, or use `--force-ip-prompt` with the device TUN address. |
| App builds but Rust code is not called | `main.rs` contains logic instead of calling `lib.rs::run()` | Move logic to `lib.rs`; keep `main.rs` thin. |
| Plugin works on desktop, not mobile | Plugin is desktop-only | Check the plugin platform table; gate with `#[cfg(desktop)]`. |
| Android Play rejects bundle | 16 KB page alignment | Use NDK 28+ or the `max-page-size=16384` rustflag. |

## Linux graphics

Symptoms: blank/white window, flicker, crash on resize, `AcceleratedSurfaceDMABuf was unable to construct a complete framebuffer`, `Gdk-Message: Error 71 ... Wayland`.

Workarounds, in order:

1. Enable kernel mode setting for NVIDIA (`nvidia_drm.modeset=1` on older drivers).
2. `__NV_DISABLE_EXPLICIT_SYNC=1`.
3. `WEBKIT_DISABLE_DMABUF_RENDERER=1`.
4. `WEBKIT_DISABLE_COMPOSITING_MODE=1` as a last resort.

Set them in the shell for testing, or in `main()` before creating the WebView so users do not need to. These reduce acceleration; prefer fixing the driver when possible.

Also check that the app was built against WebKitGTK 4.1, not an old 4.0 development package.

## Windows

- **MSI**: requires Windows and the VBSCRIPT optional feature. If `light.exe` fails, enable VBSCRIPT. WiX cannot run on Linux/macOS.
- **NSIS**: can be cross-compiled with `cargo-xwin` + NSIS + LLVM/lld, but test on Windows. MSI cannot be cross-compiled.
- **WebView2**: the default installer downloads the bootstrapper. Offline environments need `offlineInstaller`; Windows 7 needs `embedBootstrapper` or offline.
- **SmartScreen**: unsigned downloads warn; reputation builds over time. EV certificates no longer grant instant reputation. Sign every release with the same certificate.
- **Silent install**: NSIS `/S` (uppercase), MSI `/quiet`; required by the Microsoft Store.
- **Publisher conflict**: `publisher` must differ from `productName` for the Microsoft Store.

## macOS

- **"App is damaged"**: unsigned or not notarized Apple Silicon build. Sign with a Developer ID and notarize, or use ad-hoc signing for local testing.
- **Gatekeeper still warns with ad-hoc signing**: expected; users must allow it in Privacy & Security.
- **App Store rejection**: sandbox entitlement, provisioning profile, `category`, encryption declaration (`ITSAppUsesNonExemptEncryption`), and no private APIs (including `devtools`).
- **Universal build fails**: install both `aarch64-apple-darwin` and `x86_64-apple-darwin`, and build with `--target universal-apple-darwin`.
- **DMG layout missing on CI**: known issue; icon positions are not applied on CI runners.
- **`xcrun: error: unable to find utility altool`**: use Xcode's `xcrun altool` with an App Store Connect API key, or Transporter.
- **PATH (macOS/Linux)**: GUI apps do not inherit the shell `PATH`; use `fix-path-env-rs` or absolute paths.

## Updater and signing

| Error | Cause | Fix |
| --- | --- | --- |
| `signature verification failed` | Wrong public key, wrong `.sig`, or mismatched artifact | Regenerate the manifest from the exact `.sig`; verify the public key. |
| `pubkey` parse error | Path used instead of key content | Paste the public key content. |
| No update found | Version not greater, endpoint 204, or platform key mismatch | Check SemVer, endpoint response, and `OS-ARCH` keys. |
| HTTPS error | Plain HTTP endpoint | Use HTTPS; `dangerousInsecureTransportProtocol` only for local testing. |
| Updater artifacts missing | `createUpdaterArtifacts` false | Set it to `true` and rebuild with the signing key. |
| Windows install fails silently | Insufficient privileges with `quiet` mode | Use `passive` or per-machine install. |
| Private key lost | Cannot sign future updates | There is no recovery; keep backups offline. |

See `distribution.md` for key generation, manifest formats, and install modes.

## Migrating from Tauri 1

Run the automated migration first, then read the output:

```sh
npm install @tauri-apps/cli@latest
npm run tauri migrate
```

Key changes:

| Tauri 1 | Tauri 2 |
| --- | --- |
| `package.productName`, `package.version` | top-level `productName`, `version` |
| `tauri` config key | `app` |
| `tauri.allowlist` | capabilities + permissions |
| `build.distDir` | `build.frontendDist` |
| `build.devPath` | `build.devUrl` |
| `tauri.bundle` | top-level `bundle` |
| `tauri.systemTray` | `app.trayIcon` |
| `tauri.updater` | `plugins.updater` + `bundle.createUpdaterArtifacts` |
| `@tauri-apps/api/tauri` | `@tauri-apps/api/core` |
| `WebviewWindow` imported from `@tauri-apps/api/window` | import it from `@tauri-apps/api/webviewWindow` (`getCurrentWindow`/`Window` stay in `@tauri-apps/api/window`) |
| `app.get_window()` | `app.get_webview_window()` |
| `emit` to everyone | `emit` global; `emit_to` targeted |
| `listen_global` | `listen_any` |
| `system-tray` feature | `tray-icon` feature |
| `TAURI_PRIVATE_KEY` | `TAURI_SIGNING_PRIVATE_KEY` |
| `TAURI_KEY_PASSWORD` | `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` |

For mobile, add `[lib] crate-type = ["staticlib", "cdylib", "rlib"]`, move logic to `lib.rs`, and add `#[cfg_attr(mobile, tauri::mobile_entry_point)]` to `run()`. Then run `tauri android init` / `tauri ios init`.

Migration is not just renaming: v1 plugins are now separate crates (`tauri-plugin-dialog`, `tauri-plugin-shell`, ...), and the allowlist must be rebuilt as capabilities. Test each window and platform after migrating.
