# Tauri 2 security

Tauri's security model assumes the WebView is compromised and the Rust core is trusted. The IPC layer is the trust boundary: frontend code can only call commands that capabilities grant to its window, and command implementations must enforce scopes.

Official docs: <https://v2.tauri.app/security/>.

## Contents

- [Threat model](#threat-model)
- [Capabilities](#capabilities)
- [Permissions](#permissions)
- [Scopes](#scopes)
- [CSP](#csp)
- [Asset protocol](#asset-protocol)
- [HTTP headers](#http-headers)
- [Isolation pattern](#isolation-pattern)
- [Hardening checklist](#hardening-checklist)
- [Debugging permission errors](#debugging-permission-errors)

## Threat model

| Boundary | Trust | Implication |
| --- | --- | --- |
| Rust core and plugins | Trusted | Full system access; a bug here bypasses all WebView restrictions. |
| WebView content | Untrusted | Can run arbitrary JavaScript if XSS or remote content is allowed. |
| IPC | Enforced boundary | Capabilities decide which commands a window may call; scopes constrain arguments. |
| Dependencies | Varies | Audit npm and Cargo dependencies; a compromised dependency ships with the app. |
| Updates | High value | The updater must use HTTPS and signed artifacts; losing the private key blocks future updates. |

Capabilities do **not** protect against malicious Rust code, overly broad scopes, incorrect scope checks inside commands, compromised dependencies, or WebView zero-days. They minimize the impact of a frontend compromise and prevent accidental exposure of system APIs.

## Capabilities

A capability grants permissions to specific windows, webviews, and platforms.

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "main-capability",
  "description": "Capability for the main window",
  "windows": ["main"],
  "platforms": ["linux", "macOS", "windows"],
  "permissions": [
    "core:default",
    "core:window:allow-set-title",
    "fs:allow-read-text-file",
    {
      "identifier": "fs:scope",
      "allow": ["$HOME/documents/**/*"],
      "deny": ["$HOME/documents/private/**"]
    }
  ]
}
```

Rules:

- Files in `src-tauri/capabilities/` are automatically enabled when `app.security.capabilities` is not set or is an empty array. Once you list capabilities there explicitly, only those are used.
- `windows` matches window labels; use `["*"]` only when every window genuinely needs the permission.
- `platforms` accepts `linux`, `macOS`, `windows`, `iOS`, `android` (case-sensitive for the Apple names).
- `remote` grants access to remote URLs. Avoid it; on Linux and Android Tauri cannot distinguish an iframe from the window itself.
- Capabilities can be JSON or TOML.

Separate capabilities by risk instead of one giant `default.json`:

```text
src-tauri/capabilities/
├── default.json        # core:default for main
├── filesystem.json     # fs permissions for the editor window
├── updater.json        # updater permissions for main
└── mobile.json         # mobile-only plugins, platforms: ["iOS", "android"]
```

By default, every app command in `generate_handler![]` is callable by every window. To gate app commands with the ACL, list them in `build.rs`:

```rust
fn main() {
    tauri_build::try_build(
        tauri_build::Attributes::new()
            .app_manifest(tauri_build::AppManifest::new().commands(&["greet", "save_file"])),
    )
    .unwrap();
}
```

This generates `allow-greet`, `deny-greet`, `allow-save-file`, and `deny-save-file` app permissions. Add the `allow-*` entries to the capabilities for the windows that need them; app permission identifiers have no plugin prefix. Without an app manifest, app commands are not ACL-gated, while capabilities still gate plugin and core commands.

## Permissions

A permission allows or denies specific commands and can attach scopes.

```toml
[[permission]]
identifier = "allow-save"
description = "Enables the save command."
commands.allow = ["save"]

[[permission.scope.allow]]
path = "$APPDATA/**"
```

Identifiers:

- `<name>:default` — the plugin's default permission set.
- `<name>:allow-<command>` / `<name>:deny-<command>` — generated per command.
- Plugin prefix `tauri-plugin-` is added automatically; use `fs:allow-read-text-file`, not `tauri-plugin-fs:...`.

Permission sets group permissions:

```toml
[[set]]
identifier = "editor-access"
description = "Read and write documents"
permissions = ["fs:allow-read-text-file", "fs:allow-write-text-file", "dialog:allow-open"]
```

Application-level permissions live in `src-tauri/permissions/*.toml`; capabilities reference them by identifier. Use `default` for the minimum useful set and require explicit opt-in for dangerous commands.

## Scopes

Scopes constrain **arguments** of an allowed command. `deny` always wins over `allow`.

- Filesystem-style scopes use glob patterns: `$HOME/**/*`, `$APPDATA/**`, `$RESOURCE/**/*`.
- HTTP-style scopes allow/deny URLs.
- Shell scopes allow specific binaries and validate arguments.
- Scope types are plugin-specific; check the plugin page.

Inline scope in a capability:

```json
{
  "identifier": "fs:allow-write-text-file",
  "allow": [{ "path": "$HOME/test.txt" }]
}
```

A permission-level global scope:

```toml
[[permission]]
identifier = "allow-home-read"
description = "Read files under the home directory"

[[permission.scope.allow]]
path = "$HOME/**/*"
```

Command vs global scope in plugin code:

```rust
use tauri::ipc::{CommandScope, GlobalScope};

#[tauri::command]
async fn spawn(
    command_scope: CommandScope<'_, Entry>,
    global_scope: GlobalScope<'_, Entry>,
) -> Result<()> {
    let allowed = command_scope.allows();
    let denied = command_scope.denies();
    let also_allowed = global_scope.allows();
    // Enforce BOTH; do not trust arguments without checking.
    Ok(())
}
```

Pitfalls:

- A permission without a scope usually means "any argument", which is often too broad.
- A scope that looks right but never matches often uses a relative glob against an absolute resolved path. Use `$HOME`, `$APPDATA`, or a leading `/`.
- On Unix, `requireLiteralLeadingDot` defaults to `true`, so `*` and `**` do not match dot-prefixed path components. Name the segment literally or set it to `false` deliberately.
- `deny` is evaluated after `allow`; use it to carve out secrets, `.ssh`, `.env`, and WebView data directories.

## CSP

CSP is only active when configured. Tauri hashes local scripts and adds nonces for styles/external scripts at build time, so you only declare what is unique to your app.

```json
{
  "app": {
    "security": {
      "csp": {
        "default-src": "'self' customprotocol: asset:",
        "connect-src": "ipc: http://ipc.localhost",
        "font-src": ["https://fonts.gstatic.com"],
        "img-src": "'self' asset: http://asset.localhost blob: data:",
        "style-src": "'self' 'unsafe-inline' https://fonts.googleapis.com"
      }
    }
  }
}
```

- Prefer `'self'` and explicit hosts; avoid `*` and remote script CDNs.
- `connect-src` needs `ipc: http://ipc.localhost` for Tauri IPC.
- Use `devCsp` when the dev server needs looser rules than production.
- If a framework injects inline styles, keep `'unsafe-inline'` for `style-src` only, not `script-src`.
- Tauri appends hashes/nonces automatically; do not copy a build-time hash into the config.

## Asset protocol

`asset:` URLs let the WebView load local files. They are disabled until explicitly enabled.

```json
{
  "app": {
    "security": {
      "assetProtocol": {
        "enable": true,
        "scope": {
          "allow": ["$APPCACHE/**/*", "$RESOURCE/**/*"],
          "deny": ["$APPCACHE/**/secrets/**"],
          "requireLiteralLeadingDot": false
        }
      }
    }
  }
}
```

```ts
import { convertFileSrc } from '@tauri-apps/api/core';

const url = convertFileSrc('/path/to/image.png');
```

- Prefer narrow directories (`$APPCACHE`, `$RESOURCE`) over `$HOME/**/*` or `**/*`.
- Use `**/*` for files under a tree; bare `**` is easy to misuse.
- For user-selected paths that must survive restarts, use `tauri-plugin-persisted-scope`; enable its `protocol-asset` feature for asset URLs.
- Register `tauri_plugin_fs` before `tauri_plugin_persisted_scope`.

## HTTP headers

`app.security.headers` injects a limited set of response headers into WebView responses (not IPC responses):

```json
{
  "app": {
    "security": {
      "headers": {
        "Cross-Origin-Opener-Policy": "same-origin",
        "Cross-Origin-Embedder-Policy": "require-corp",
        "X-Content-Type-Options": "nosniff"
      }
    }
  }
}
```

Allowed names include `Access-Control-*`, `Cross-Origin-*`, `Permissions-Policy`, `Service-Worker-Allowed`, `Timing-Allow-Origin`, `X-Content-Type-Options`, and `Tauri-Custom-Header`. CSP is configured separately.

## Isolation pattern

The isolation pattern runs frontend code in an isolated iframe and validates IPC messages with a separate script. It is defense-in-depth for apps that load untrusted content.

```json
{
  "app": {
    "security": {
      "pattern": {
        "use": "isolation",
        "options": {
          "dir": "../dist-isolation"
        }
      }
    }
  }
}
```

The `dir` contains an `index.html` that intercepts and validates messages before forwarding them to Tauri. It adds complexity and is not a substitute for capabilities and CSP.

## Hardening checklist

- [ ] Grant the minimum permissions per window; split capabilities by feature.
- [ ] Use `platforms` so mobile-only and desktop-only plugins are not exposed everywhere.
- [ ] Scope filesystem, shell, HTTP, and asset protocol access narrowly; add explicit `deny` rules.
- [ ] Do not expose app commands to windows that do not need them (`AppManifest::commands`).
- [ ] Set a restrictive CSP and avoid remote scripts, remote content, and `unsafe-eval`.
- [ ] Keep `tauri`, `tauri-build`, the CLI, Rust, Node, and all dependencies updated; run `cargo audit` and `npm audit`.
- [ ] Never store secrets in frontend code, localStorage, or event payloads.
- [ ] Sign updater artifacts and serve update manifests over HTTPS; protect the private key.
- [ ] Sign Windows and macOS binaries; notarize macOS builds distributed outside the App Store.
- [ ] Only develop on trusted networks; the Tauri dev server has no mutual authentication or encryption.
- [ ] Report vulnerabilities privately via GitHub Security Advisories or <security@tauri.app>.

## Debugging permission errors

Typical message:

```text
fs.write_text_file not allowed. Permissions associated with this command:
fs:allow-app-write, fs:allow-app-write-recursive, ...
```

Steps:

1. Find the plugin and command in the docs (`fs` → `allow-write-text-file`).
2. Add the permission to the capability for the window that calls it.
3. Add a scope if the command is scoped; a bare permission often has no path access.
4. Confirm the capability is enabled: omit `app.security.capabilities` or include its identifier.
5. Regenerate schemas by running `tauri dev` or `tauri build`; the `$schema` files in `src-tauri/gen/schemas` are build outputs.
6. Check the window label; `windows: ["main"]` does not apply to a window labeled `settings`.

For a command you wrote, remember that app commands are not permission-gated by default; use `AppManifest::commands` and validate arguments inside the command.
