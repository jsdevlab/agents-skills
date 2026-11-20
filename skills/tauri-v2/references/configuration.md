# `tauri.conf.json` configuration

The Tauri configuration drives the CLI, the runtime, window creation, bundling, and plugin setup. Official reference: <https://v2.tauri.app/reference/config/>.

## Contents

- [File resolution and formats](#file-resolution-and-formats)
- [Merging and overrides](#merging-and-overrides)
- [Root keys](#root-keys)
- [Build](#build)
- [App](#app)
- [Security](#security)
- [Bundle](#bundle)
- [Plugins](#plugins)
- [CLI overrides and environment](#cli-overrides-and-environment)
- [Pitfalls](#pitfalls)

## File resolution and formats

The CLI looks for the configuration in the `src-tauri` directory. The default format is JSON (`tauri.conf.json`). JSON5 and TOML are opt-in:

```toml
[build-dependencies]
tauri-build = { version = "2", features = ["config-json5"] }

[dependencies]
tauri = { version = "2", features = ["config-json5"] }
```

Use `config-toml` for `Tauri.toml`. Enable the feature on **both** `tauri` and `tauri-build`, or the file is not recognized.

A minimal valid config:

```json
{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "My App",
  "version": "1.0.0",
  "identifier": "com.example.myapp",
  "build": {
    "frontendDist": "../dist"
  },
  "app": {
    "windows": [{ "label": "main", "title": "My App" }]
  },
  "bundle": {
    "active": true,
    "targets": "all"
  }
}
```

## Merging and overrides

Platform-specific files are read automatically:

- `tauri.linux.conf.json`
- `tauri.windows.conf.json`
- `tauri.macos.conf.json`
- `tauri.android.conf.json`
- `tauri.ios.conf.json`

They are merged with the base file using JSON Merge Patch (RFC 7396). Objects merge key by key; **arrays are replaced as a whole**, including nested arrays such as `bundle.resources` and `app.windows`. If a platform file overrides an array, repeat everything you want to keep.

A separate config can also be layered per invocation:

```sh
npm run tauri build -- --config src-tauri/tauri.beta.conf.json
```

This is useful for beta channels, App Store builds, or Microsoft Store builds. The `--config` value can be a path or a raw JSON string and uses the same merge rules.

## Root keys

| Key | Type | Meaning |
| --- | --- | --- |
| `productName` | string | Display/product name. |
| `version` | string | App version, or a path to a `package.json` containing the `version` field. If omitted, Tauri reads `package.version` from `src-tauri/Cargo.toml`. |
| `identifier` | string | Reverse-DNS bundle identifier, e.g. `com.example.myapp`. Must match the App Store/Play Console bundle ID. |
| `mainBinaryName` | string | Binary name; set it when the product name differs from the Cargo binary name. |
| `build` | object | Dev server, frontend output, and build hooks. |
| `app` | object | Windows, security, tray, and runtime behavior. |
| `bundle` | object | Installers, icons, resources, signing, and updater artifacts. |
| `plugins` | object | Per-plugin configuration keyed by plugin name. |

`identifier` rules: use reverse-DNS with alphanumeric characters, hyphens, and periods. Underscores and other characters are invalid. Tauri warns when the identifier ends in `.app` because it conflicts with the macOS bundle extension; avoid that suffix.

## Build

```json
{
  "build": {
    "devUrl": "http://localhost:5173",
    "frontendDist": "../dist",
    "beforeDevCommand": "npm run dev",
    "beforeBuildCommand": "npm run build",
    "beforeBundleCommand": "node scripts/prepare-bundle.mjs",
    "features": [],
    "runner": "cargo"
  }
}
```

| Key | Meaning |
| --- | --- |
| `devUrl` | URL the WebView loads in `tauri dev`. Must match the frontend dev server. |
| `frontendDist` | Directory or URL of built frontend assets. Relative paths are resolved from `src-tauri`. |
| `beforeDevCommand` | Shell command started before `tauri dev`; usually the frontend dev server. |
| `beforeBuildCommand` | Shell command run before `tauri build`; usually the frontend production build. |
| `beforeBundleCommand` | Command run after compilation but before bundling. |
| `features` | Cargo features to enable for the app crate. |
| `runner` | Cargo runner, e.g. `cargo-xwin` for cross-compiling. |

`beforeDevCommand` and `beforeBuildCommand` are optional. Omit them when the frontend lives in a separate directory or repository and is started/built by its own tooling:

```json
{
  "build": {
    "devUrl": "http://localhost:5173",
    "frontendDist": "../../web/dist"
  }
}
```

Both hooks also accept an object form, `{ "script": "...", "cwd": "../web" }` (`beforeDevCommand` additionally supports `wait`), which is convenient when the frontend lives in another directory.

`frontendDist` is resolved relative to the directory containing `tauri.conf.json` (`src-tauri`), and it can be a path or a URL. A URL means the packaged app loads remote content; prefer bundling local assets for offline apps. Run the Tauri CLI from the directory containing `src-tauri`; there is no `--project-dir`. In CI, `tauri-action`'s `projectPath` input points at that directory when the Tauri project is not at the repository root.

If there is no frontend framework, point `frontendDist` at a directory containing `index.html`; Tauri serves it directly.

Windows absolute paths in `frontendDist` are parsed as URLs (`D:\dist` looks like scheme `d`). Use the extended-length form `\\\\?\\D:\\path\\to\\dist` and backslashes only, or use a relative path.

## App

```json
{
  "app": {
    "withGlobalTauri": false,
    "windows": [
      {
        "label": "main",
        "title": "My App",
        "width": 1000,
        "height": 700,
        "minWidth": 640,
        "minHeight": 480,
        "resizable": true,
        "fullscreen": false,
        "center": true,
        "decorations": true,
        "transparent": false,
        "visible": true,
        "alwaysOnTop": false,
        "url": "index.html"
      }
    ],
    "trayIcon": {
      "iconPath": "icons/icon.png",
      "tooltip": "My App"
    },
    "security": {
      "csp": null
    }
  }
}
```

Common window keys:

| Key | Default | Notes |
| --- | --- | --- |
| `label` | `"main"` | Identity for capabilities and `emit_to`. |
| `url` | `index.html` | Can be a remote URL or an app route. |
| `width` / `height` | 800 / 600 | Logical pixels. |
| `minWidth` / `minHeight` | none | Size constraints. |
| `resizable` | `true` | Disable for fixed-size tools. |
| `decorations` | `true` | `false` for a custom titlebar. |
| `transparent` | `false` | Requires platform support and often `macOSPrivateApi`. |
| `visible` | `true` | Set `false` for splashscreen patterns. |
| `alwaysOnTop` | `false` | Overlay/utility windows. |
| `titleBarStyle` | `"Visible"` | macOS: `Visible`, `Transparent`, `Overlay`. |
| `useHttpsScheme` | `false` | Windows and Android origin scheme; changing it resets WebView storage. |

`withGlobalTauri: true` exposes `window.__TAURI__` for scripts that do not use the npm package. Prefer the npm package.

## Security

```json
{
  "app": {
    "security": {
      "csp": {
        "default-src": "'self'",
        "connect-src": "ipc: http://ipc.localhost",
        "img-src": "'self' asset: http://asset.localhost data: blob:",
        "style-src": "'self' 'unsafe-inline'"
      },
      "capabilities": ["default"],
      "assetProtocol": {
        "enable": true,
        "scope": ["$APPCACHE/**/*", "$RESOURCE/**/*"]
      },
      "headers": {
        "Cross-Origin-Opener-Policy": "same-origin"
      }
    }
  }
}
```

| Key | Meaning |
| --- | --- |
| `csp` | Content Security Policy. String or object. `null` disables the restriction. |
| `devCsp` | CSP override used only in development. |
| `capabilities` | Capability identifiers or inline capabilities. Omit it or set an empty array to auto-enable every file in `src-tauri/capabilities/`. |
| `assetProtocol.enable` | Enables `asset:` URLs for local files. |
| `assetProtocol.scope` | Allow/deny globs for the asset protocol. |
| `headers` | Response headers injected into WebView responses. |
| `pattern` | Isolation pattern configuration. |
| `freezePrototype` | Freezes `Object.prototype` against prototype pollution. |

Capabilities are covered in depth in `security.md`.

## Bundle

```json
{
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
    "resources": ["lang/*", "assets/**/*"],
    "externalBin": ["binaries/my-sidecar"],
    "fileAssociations": [
      { "ext": ["mydata"], "mimeType": "application/octet-stream" }
    ],
    "createUpdaterArtifacts": true,
    "category": "Utility",
    "publisher": "Example Inc.",
    "licenseFile": "../LICENSE",
    "windows": {
      "webviewInstallMode": { "type": "downloadBootstrapper" }
    },
    "macOS": {
      "minimumSystemVersion": "12.0"
    },
    "linux": {
      "deb": {},
      "rpm": {},
      "appimage": {}
    },
    "android": {
      "minSdkVersion": 24
    },
    "iOS": {
      "bundleVersion": "100"
    }
  }
}
```

| Key | Meaning |
| --- | --- |
| `targets` | `"all"` or an array: `deb`, `rpm`, `appimage`, `pacman`, `msi`, `nsis`, `app`, `dmg`. |
| `icon` | Icons used by the app and installers. Generate them with `tauri icon`. |
| `resources` | Extra files bundled under `$RESOURCE`; list or source→destination map. |
| `externalBin` | Sidecar binaries; files must carry the target-triple suffix. |
| `fileAssociations` | File types the app opens; drives Android intent filters and iOS `CFBundleDocumentTypes`. |
| `createUpdaterArtifacts` | `true` creates updater bundles and `.sig` files; `"v1Compatible"` for migrations. |
| `category` | macOS App Store category, e.g. `Utility`, `Productivity`. |
| `publisher` | Windows publisher; required to differ from the product name for the Microsoft Store. |
| `licenseFile` | License bundled by installers. |
| `windows.webviewInstallMode` | `downloadBootstrapper` (default), `embedBootstrapper`, `offlineInstaller`, `fixedRuntime`, `skip`. |
| `windows.wix` | MSI customization: template, fragments, language. |
| `windows.nsis` | NSIS customization: template, languages, hooks, `installMode`. |
| `macOS.minimumSystemVersion` | Minimum macOS version. |
| `macOS.entitlements` | Path to `Entitlements.plist`. |
| `macOS.frameworks` / `macOS.files` | Extra frameworks and bundle files. |
| `macOS.dmg` | DMG background, size, positions. |
| `linux.deb` / `linux.rpm` / `linux.appimage` / `linux.pacman` | Linux package-specific files, dependencies, scripts, and PKGBUILD options. |
| `android.versionCode` / `minSdkVersion` | Android version and API level. |
| `iOS.bundleVersion` | iOS `CFBundleVersion`. |

Bundle targets are host-dependent: MSI can only be built on Windows; DMG and `.app` only on macOS; Linux packages need WebKitGTK 4.1 development libraries. See `distribution.md`.

## Plugins

Plugin config lives under the plugin's name:

```json
{
  "plugins": {
    "updater": {
      "pubkey": "CONTENT OF THE PUBLIC KEY",
      "endpoints": ["https://releases.example.com/{{target}}/{{arch}}/{{current_version}}"]
    },
    "store": {}
  }
}
```

Plugin names match the crate's plugin name, not the full `tauri-plugin-` prefix. Unknown keys may be rejected by the plugin's config parser.

## CLI overrides and environment

Useful flags:

```sh
npm run tauri dev -- --config src-tauri/tauri.local.conf.json
npm run tauri build -- --no-bundle
npm run tauri bundle -- --bundles app,dmg
npm run tauri build -- --target aarch64-apple-darwin
npm run tauri build -- --debug
```

Tauri sets these environment variables for build scripts and frontend builds:

| Variable | Value |
| --- | --- |
| `TAURI_ENV_PLATFORM` | `windows`, `darwin`, `linux`, `android`, `ios` |
| `TAURI_ENV_ARCH` | `x86_64`, `aarch64`, `i686`, `armv7`, ... |
| `TAURI_ENV_FAMILY` | `windows`, `unix`, ... |
| `TAURI_ENV_PLATFORM_VERSION` | OS version |
| `TAURI_ENV_PLATFORM_TYPE` | `desktop` or `mobile` |
| `TAURI_ENV_TARGET_TRIPLE` | Rust target triple, e.g. `x86_64-pc-windows-msvc` |
| `TAURI_ENV_DEBUG` | `true` for debug builds |

Signing and updater variables (`TAURI_SIGNING_PRIVATE_KEY`, `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`, `APPLE_*`, `WINDOWS_*`) are covered in `distribution.md`.

## Pitfalls

| Problem | Cause | Fix |
| --- | --- | --- |
| Platform config loses resources | Arrays are replaced, not merged | Repeat the full array in the platform file. |
| JSON5/TOML file ignored | Feature not enabled on both crates | Add `config-json5`/`config-toml` to `tauri` and `tauri-build`. |
| `frontendDist` resolves to `d:` | Windows absolute path parsed as a URL | Use `\\\\?\\D:\\...` or a relative path. |
| Capability file ignored | `app.security.capabilities` is set and omits it | Add the identifier or remove the explicit list. |
| MSI build fails with `light.exe` | VBSCRIPT optional feature disabled | Enable Windows optional feature VBSCRIPT. |
| App Store rejects identifier | Underscore or `.app` suffix | Use reverse-DNS with letters, digits, hyphens, periods. |
| Build uses the wrong version | `version` absent, `Cargo.toml` version used | Set `version` in `tauri.conf.json`. |
