# Linux support status (experimental)

This document records the current state of Linux bring-up work in this working tree. Linux was previously unsupported; these changes are **local/uncommitted experiments** aimed at making the app launch and render its privileged UI on Linux.

> **Scope note:** Changes were made with the explicit goal of getting Zephium working on Linux. They do not yet represent full, production-qualified Linux support across all features or release/packaging paths.

## Current status

- `zephium-desktop` compiles on Linux (`cargo check -p zephium-desktop --features tauri/custom-protocol`).
- The frontend can be built into `frame/dist` and served via Tauri’s bundled asset/custom-protocol path.
- A debug build launched on Linux and showed:
  - native bootstrap/session startup completed,
  - the main document initialized and window was shown,
  - the privileged UI DOM/events were present in WebKit Inspector.
- A blank/invisible main window was traced to Linux GTK composition: the privileged chrome `WebKitWebView`, reparented into a `gtk::Fixed`, could retain a `1x1`/unmapped allocation and not paint. An explicit allocation fix was added; after that the chrome was observed as allocated/mapped (e.g. `1200x800` then larger) and the user reported diagnosing the UI via Inspector.

## Modified files (uncommitted)

| File | Purpose |
|---|---|
| `desktop/src/lib.rs` | Removed the hard Linux startup exit so Linux can run. (A temporary debug `open_devtools()` hook was added during diagnosis and then removed.) |
| `desktop/src/platform/linux.rs` | Linux GTK privileged-window composition: keep the Tauri chrome `WebKitWebView` explicitly moved/sized/allocated inside the `gtk::Fixed` container so it fills the window and paints. |
| `desktop/src/launcher_trigger.rs` | Linux compatibility for builds without `tauri-plugin-global-shortcut`: provide unsupported/no-op behavior on Linux and gate shortcut-plugin-dependent tests appropriately; Linux reports launcher shortcuts as unsupported where the global-shortcut backend is unavailable. |
| `desktop/src/page_assist.rs` | Keep explicit port imports; gate native page-assist `product`/`handoff` to macOS/Windows (`work-product` path depends on macOS/Windows engine/page-reader/agentic-browser and desktop work-composition wiring). Linux uses the existing no-op/stub-compatible path via the module’s service handling. |
| `vendor/wry/src/webkitgtk/web_context.rs` | **Vendored fork mitigation** for Linux/WebKitGTK custom-protocol bursts: add CORS headers (`Access-Control-Allow-Origin: *`, `Access-Control-Expose-Headers: Tauri-Response`) to overflow/empty-status scheme responses so failures are diagnosable; raise the custom-protocol in-flight admission ceiling (`CUSTOM_PROTOCOL_IN_FLIGHT_LIMIT * 4`) as a mitigation. This is not the full queued `WaitList` behavior used on wkwebview. |

`desktop/tauri.linux.conf.json` is present in the tree (GTK app id, main window initially `create: false`; window `decorations: false`, `transparent: true`). Review Linux window/compositor behavior with the GTK composition changes.

## How to build and run (Linux, local/dev)

These commands were used during bring-up. They are **not** a substitute for Zephium’s normal CI/native qualification process.

```sh
# From repo root
pnpm install --frozen-lockfile
pnpm --dir frame run build

# Build desktop with bundled frontend/custom protocol
cargo build -p zephium-desktop --features tauri/custom-protocol

# Run (environment used during debugging on this host)
DISPLAY=:0 GDK_BACKEND=x11 WEBKIT_DISABLE_DMABUF_RENDERER=1 \
  ./target/debug/zephium-desktop
```

Notes:
- Without `tauri/custom-protocol` and a built `frame/dist`, startup can fail with a privileged navigation policy error pointing at `http://localhost:1420/...` (dev URL) instead of bundled assets.
- `WEBKIT_DISABLE_DMABUF_RENDERER=1` and explicit `DISPLAY`/`GDK_BACKEND` were used for this host’s debugging session; they may not be required everywhere.

