---
name: flutter-desktop-platform
description: Handle Windows and Linux desktop specifics in Flutter — window management, file system, native integration, platform channels, per-platform behaviour differences. Do NOT use for packaging and distribution — that's flutter-release.
---

# flutter-desktop-platform

## Owns
- Desktop platform integration, window behaviour, file system access, native channels.
- Platform divergence between Android, Windows, and Linux at runtime.

## Does not own
- Packaging, signing, installers, distribution → `flutter-release`.
- Adaptive layout and desktop-appropriate UI → `flutter-ui-craft`.
- App structure → `flutter-architecture`.

---

## 1. Check plugin support before you design the feature

**Flutter desktop is less mature than mobile, and plugin support is uneven — Linux especially.** A
package with 2,000 likes may have Android and iOS only, or a Linux implementation that is a stub.

**Verify on pub.dev before designing anything around it:** open the package page and read the
platform table, then check the Linux implementation actually exists in the repo rather than trusting
the badge. Then build a throwaway spike that calls it on Windows *and* Linux before writing the
feature.

**Discovering a plugin is Android-only after building the UI is the most common wasted week here.**
Categories where desktop support is routinely missing or partial: biometrics, push notifications,
in-app purchase, camera, background execution, secure storage (backed by very different primitives
per platform), and anything touching system permissions.

When there is no desktop implementation, the options are: a different package, a platform channel you
write, a degraded-but-honest desktop path, or dropping the feature on desktop. Pick deliberately and
tell the user which.

---

## 2. Window management

Wire this up on the first day of desktop support, not at the end.

- **Minimum size**, set to the narrowest width the compact tier actually works at, so the layout
  cannot be dragged into a broken state.
- **Remember size, position, and maximised state across launches.** Validate the restored rectangle
  is still on a connected display — a window restored to an unplugged monitor is invisible, and the
  app simply looks broken.
- **Open at a desktop-appropriate default size.** A fixed phone-sized window reads as a port instantly.
- A **custom title bar** is a real cost, not a free one: you then own dragging, double-click-to-
  maximise, snap regions, the system menu, and the Windows/Linux button-order difference.
- Confirm-on-close for unsaved work (`window_manager`'s prevent-close plus your own dirty check).
  Multi-window support is still limited — validate it early if the product needs it.

---

## 3. File system

**`path_provider` gives the correct per-platform directory. Never write next to the executable** — on
Windows that is often under `Program Files` and unwritable without elevation; on Linux it may be
read-only or shared between users.

| Purpose | Windows | Linux |
|---|---|---|
| App config/state | `%APPDATA%\<app>` | `$XDG_CONFIG_HOME` or `~/.config/<app>` |
| Cache | `%LOCALAPPDATA%\<app>\Cache` | `$XDG_CACHE_HOME` or `~/.cache/<app>` |
| User documents | `%USERPROFILE%\Documents` | `$XDG_DOCUMENTS_DIR` or `~/Documents` |

Also:
- **Use `package:path`'s `join`; never concatenate with `/`.** Linux filesystems are case-sensitive —
  `Assets/logo.png` and `assets/logo.png` are two files, which breaks builds that worked on Windows.
- File pickers (`file_selector`) go through the XDG desktop portal in a Flatpak build and behave
  differently there; test the packaged artifact, not the dev build.
- **Drag-and-drop targets** — desktop users expect to drop files onto the window. Handle it with
  visible drop feedback. "Open with" associations register at install time (`flutter-release`).
- Windows long paths (>260 chars) still fail without the manifest opt-in; handle the error.

---

## 4. Keyboard and pointer

Desktop input is a different interaction model, not an accessory to touch.

- **`Shortcuts` / `Actions` / `Focus`** — declare intents, bind keys once, and keep the actions
  testable. Scattering `RawKeyboardListener` around does not compose.
- **Standard accelerators are muscle memory**: Ctrl+N/O/S/W/Q, Ctrl+Z / Ctrl+Shift+Z, Ctrl+F,
  F2 to rename, Delete, Escape to dismiss, Enter to confirm, Tab traversal, Space to activate.
  Breaking these is a bug even if nothing crashes.
- **Right-click context menus** wherever the platform convention expects them — lists, text, canvas.
  Their absence reads as an unfinished app.
- Cursor changes (`MouseRegion` with `SystemMouseCursors`) on anything draggable, resizable, or
  clickable.
- Mouse wheel and trackpad: `Scrollable` handles the basics, but horizontal scroll, Shift+wheel, and
  Ctrl+wheel-to-zoom need explicit handling. Trackpad gestures arrive as `PointerPanZoom` events on
  desktop, not as pan drags.
- Visible focus indication, and a logical `FocusTraversalGroup` order — keyboard-only navigation must
  reach every control.

---

## 5. Platform channels

Reach for one only after confirming no maintained package covers it.

- Prefer a **`FlutterPlugin` with a federated structure** over ad-hoc channel calls in `main.dart` —
  it keeps the per-platform code in one place and testable.
- Windows: C++ in `windows/runner` or a plugin, against Win32/WinRT. Linux: C against GTK/GLib.
- **Always ship a graceful fallback.** Wrap every invocation and handle `MissingPluginException` —
  it is what you get on a platform with no implementation, and an unhandled one crashes the feature
  on exactly the platform you tested least.
- Keep channel calls coarse. A chatty channel is both slow and hard to reason about; return a
  struct, not twelve separate calls.
- Long-running native work goes on a background thread and reports back via an `EventChannel`;
  blocking the platform thread freezes the UI.

---

## 6. Divergence to plan for

- **Linux distro variance is the largest source of "works on my machine":** GTK version differences,
  Wayland vs X11 behaviour (window positioning, screen capture, and clipboard all differ), and
  missing system libraries on a minimal install. Declare runtime dependencies explicitly.
- **Windows SmartScreen flags unsigned binaries** — a runtime reality users hit, though the fix (a
  code-signing certificate) lives in `flutter-release`.
- **DPI scaling**: test at 125%, 150%, and 200%, and drag a window between monitors of different scale.
- **Font availability differs** — bundle the fonts you rely on and declare fallbacks, or text renders
  as boxes on a minimal Linux install. Clipboard, notification, and tray behaviour also vary.

---

## 7. Test matrix

**A build that runs on the dev machine proves nothing.** The dev machine has the toolchain, the
fonts, the libraries, and the debug entitlements.

Minimum: a **clean Windows VM** with no dev tools installed, and **at least two Linux distributions**
— one Debian/Ubuntu family and one other (Fedora or Arch), ideally one Wayland and one X11 session.
Run the packaged artifact, not `flutter run`.

Check on each: first launch with no existing config, window restore across a restart, file
picker and drag-drop, DPI scaling, keyboard shortcuts, and the app's behaviour when a permission or
directory is unavailable.

---

## 8. One codebase, branching behaviour

**Branch behaviour, not codebases.** Keep the shared core shared, and put `Platform.isWindows` /
`defaultTargetPlatform` guards at the edges — in a platform service, a capability check, or a
factory — never scattered through the widget tree.

```dart
abstract interface class ShellService { Future<void> revealInFileManager(String path); }
// WindowsShellService, LinuxShellService, AndroidShellService — chosen once, in DI.
```

Use `defaultTargetPlatform` rather than `Platform` in widget code (`dart:io` `Platform` throws on
web, and `defaultTargetPlatform` is overridable in tests). Prefer **capability checks over platform
checks** where you can: "does this support file drop" ages better than "is this Windows".
