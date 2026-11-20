# Tauri 2 recipes

End-to-end recipes from the official Learn section: splashscreen, tray, custom titlebar, menus, sidecars, and mobile integration. Docs: <https://v2.tauri.app/learn/>.

## Contents

- [Splashscreen](#splashscreen)
- [System tray](#system-tray)
- [Custom titlebar](#custom-titlebar)
- [Window menus](#window-menus)
- [Node.js sidecar](#nodejs-sidecar)
- [Mobile file associations](#mobile-file-associations)
- [Mobile multi-window](#mobile-multi-window)
- [Deep links](#deep-links)

## Splashscreen

Create a hidden main window and a visible splash window, then show `main` when setup finishes.

```json
{
  "app": {
    "windows": [
      { "label": "main", "visible": false },
      { "label": "splashscreen", "url": "/splashscreen" }
    ]
  }
}
```

Track completion in managed state:

```rust
use std::sync::Mutex;
use tauri::{AppHandle, Manager, State};

struct SetupState {
    frontend_task: bool,
    backend_task: bool,
}

#[tauri::command]
async fn set_complete(
    app: AppHandle,
    state: State<'_, Mutex<SetupState>>,
    task: String,
) -> Result<(), ()> {
    let mut state = state.lock().unwrap();
    match task.as_str() {
        "frontend" => state.frontend_task = true,
        "backend" => state.backend_task = true,
        _ => return Err(()),
    }

    if state.frontend_task && state.backend_task {
        if let Some(splash) = app.get_webview_window("splashscreen") {
            let _ = splash.close();
        }
        if let Some(main) = app.get_webview_window("main") {
            let _ = main.show();
            let _ = main.set_focus();
        }
    }
    Ok(())
}
```

The frontend calls `invoke('set_complete', { task: 'frontend' })` after its setup, and Rust calls the same command for the backend. If either side can fail, add a timeout so the splashscreen does not hang forever.

A splashscreen is often unnecessary; a main window with an in-app loading indicator is usually better UX.

## System tray

Enable the Cargo feature:

```toml
[dependencies]
tauri = { version = "2", features = ["tray-icon"] }
```

Minimal tray with a quit item:

```rust
use tauri::{
    menu::{Menu, MenuItem},
    tray::TrayIconBuilder,
};

tauri::Builder::default().setup(|app| {
    let quit = MenuItem::with_id(app, "quit", "Quit", true, None::<&str>)?;
    let menu = Menu::with_items(app, &[&quit])?;

    TrayIconBuilder::new()
        .icon(app.default_window_icon().unwrap().clone())
        .menu(&menu)
        .show_menu_on_left_click(false)
        .on_menu_event(|app, event| {
            if event.id.as_ref() == "quit" {
                app.exit(0);
            }
        })
        .on_tray_icon_event(|tray, event| {
            if let tauri::tray::TrayIconEvent::Click {
                button: tauri::tray::MouseButton::Left,
                button_state: tauri::tray::MouseButtonState::Up,
                ..
            } = event
            {
                let app = tray.app_handle();
                if let Some(window) = app.get_webview_window("main") {
                    let _ = window.show();
                    let _ = window.set_focus();
                }
            }
        })
        .build(app)?;
    Ok(())
});
```

```json
{ "permissions": ["core:tray:default", "core:menu:default"] }
```

- Left-click menu behavior is controlled by `show_menu_on_left_click` / `menuOnLeftClick`.
- Tray mouse events are not emitted on Linux; the icon and context menu still work.
- Tray icons are desktop-only.
- Use `TrayIconBuilder::new()` once; keep the returned handle if you need to update the icon or tooltip later.

## Custom titlebar

Disable native decorations:

```json
{
  "app": {
    "windows": [
      { "label": "main", "decorations": false }
    ]
  }
}
```

Grant window controls:

```json
{
  "permissions": [
    "core:window:default",
    "core:window:allow-close",
    "core:window:allow-minimize",
    "core:window:allow-toggle-maximize",
    "core:window:allow-start-dragging"
  ]
}
```

HTML and CSS:

```html
<div class="titlebar" data-tauri-drag-region>
  <span>My App</span>
  <div class="controls">
    <button id="minimize">—</button>
    <button id="maximize">□</button>
    <button id="close">×</button>
  </div>
</div>
```

```css
.titlebar {
  height: 32px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  position: fixed;
  inset: 0 0 auto 0;
  background: #1f2937;
  color: white;
  user-select: none;
}
.titlebar .controls { display: flex; }
.titlebar button { width: 32px; height: 32px; }
```

```ts
import { getCurrentWindow } from '@tauri-apps/api/window';

const appWindow = getCurrentWindow();

document.getElementById('minimize')?.addEventListener('click', () => appWindow.minimize());
document.getElementById('maximize')?.addEventListener('click', () => appWindow.toggleMaximize());
document.getElementById('close')?.addEventListener('click', () => appWindow.close());
```

`data-tauri-drag-region` only applies to the element it is on; add it to child elements if they should also drag. Interactive children (buttons, inputs) should not have it.

On macOS, a fully custom titlebar loses native window management features. An alternative is `titleBarStyle: "Transparent"` plus a custom background color.

## Window menus

```rust
use tauri::menu::{MenuBuilder, SubmenuBuilder};

tauri::Builder::default().setup(|app| {
    let about = SubmenuBuilder::new(app, "About")
        .text("quit", "Quit")
        .build()?;

    let file = SubmenuBuilder::new(app, "File")
        .text("new", "New")
        .text("open", "Open")
        .text("save", "Save")
        .separator()
        .text("close", "Close")
        .build()?;

    let edit = SubmenuBuilder::new(app, "Edit")
        .text("undo", "Undo")
        .text("redo", "Redo")
        .build()?;

    let menu = MenuBuilder::new(app)
        .items(&[&about, &file, &edit])
        .build()?;

    app.set_menu(menu)?;

    app.on_menu_event(|_app, event| {
        match event.id().0.as_str() {
            "quit" => std::process::exit(0),
            "new" => { /* ... */ }
            "open" => { /* ... */ }
            _ => {}
        }
    });

    Ok(())
});
```

JavaScript equivalent:

```ts
import { Menu, Submenu } from '@tauri-apps/api/menu';

const about = await Submenu.new({
  text: 'About',
  items: [{ id: 'quit', text: 'Quit', action: () => console.log('quit') }],
});

const file = await Submenu.new({
  text: 'File',
  items: [
    { id: 'new', text: 'New', action: () => console.log('new') },
    { id: 'open', text: 'Open', action: () => console.log('open') },
  ],
});

const menu = await Menu.new({ items: [about, file] });
await menu.setAsAppMenu();
```

- On macOS, top-level menu entries **must** be submenus; the first submenu becomes the application menu.
- Menu items can be basic, check, icon, separator, or submenu items.
- `on_menu_event` / `action` handles custom item IDs.
- Menus are desktop-only. Grant `core:menu:default` if the frontend creates menus.

## Node.js sidecar

Bundle a Node.js executable or script as a sidecar:

```json
{
  "bundle": {
    "externalBin": ["binaries/my-node-app"]
  }
}
```

The file on disk must include the target triple: `binaries/my-node-app-x86_64-pc-windows-msvc.exe`. Get the current tuple with `rustc --print host-tuple`.

If the sidecar is a script, compile it to a single executable with a tool such as `pkg`, `nexe`, or Node's SEA, then rename it with the triple. You cannot rely on a system Node installation.

Run from Rust:

```rust
use tauri_plugin_shell::ShellExt;

let command = app.shell().sidecar("my-node-app")?.args(["--port", "3000"]);
let (mut rx, mut child) = command.spawn()?;

tauri::async_runtime::spawn(async move {
    while let Some(event) = rx.recv().await {
        if let tauri_plugin_shell::process::CommandEvent::Stdout(line) = event {
            println!("{}", String::from_utf8_lossy(&line));
        }
    }
});
```

Run from JavaScript:

```ts
import { Command } from '@tauri-apps/plugin-shell';

const command = Command.sidecar('binaries/my-node-app', ['--port', '3000']);
const output = await command.execute();
console.log(output.stdout);
```

Grant it:

```json
{
  "identifier": "shell:allow-execute",
  "allow": [
    {
      "name": "binaries/my-node-app",
      "sidecar": true,
      "args": ["--port", { "validator": "\\d+" }]
    }
  ]
}
```

Use `shell:allow-spawn` instead if you call `spawn()` rather than `execute()`. The JavaScript `sidecar()` string must match `externalBin`; the Rust `sidecar()` call takes only the binary name.

For a long-running local server, the Rust-only `localhost` plugin can serve app assets over localhost instead of the custom protocol; review its security caution before using it.

## Mobile file associations

Declare file types:

```json
{
  "bundle": {
    "fileAssociations": [
      {
        "ext": ["mydata"],
        "mimeType": "application/octet-stream",
        "exportedType": {
          "identifier": "com.example.myapp.mydata",
          "conformsTo": ["public.data"]
        }
      }
    ]
  }
}
```

Handle files opened while running and at cold start:

```rust
use std::sync::Mutex;
use tauri::{Emitter, Manager};

struct OpenedUrls(Mutex<Vec<tauri::Url>>);

#[tauri::command]
fn opened_urls(app: tauri::AppHandle) -> Vec<tauri::Url> {
    app.state::<OpenedUrls>().0.lock().unwrap().clone()
}

tauri::Builder::default()
    .manage(OpenedUrls(Mutex::new(vec![])))
    .invoke_handler(tauri::generate_handler![opened_urls])
    .build(tauri::generate_context!())?
    .run(|app, event| {
        #[cfg(any(target_os = "macos", target_os = "ios", target_os = "android"))]
        if let tauri::RunEvent::Opened { urls } = event {
            app.state::<OpenedUrls>().0.lock().unwrap().extend(urls.clone());
            let _ = app.emit("opened", urls);
        }
    });
```

The frontend calls `invoke('opened_urls')` on startup and listens for the `opened` event for warm starts. File associations are supported on macOS, iOS, and Android; desktop Windows/Linux setup is installer-specific.

## Mobile multi-window

Add the capability:

```json
{ "permissions": ["core:default", "core:webview:allow-create-webview-window"] }
```

Create a window with a platform-specific activity or scene:

```ts
import { WebviewWindow } from '@tauri-apps/api/webviewWindow';

const window = new WebviewWindow(`detail-${id}`, {
  url: `detail/${id}`,
  activityName: 'DetailActivity',
});

window.once('tauri://created', () => {});
window.once('tauri://error', (error) => console.error(error));
```

Android uses Activity Embedding and split rules in `src-tauri/gen/android`; iOS uses `UIScene` and `RunEvent::SceneRequested`. Both need platform project changes beyond the config. Multi-window requires Android 12L+ / iOS 13+ and is primarily a tablet experience; see the official guide for the full manifest and plist setup.

## Deep links

Configure schemes:

```json
{
  "plugins": {
    "deep-link": {
      "desktop": { "schemes": ["myapp"] },
      "mobile": [{ "scheme": ["myapp"], "appLink": false }]
    }
  }
}
```

```json
{ "permissions": ["deep-link:default"] }
```

Desktop needs the scheme registered by the installer; mobile needs Android intent filters or iOS associated domains. Use `onOpenUrl` from `@tauri-apps/plugin-deep-link` to receive URLs. Combine with the single-instance plugin so a second launch forwards the URL to the running app.