## Diagnosis notes (what was observed)

- Early symptom: window opened but appeared empty/blank.
- WebKit Inspector showed onboarding DOM/events while the actual window showed nothing.
- `frame/dist/browser.html` existed and referenced `/assets/*`.
- Console errors seen during diagnosis included IPC fetch/CORS symptoms involving `ipc://localhost/*` (e.g. `setting_get`) and `tauri://localhost` origin context; after vendored Wry webkitgtk changes, these specific overflow-masking/CORS symptoms were addressed for the tested case, but the root visible-blank issue was the GTK `Fixed` allocation/mapping of the chrome webview.
- Wayland/XDG global-shortcut portal messages like `app.zephium` app info not found were observed during uninstalled runs; the app reported focused-window fallback behavior. This was not identified as the cause of the blank window.

## Known limitations / follow-ups

1. **Vendored `wry` fork change requires proper fork handling.** `vendor/wry/src/webkitgtk/web_context.rs` was modified as a mitigation. Before treating this as accepted, follow the repository’s fork/provenance rules (`FORK.toml`/`UPSTREAM.md`/`REBASE.md`/reviewed patch process where applicable) and validate it does not regress macOS/Windows behavior.
2. **WebKitGTK custom-protocol admission is still incomplete.** Linux/webkitgtk currently rejects when the in-flight limit is exhausted; the mitigation raises the ceiling to `CUSTOM_PROTOCOL_IN_FLIGHT_LIMIT * 4` and adds CORS headers on overflow responses. The proper fix is to port/adapt `WaitList`-style queuing (similar to wkwebview behavior) so startup/request bursts wait instead of failing.
3. **Page Assist native service is not enabled on Linux.** The current implementation is gated to macOS/Windows due to engine page-reader/agentic-browser and desktop work-composition dependencies.
4. **Global shortcuts / launcher trigger parity.** Linux has no `tauri-plugin-global-shortcut` path in the same form used by the launcher trigger module; launcher shortcuts are treated as unsupported in that module. Existing Linux global-shortcut/portal/X11 handling in `desktop/src/lib.rs` and related code should be reviewed separately for full product behavior.
5. **Linux engine/platform parity is incomplete.** `zephium-engine` has Linux platform modules, but feature parity and native behavior should be validated against the same ownership boundaries used on macOS/Windows (content, input, permissions, downloads, devtools, media, etc.).
6. **Window/compositor presentation.** `tauri.linux.conf.json` uses transparent/decorations-off settings; verify behavior across X11/Wayland/compositors with the GTK composition fix.
7. **Packaging/app identity.** Uninstalled runs may not satisfy portal/desktop app-id expectations (e.g. `app.zephium`). Validate installed/package metadata, desktop templates, and portal integration.
8. **No hosted/native Linux qualification in this report.** The checks performed here were local Linux bring-up checks. Zephium’s normal process requires reproducing affected CI jobs and verifying affected native paths on supported platforms; macOS/Windows impact from shared/vendored changes must be assessed before merge.

## Verification performed (local)

- `cargo check -p zephium-desktop` / `cargo check -p zephium-desktop --features tauri/custom-protocol` — passed (warnings present).
- `cargo build --release -p zephium-desktop` was previously reported successful during bring-up; later debug rebuilds were used after Linux/Wry fixes.
- Frontend built with `pnpm install --frozen-lockfile` and `pnpm --dir frame run build` (local store adjustments were used in the session environment).
- Manual Linux debug runs observed startup completion and, after the GTK allocation fix, a mapped/sized chrome webview.

## Safety/cleanup reminders

- Do not use an installed/personal Zephium profile for development checks.
- Revert or properly justify temporary debugging aids before merge (none intended to remain: devtools auto-open hook removed; diagnostic prints removed).
- Re-audit security-sensitive boundaries touched by Linux composition and vendored protocol handling (privileged webview hardening, custom protocol admission, CSP/navigation policy, exposed IPC surface).
