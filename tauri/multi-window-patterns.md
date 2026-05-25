---
date: 2026-05-25
source: nicermd Tauri shell (versions 0.1.11–0.1.22)
---

# Tauri multi-window patterns

Patterns worked out while building a true multi-window Tauri 2 shell.
Most of these are non-obvious and cost a release each to discover.
Posted so the next project skips that.

## Window labels are your identity key

Tauri windows have a `label` string. Most per-window state should be
keyed on this label, not generated at runtime. Read it from JS via:

```ts
function getWindowLabel(): string {
  return (window as any).__TAURI_INTERNALS__?.metadata?.currentWindow?.label
    ?? 'main'
}
```

### Why this matters

- `localStorage` is shared across same-origin webviews. Without a
  label prefix, two windows clobber each other's persistence.
- `tauri-plugin-window-state` auto-saves geometry per label — works
  perfectly if your labels are stable, breaks if you regenerate
  labels per launch.
- `app.emit_to(label, …)` routes menu events to the focused window
  only.

### Pattern: per-window localStorage

```ts
const label = getWindowLabel()
localStorage.setItem(`myapp:mode:${label}`, mode)
```

## Spawning new windows from JS without races

The naive flow has a race: spawn a window, then emit a payload to it
after build. But the new window may finish booting and consume the
empty default before the emit lands.

**Pattern: pre-assign label, stash payload on Rust side, then build.**

```rust
#[tauri::command]
async fn spawn_window_with_payload(
    app: AppHandle,
    payload: serde_json::Value,
) -> Result<String, String> {
    let label = format!("win-{}", next_window_id());
    PENDING_PAYLOADS.lock().unwrap().insert(label.clone(), payload);
    WebviewWindowBuilder::new(&app, &label, WebviewUrl::default())
        .title("My App")
        .build()
        .map_err(|e| e.to_string())?;
    Ok(label)
}

#[tauri::command]
fn drain_window_payload(label: String) -> Option<serde_json::Value> {
    PENDING_PAYLOADS.lock().unwrap().remove(&label)
}
```

JS side drains on boot:

```ts
const payload = await invoke('drain_window_payload', { label: getWindowLabel() })
if (payload) applyPayload(payload)
```

The new window's `drain_window_payload` always finds the payload
because the insert happens before the build.

## Deep links must dispatch on the main thread

On macOS, `WebviewWindowBuilder::build` *requires* the main thread.
`tauri-plugin-deep-link`'s `app.deep_link().on_open_url()` callback
fires off-main — calling build from it hangs or crashes silently.

**Three options, in order of robustness:**

1. **Handle deep links in JS via `onOpenUrl`, then invoke an IPC
   command to spawn the window.** IPC commands run on main. This
   was the reliable path for us.
2. **`RunEvent::Opened`** in `Builder::run` runs on main; route
   `Event::Opened { urls }` to spawn logic there.
3. **Wrap in `app.run_on_main_thread(...)`** if you must call from
   the off-thread callback. In practice this dispatched silently
   for us; option 1 was more reliable.

### Plugin scoping gotcha

`tauri-plugin-deep-link` emits its events globally — every window
gets the callback. If you handle in JS, restrict to one window:

```ts
if (label === 'main') {
  await onOpenUrl((urls) => { /* handle */ })
}
```

Otherwise deep links spawn N windows for N existing windows.

## Capability scoping

```jsonc
// capabilities/default.json
{
  "windows": ["*"]  // new windows inherit all permissions
}
```

Without `"*"`, new windows can't drag, save, dialog, or use most
plugins. Default is the first window only — silently.

## Filesystem scope is runtime-only

`tauri-plugin-fs` 0.1.6+ requires per-path scope authorisation at
runtime. Static `path: "$HOME/**"` in capabilities doesn't grant
read access in practice — dialog-returned paths auto-add to the
scope, but those grants don't survive restart.

**Pattern for session restore: an `allow_fs_path` Tauri command:**

```rust
#[tauri::command]
async fn allow_fs_path(
    app: AppHandle,
    path: String,
) -> Result<(), String> {
    app.fs_scope()
        .allow_file(&path)
        .map_err(|e| e.to_string())
}
```

Call it before any `readTextFile` on a restored path.

## Session restore: two-file pattern

Saving session state once-per-change to a single file looks fine
until Cmd+Q. The close cascade can overwrite the session file
mid-flight (window N's destroy fires after window 1 cleared the
state). Result: relaunch finds an empty file and spawns only the
main window.

**Pattern: two files.**

- `session-live.json` — written on every state change.
- `session-at-quit.json` — written once by the quit handler,
  before the close cascade fires.

On startup:

1. Try `session-at-quit.json` first. If present, consume + delete it.
2. Fall back to `session-live.json`.

The quit snapshot survives even if the cascade trashes the live file.

## Bring All to Front needs a custom handler

The Tauri 2 / muda predefined `bring_all_to_front` menu item silently
no-ops for windows with `TitleBarStyle::Overlay`. Those windows
aren't in `NSApp.arrangeInFront`'s sweep.

```rust
fn bring_all_to_front(app: &AppHandle) {
    for (_label, window) in app.webview_windows() {
        let _ = window.unminimize();
        let _ = window.show();
        let _ = window.set_focus();
    }
}
```

Wire to a custom menu item via `menu_event` handler.

## Quit needs a custom handler too (for cascade ordering)

`PredefinedMenuItem::quit` calls `app.exit()` directly, bypassing
your per-window `CloseRequested` handlers. If you have dirty-discard
prompts or pre-cascade snapshots to write, replace the predefined
quit with a custom one that fires `WindowEvent::CloseRequested` per
window in your chosen order.

## Dirty-aware Open-With routing

When the user opens a file via macOS Open-With while the focused
window has unsaved changes, replacing the doc in that window would
lose work.

Track dirty state per window in Rust (synced from JS via a
`set_window_dirty` command); on `RunEvent::Opened`, route the file
to either the focused clean window (replace) or a fresh spawn
(preserve).

## Window-state plugin caveats

`tauri-plugin-window-state` auto-saves position/size on window close
and auto-restores on next launch. Trip-wires:

- It only restores geometry for windows whose labels you spawn. For
  dynamic labels (`win-1`, `win-2`, …) you also need to spawn the
  windows on launch from your own session-restore code — the plugin
  handles geometry, not existence.
- Bump your label counter past the highest restored label so new
  spawns don't collide.

## Context menus — use the native popup

Custom right-click menus in WKWebView look stylistically jarring vs
system menus. For polish, use `@tauri-apps/api/menu`'s `Menu.popup()`
for native-look context menus:

```ts
const { Menu, MenuItem } = await import('@tauri-apps/api/menu')
const item = await MenuItem.new({
  text: 'Open Link in New Window',
  action: () => openInNewWindow(target),
})
const menu = await Menu.new({ items: [item] })
await menu.popup()
```

## What still bites

- `ask()`-based dialogs accept Enter for the default button but
  Escape doesn't reliably cancel and Left/Right arrows don't move
  the highlighted button on macOS. Limitation of
  `tauri-plugin-dialog`'s NSAlert wrapping vs the native NSAlert.
  Live with click-only for now.
- A reported "third window crash" symptom was unreproducible across
  16 multi-window-heavy releases; left an open ticket and dropped
  it after no recurrence. Worth being suspicious of new Tauri
  versions on this front.
