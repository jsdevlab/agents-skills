# Packaging, signing, and distribution

Tauri builds the app and its platform bundles in one command. This reference covers installers, signing, app stores, and the updater. Official docs: <https://v2.tauri.app/distribute/>.

## Contents

- [Build and bundle](#build-and-bundle)
- [Versioning and icons](#versioning-and-icons)
- [Windows](#windows)
- [macOS](#macos)
- [Linux](#linux)
- [Android](#android)
- [iOS](#ios)
- [App stores](#app-stores)
- [Updater](#updater)
- [Release checklist](#release-checklist)

## Build and bundle

```sh
npm run tauri build                  # compile + bundle for the host
npm run tauri build -- --no-bundle   # compile only
npm run tauri bundle -- --bundles app,dmg
npm run tauri build -- --target aarch64-apple-darwin
npm run tauri build -- --debug       # debuggable build
```

`bundle.targets` is `"all"` by default, which builds every format supported by the host. To limit it:

```json
{
  "bundle": {
    "targets": ["nsis", "msi"]
  }
}
```

| Platform | Targets | Output |
| --- | --- | --- |
| Windows | `msi`, `nsis` | `src-tauri/target/release/bundle/msi/*.msi`, `.../nsis/*-setup.exe` |
| macOS | `app`, `dmg` | `.../bundle/macos/*.app`, `.../dmg/*.dmg` |
| Linux | `deb`, `rpm`, `appimage`, `pacman` | `.../bundle/deb/*.deb`, `.../rpm/*.rpm`, `.../appimage/*.AppImage`, `.../bundle/pacman/` |
| Android | AAB/APK | `src-tauri/gen/android/app/build/outputs/bundle/universalRelease/*.aab` |
| iOS | IPA | `src-tauri/gen/apple/build/arm64/*.ipa` |

Host constraints: MSI requires Windows and the VBSCRIPT optional feature; `.app`/DMG require macOS; Linux packages need WebKitGTK 4.1 development headers and should be built on Ubuntu 22.04 or Debian 12 for glibc compatibility.

## Versioning and icons

Set the version in `tauri.conf.json > version` (recommended) or fall back to `src-tauri/Cargo.toml`.

- Android `versionCode` defaults to `major * 1_000_000 + minor * 1_000 + patch`; override with `bundle.android.versionCode`.
- iOS/macOS `CFBundleVersion` defaults to the app version; override with `bundle.iOS.bundleVersion` / `bundle.macOS.bundleVersion`.
- File-association metadata is generated from `bundle.fileAssociations`.

Generate every icon from one square PNG or SVG:

```sh
npm run tauri icon ./app-icon.png
npm run tauri icon ./app-icon.png -- --ios-color '#ffffff'
```

Output goes to `src-tauri/icons` and the mobile projects. Commit the generated icons.

## Windows

Two installer formats:

- **NSIS** (`-setup.exe`): per-user by default, supports `installMode: "perMachine"` or `"both"`, cross-compilable from Linux/macOS with caveats.
- **MSI** (`WiX`): Windows-only, can only be built on Windows, requires VBSCRIPT.

```json
{
  "bundle": {
    "windows": {
      "webviewInstallMode": { "type": "downloadBootstrapper" },
      "nsis": {
        "installMode": "perMachine",
        "languages": ["English", "SimpChinese"],
        "displayLanguageSelector": true
      },
      "wix": {
        "language": ["en-US", "zh-CN"]
      }
    }
  }
}
```

WebView2 install modes:

| Mode | Internet needed | Extra size | Use |
| --- | --- | --- | --- |
| `downloadBootstrapper` | Yes | 0 MB | Default; smallest installer. |
| `embedBootstrapper` | Yes | ~1.8 MB | Better Windows 7/MSI compatibility. |
| `offlineInstaller` | No | ~127 MB | Offline environments and Microsoft Store. |
| `fixedRuntime` | No | ~180 MB | Pin a WebView2 runtime version. |
| `skip` | No | 0 MB | Only when WebView2 is guaranteed; not recommended. |

Signing options:

- Certificate thumbprint:
  ```json
  {
    "bundle": {
      "windows": {
        "certificateThumbprint": "A1B1...",
        "digestAlgorithm": "sha256",
        "timestampUrl": "http://timestamp.example.com"
      }
    }
  }
  ```
- Azure Key Vault / custom tool:
  ```json
  {
    "bundle": {
      "windows": {
        "signCommand": "relic sign --file %1 --key azure --config relic.conf"
      }
    }
  }
  ```
- Azure Artifact Signing (formerly Trusted Signing):
  ```json
  {
    "bundle": {
      "windows": {
        "signCommand": "artifact-signing-cli -e https://wus2.codesigning.azure.net -a MyAccount -c MyProfile -d MyApp %1"
      }
    }
  }
  ```

SmartScreen reputation builds over time even with an EV certificate; signing every release with the same certificate helps. Signing does not guarantee an immediate warning-free download.

Microsoft Store requires an offline WebView2 installer, silent installation (`/S` for NSIS, `/quiet` for MSI), and a `publisher` that differs from `productName`.

## macOS

Bundle formats:

- `.app` — application bundle; required for the App Store.
- `.dmg` — drag-to-Applications installer for direct download.

```json
{
  "bundle": {
    "macOS": {
      "minimumSystemVersion": "12.0",
      "entitlements": "./Entitlements.plist",
      "frameworks": ["CoreAudio"],
      "files": {
        "embedded.provisionprofile": "./profile.provisionprofile"
      },
      "dmg": {
        "background": "./images/dmg-background.png",
        "windowSize": { "width": 800, "height": 600 }
      }
    }
  }
}
```

Signing and notarization:

| Variable | Purpose |
| --- | --- |
| `APPLE_CERTIFICATE` | Base64 `.p12` signing certificate. |
| `APPLE_CERTIFICATE_PASSWORD` | Password for the `.p12`. |
| `APPLE_SIGNING_IDENTITY` | Keychain identity, e.g. `Developer ID Application: ...`. |
| `APPLE_API_ISSUER` / `APPLE_API_KEY` / `APPLE_API_KEY_PATH` | App Store Connect API key for notarization/upload. |
| `APPLE_ID` / `APPLE_PASSWORD` / `APPLE_TEAM_ID` | Apple ID notarization alternative. |

- **Developer ID Application** is for direct distribution; notarization is required.
- **Apple Distribution** is for the App Store; provisioning profile + sandbox entitlement required.
- Ad-hoc signing (`"signingIdentity": "-"`) avoids the "damaged" warning on Apple Silicon but does not bypass Gatekeeper's unknown-developer prompt.
- `xcrun notarytool` is used under the hood; builds can use `--skip-stapling` for an initial pass.

DMG icon positions are not applied on CI runners due to a known issue.

## Linux

Build on the oldest distribution you support. Ubuntu 22.04 and Debian 12 provide WebKitGTK 4.1 packages and a suitable glibc baseline. Newer build hosts raise the minimum glibc and can produce `GLIBC_2.xx not found` on older targets.

```json
{
  "bundle": {
    "linux": {
      "deb": {
        "depends": ["libwebkit2gtk-4.1-0"],
        "files": { "/usr/share/myapp/README.md": "../README.md" }
      },
      "rpm": {
        "release": "1",
        "epoch": 0,
        "files": {},
        "postInstallScript": "./scripts/postinstall.sh"
      },
      "appimage": {
        "bundleMediaFramework": true,
        "files": { "/usr/share/myapp/data": "../data/" }
      }
    }
  }
}
```

- **AppImage**: portable, larger; `bundleMediaFramework` bundles GStreamer for audio/video and is fully supported only on Ubuntu build hosts.
- **Debian**: depends on `libwebkit2gtk-4.1-0`, `libgtk-3-0`, and appindicator when the tray is used.
- **RPM**: supports pre/post install/remove scripts, `depends`, `provides`, `conflicts`, `obsoletes`, `files`, `desktopTemplate`, `epoch`.
- **Pacman**: Tauri can emit a PKGBUILD-based package (`pacman` target); AUR publishing itself remains an external `PKGBUILD` workflow.
- **Snapcraft**: external `snapcraft.yaml`; builds a `.deb` with Tauri, extracts it, and stages runtime libraries.
- **Flathub**: external Flatpak manifest + generated `node-sources.json` and `cargo-sources.json`.
- **AUR**: external `PKGBUILD`; depend on `webkit2gtk-4.1` and related libraries.

AppImage signing uses GPG:

```sh
SIGN=1 SIGN_KEY=<key-id> APPIMAGETOOL_SIGN_PASSPHRASE=<passphrase> \
APPIMAGETOOL_FORCE_SIGN=1 npm run tauri build -- --bundles appimage
```

The signature is embedded but not automatically verified; publish the key ID on an authenticated channel.

## Android

```sh
npm run tauri android build -- --aab
npm run tauri android build -- --apk
npm run tauri android build -- --aab --target aarch64 --target armv7
npm run tauri android build -- --apk --split-per-abi
```

- Minimum supported Android is 7.0 (SDK 24); override with `bundle.android.minSdkVersion`.
- AAB is the Play Store format; APK is for testing or direct distribution.
- Signing: create `upload-keystore.jks`, then `src-tauri/gen/android/keystore.properties`:

```properties
password=<password>
keyAlias=upload
storeFile=/path/to/upload-keystore.jks
```

Configure `signingConfigs.release` in `src-tauri/gen/android/app/build.gradle.kts` and use it in `buildTypes.release`. In CI, write `keystore.properties` from secrets and decode the base64 keystore.

- First Play Console upload must be manual so Google can verify the signature and bundle ID.
- Android 16 KB page alignment is required for new submissions; NDK 28+ handles it.

## iOS

```sh
npm run tauri ios build -- --export-method app-store-connect
npm run tauri ios build -- --open   # open the Xcode project
```

Signing:

- **Automatic** via Xcode; CI uses an App Store Connect API key (`APPLE_API_ISSUER`, `APPLE_API_KEY`, `APPLE_API_KEY_PATH`).
- **Manual** via `IOS_CERTIFICATE` (base64 `.p12`), `IOS_CERTIFICATE_PASSWORD`, and `IOS_MOBILE_PROVISION` (base64 provisioning profile).
- The bundle identifier must match the App ID and `tauri.conf.json > identifier`.
- The IPA is written to `src-tauri/gen/apple/build/arm64/<AppName>.ipa`.

## App stores

| Store | Build | Key requirements |
| --- | --- | --- |
| Apple App Store (macOS) | `tauri build --bundles app --target universal-apple-darwin`, then `productbuild` → `.pkg` | Paid Apple Developer account, provisioning profile, App Sandbox entitlement, `category`, signed PKG, `altool` upload. |
| Apple App Store (iOS) | `tauri ios build --export-method app-store-connect` | App ID, provisioning profile, distribution certificate, IPA, `altool`. |
| Google Play | `tauri android build --aab` | Play Console account, signed AAB, `versionCode`, manual first upload. |
| Microsoft Store | NSIS/MSI with `offlineInstaller`, silent install | Partner Center registration, `/S` (NSIS) or `/quiet` (MSI), publisher ≠ product name, code signing. |

## Updater

The updater is a plugin with mandatory signature verification. It cannot be disabled.

### Keys

```sh
npm run tauri signer generate -- -w ~/.tauri/myapp.key
```

- Put the **public** key content in `plugins.updater.pubkey` (content, not a file path).
- Keep the private key offline; losing it prevents updating existing installations.
- Build with `TAURI_SIGNING_PRIVATE_KEY` (path or content) and optional `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`. `.env` files are not read by the bundler.

### Artifacts

```json
{
  "bundle": {
    "createUpdaterArtifacts": true
  }
}
```

This produces updater bundles and `.sig` files:

- Linux: `.AppImage` + `.AppImage.sig`
- macOS: `.app.tar.gz` + `.sig`
- Windows: `.msi`/`-setup.exe` + `.sig`

Use `"v1Compatible"` only when migrating existing v1 installations.

### Endpoints

```json
{
  "plugins": {
    "updater": {
      "pubkey": "CONTENT OF PUBLIC KEY",
      "endpoints": [
        "https://releases.example.com/{{target}}/{{arch}}/{{current_version}}",
        "https://github.com/owner/repo/releases/latest/download/latest.json"
      ]
    }
  }
}
```

Dynamic variables: `{{current_version}}`, `{{target}}` (`linux`, `windows`, `darwin`), `{{arch}}` (`x86_64`, `i686`, `aarch64`, `armv7`). HTTPS is enforced in production; `dangerousInsecureTransportProtocol` allows HTTP and should not ship.

Static JSON manifest:

```json
{
  "version": "1.2.0",
  "notes": "Fixes and improvements",
  "pub_date": "2026-11-20T12:00:00Z",
  "platforms": {
    "linux-x86_64": { "signature": "<contents of .sig>", "url": "https://..." },
    "windows-x86_64": { "signature": "<contents of .sig>", "url": "https://..." },
    "darwin-aarch64": { "signature": "<contents of .sig>", "url": "https://..." }
  }
}
```

Dynamic server: return `204 No Content` when current, or `200` with `version`, `url`, `signature`, optional `notes`/`pub_date`.

### Install

```ts
import { check } from '@tauri-apps/plugin-updater';
import { relaunch } from '@tauri-apps/plugin-process';

const update = await check();
if (update) {
  await update.downloadAndInstall((event) => {
    if (event.event === 'Progress') console.log(event.data.chunkLength);
  });
  await relaunch();
}
```

```json
{ "permissions": ["updater:default", "process:allow-restart"] }
```

Windows install modes: `passive` (default, progress bar, no interaction), `basicUi` (requires interaction), `quiet` (no UI; cannot elevate). Windows exits the app before installing; use `on_before_exit` for cleanup.

## Release checklist

- [ ] Version bumped in `tauri.conf.json`; Android `versionCode` / iOS `bundleVersion` correct.
- [ ] `tauri icon` regenerated; icons committed.
- [ ] `cargo test`, frontend tests, and WebDriver smoke tests pass.
- [ ] `tauri build` succeeds on each target OS (or CI matrix).
- [ ] Windows and macOS binaries signed; macOS notarized if distributed outside the App Store.
- [ ] Linux bundles built on the oldest supported baseline.
- [ ] Updater artifacts signed; public key matches the app; manifest URLs reachable over HTTPS.
- [ ] Installer permissions, file associations, and deep links verified on a clean machine.
- [ ] Checksums/signatures published next to downloads.
