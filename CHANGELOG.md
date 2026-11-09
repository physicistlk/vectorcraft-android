# Changelog

## 2026-11-09 — Upstream v0.23.0

| | |
|---|---|
| APK | `vectorcraft-0.23.0-arm64-v8a-20261109.apk` — sha256 `934c05e61c1f31d222486403ed41fb62f85bcee9cc9a4d88cb29da46e68b3e05` |
| Package | `ai.storyteller.vectorcraft` — versionName `0.23.0`, minSdk 26, targetSdk 35 |
| ABI | `arm64-v8a` only |

### Changed
- Updated port to upstream v0.23.0 (pin `5d0e892e`).
- Port patch applied cleanly; only `Cargo.lock` + `Cargo.toml` (winit → `vendor/winit`) tracked changes.

## 2026-11-07 — Upstream v0.22.0

| | |
|---|---|
| APK | `vectorcraft-0.22.0-arm64-v8a-20261107.apk` — sha256 `92e690e2333ec536d8c54e2bd050e353ce5b6b0c5db924b3b491d7f9eb4e1ea3` |
| Package | `ai.storyteller.vectorcraft` — versionName `0.22.0`, minSdk 26, targetSdk 35 |
| ABI | `arm64-v8a` only |

### Changed
- Updated port to upstream v0.22.0 (pin `654749c1`).


## 2026-11-04 — Upstream v0.21.0

| | |
|---|---|
| APK | `vectorcraft-0.21.0-arm64-v8a-20261104.apk` — sha256 `5efd4e34b0d1f24daac8517eab8553231a05eca4278b2acbc7d77fc6ba5e801c` |
| Package | `ai.storyteller.vectorcraft` — versionName `0.21.0`, minSdk 26, targetSdk 35 |
| ABI | `arm64-v8a` only |

### Changed
- Updated port to upstream v0.21.0 (pin `11862f61`).

## 2026-10-30 — The system clipboard (text)

| | |
|---|---|
| APK | `vectorcraft-0.18.0-arm64-v8a-20261030.apk` — sha256 in `SHA256SUMS.txt` beside the build |
| Package | `ai.storyteller.vectorcraft` — versionName `0.18.0`, minSdk 26, targetSdk 35 |
| ABI | `arm64-v8a` only |

### Changed
- **The system clipboard works for text (`src/clipboard.rs`).** Ctrl+C/X/V now produce
  `Event::Copy`/`Cut`/`Paste` (the Android backend never reports modifiers, so `egui-winit` never
  did), Paste reads `ClipboardManager` through JNI, and `OutputCommand::CopyText` is written back
  with `ClipData.newPlainText`.
- **`Services::clipboard_read` and `Services::system_clipboard` are wired**, the latter as a
  text-only `SystemClipboard`: the TEXT flavour is written and read, Paste's enablement answers for
  text, and a copy with no text flavour (a shape, with Copy as SVG off) writes nothing and is not an
  error. SVG/PDF/PNG copies still live only on VectorCraft's own clipboard.

## Unreleased — re-pinned to upstream 0.18.0, Android 16 (16 KB page size)

| | |
|---|---|
| Upstream pin | `68b68a30acad9b3d8a72802b35f43b82e2bd7d3a` (upstream `0.18.0`) — was `2ac3ba2` (`0.9.0`) |
| Package | `ai.storyteller.vectorcraft` — versionName `0.18.0`, minSdk 26, targetSdk 35 |
| ABI | `arm64-v8a` only |
| ELF page alignment | `0x4000` (16 KB) — Android 16 compatible |

### Changed
- **The upstream pin moved from 0.9.0 to 0.18.0.** Still no upstream file is changed; the whole
  port is the shell crate.
- **Android 16 page-size compatibility.** The native library is linked with
  `-Wl,-z,max-page-size=16384`, so every `LOAD` segment is 16 KB aligned.

### Fixed
- **`FilePick::filters` changed shape upstream** — from borrowed `&[(&str, &[&str])]` to an owned
  `Vec<FileFilter>`. The picker reads both shapes now.
- **`SheetApp::raw_input_hook` takes the context** upstream; the shell forwards it.


## v0.1.1 — 2026-10-13 (scrolling, modifiers, fonts, picker wording)

| | |
|---|---|
| APK | `vectorcraft-0.9.0-arm64-v8a-20261013.apk` (29.6 MB) — supersedes the v0.1.0 asset, same signing key |
| sha256 | `c7848549067eb1192e7935f45cdbf91490f3366ccfdd73c5d44ce1236f219aa6` |
| Built from | upstream `storytold/vectorcraft` @ `2ac3ba2b866ad9d34b2252dd8024f73e34f4b2b8` + this repository at the commit that released it |
| Package | `ai.storyteller.vectorcraft` — versionName `0.9.0`, versionCode `16779520`, minSdk 26, targetSdk 35 |
| ABI | `arm64-v8a` only |

### Changed
- **Modifier keys work.** `src/modifiers.rs` is now part of this port: the shell tracks Ctrl/Shift/Alt
  from the key events the platform does send (winit's Android backend never emits `ModifiersChanged`,
  which is the only source egui reads) and re-injects the state onto every key and pointer event, once
  per frame. Ctrl+A, Ctrl+S, Ctrl+Z and Shift-click work; this closes the gap v0.1.0 was published
  with.

### Notes
The build was made from exactly the source in this repository (upstream pin + the patch + the port
crate) and signed with the same local test key as v0.1.0, so it installs over it.


## v0.1.0 — first published port (VectorCraft 0.9.0)

| | |
|---|---|
| APK | `vectorcraft-0.9.0-arm64-v8a-fonts.apk` (29.6 MB) |
| sha256 | `f4fa64a2506e1fe8f61c4714c9756527020020790cb15f47fa1bc3e7ed2e9438` |
| Built from | upstream `storytold/vectorcraft` @ `2ac3ba2b866ad9d34b2252dd8024f73e34f4b2b8` + `apps/vectorcraft-android/` |
| Package | `ai.storyteller.vectorcraft` — versionName `0.9.0`, minSdk 26, targetSdk 35 |
| ABI | `arm64-v8a` only |
| Signature | throwaway local test key, not an official key |

### In this release
- Android shell crate: NativeActivity entry point, `Services` implementation, file logger, single
  window, quit that really quits (the stale-close problem described in the notes).
- In-app file picker via `Services::start_pick`, with cached listings, readable-root probing, "Show all
  files" and hidden-file count.
- Platform fonts from `/system/fonts` and VectorCraft's own **USB-visible** *Fonts* folder, via
  `set_platform_font_files` / `set_app_font_dir`.
- The desktop title bar is not drawn on Android (`custom_titlebar` off, and the `system_title_bar`
  preference pinned off so the brand mark survives).
- Launcher icon, label, storage permissions, edge-to-edge theme variant.


### Known issues
See *Known gaps* in the [README](README.md) and [`docs/PORT-NOTES.md`](docs/PORT-NOTES.md).
