# Port notes — VectorCraft on Android

Upstream: [`storytold/vectorcraft`](https://github.com/storytold/vectorcraft) @
`bcebabe3f` (v0.19.0)
Published APK: `vectorcraft-0.9.0-arm64-v8a-fonts.apk` (the v0.1.1 release, built from v0.9.0)

**One upstream file is changed.** `Cargo.toml` resolves `winit` from `vendor/winit` instead of
the git revision, so the Android backend forwards trackpad scroll and stylus/mouse hover (see the
2026-10-30 section below). Everything else the port needs lives in `apps/vectorcraft-android/`: `src/lib.rs` (379 lines) is the shell,
the entry point and the `Services` implementation, and `src/file_picker.rs` (709 lines) is the in-app
picker *and* the font registration. There is deliberately nothing in `patches/`.

This port predates the sibling ports in this set (PhotoCraft, WordCraft, PdfCraft, GridCraft,
DeckCraft), and the gaps listed at the end of this file are the reason the later ones are shaped the
way they are.

## 2026-10-31 — Preferences written while the app runs (`prefs_watch.rs`)

VectorCraft previously saved its preferences only on the exit path, so a swipe from Recents could
lose UI state including the recent-documents list. It now carries the same `prefs_watch::Writer`
the other ports use: `ui.json` is written (atomically, only when it changed, at most every 2 s)
from `Shell::logic`, and again with `store_now` on a lost-focus frame. `save_prefs` is kept for the
focus-loss path and for any caller that wants an immediate write.

## 2026-10-30 — Trackpad wheel + pen hover (`vendor/winit`)

This port previously used the git winit and changed **no** upstream file. It now vendors the
same patched winit the other ports ship (copied into `vendor/winit/`, with `Cargo.toml`
resolving winit from it), so the backend forwards `MotionAction::Scroll` and
`MotionAction::HoverMove` to `MouseWheel` and `CursorMoved`. This is the first upstream file
this port changes; the patch is no longer empty.

## 2026-10-30 — Flush preferences on suspend (`raw_input_hook`)

A lost-focus frame now calls `save_prefs`, so a swipe from Recents (or a paused Activity being
killed) no longer loses UI state including the recent-documents list.

## 2026-10-30 — Open-with and share intents (`src/intents.rs`)

**The complaint:** a document tapped in a file manager, or shared from another app, never reached
VectorCraft; and there was no way to hand the current document to another app.

`src/intents.rs` covers the half that does not need `startActivityForResult`:

- **Incoming `ACTION_VIEW`** — the launching URI is read at startup and copied out of a
  `content://` URI into the cache directory; the shell opens it on its first frame (`io::open_path`).
- **Incoming `ACTION_SEND`** — `EXTRA_TEXT` and `EXTRA_STREAM` are handled the same way.
- **Outgoing `ACTION_SEND`** — `intents::share_text` starts a chooser; `share_file` is provided but
  needs a `FileProvider` entry to work on Android 7+.

The launching intent is read in `android_main` and consumed once in `Shell::logic`.

**Scope:** opening a file handed to the app, and sharing text. SAF pickers remain deferred (no
`startActivityForResult` in `android-activity` 0.6.1); the in-app picker is still the way to open
or save.

**Checked:** the module's host test passes, the crate compiles for `aarch64-linux-android`, and the
host suite is 23/23. The JNI intent path has not run on a device.

## 2026-10-30 — The system clipboard (`src/clipboard.rs`)

**The complaint:** copy and paste worked only *inside* the app; text could not cross the app
boundary. The Edit menu's Paste was also always disabled, because nothing could answer whether the
clipboard had anything.

Two causes, both fixed here:

- **winit's Android backend has no clipboard**, so `egui-winit` fell back to its in-app simulated
  clipboard.
- **The backend never reports modifiers** (see *Modifier keys* below), and `egui-winit` decides
  Ctrl+C/X/V is a clipboard shortcut from its own modifier state. On Android the three keys arrived
  as ordinary `Event::Key` presses, so `Event::Copy`/`Cut`/`Paste` were never produced.

`src/clipboard.rs` supplies the same four pieces as its siblings, plus VectorCraft's format-aware
trait:

- `install(&app)` — this thread's Activity handle, captured before `run_native`.
- `shortcuts(raw)` — in `raw_input_hook`, after the modifier bridge: a plain Ctrl+C/X/V becomes
  `Event::Copy`/`Cut`/`Paste` and the press is consumed, as on the desktop. Ctrl+V reads the
  **system** clipboard and pushes nothing when it is empty. Ctrl+Shift/Ctrl+Alt go to the app.
- `publish(ctx)` — after the app's `ui`: every `OutputCommand::CopyText` this frame is written with
  `ClipData.newPlainText` + `setPrimaryClip`, and left in the command list so egui-winit's in-app
  copy still happens.
- `text()` — the read half, wired in twice: `Services::clipboard_read` (what Paste's menu item and
  its enablement ask) and `Services::system_clipboard`, a text-only `SystemClipboard`.

**Why `SystemClipboard` and not just the two text hooks.** VectorCraft's Copy publishes a family of
flavours (SVG, PDF, PNG, text) and its Paste reads them in a fixed order. The Android implementation
answers only for text: `write` puts the TEXT flavour on the system clipboard and does nothing when a
copy has none (a shape with Copy as SVG off — that copy still lives in the in-app clipboard, and
reporting an error would be a complaint about a copy that worked); `read` returns a text `Flavour`
when asked for `text/*`; `has` answers the same question for Paste's enablement; `holds_ours`
compares what it last wrote with what is there now. SVG/PDF/PNG never reach another app on Android,
but they are still on VectorCraft's own clipboard, so copy-paste inside the app is unchanged.

The JNI side is `ClipboardManager`/`ClipData` directly. A read Android refuses (Android 10+ allows
reads only while the app is focused) is logged and cleared, and answers "nothing".

**Checked:** eight unit tests plus the app crate's own picker/modifier tests pass on the host, and
the crate compiles for `aarch64-linux-android`. The JNI path has not run on a device — none was
connected for this build.

## 0. 2026-10-28 — The port was re-pinned to upstream v0.19.0

The port was re-pointed from upstream `0.9.0` (`2ac3ba2`) to `0.18.0` (`68b68a30`). No upstream
file was changed at that time — VectorCraft's UI crate still exposes the seams the shell needs
(`Services::start_pick`, `vectorcraft_text::set_platform_font_files`, `set_app_font_dir`) — so the
patch is still empty and the whole port is still the shell crate.

Upstream moved a long way (nearly 4 000 commits), and two of its API changes reached the shell:

- **`FilePick::filters` changed shape.** It was `&[(&'static str, &'static [&'static str])]` and is
  now `Vec<FileFilter>`, where `FileFilter = (String, Vec<String>)` — the labels and extensions are
  owned, so a host can build them at runtime. The shell's `extensions_of` helper read the old
  shape; it now has a second helper, `extensions_of_owned`, for the new one. The engine's own
  `place_filters()` still returns the old shape, so both helpers are live.
- **`SheetApp::raw_input_hook` takes the context.** Upstream's signature grew a `&egui::Context`
  parameter, so the shell's `eframe::App::raw_input_hook` forwards it instead of dropping it.

Everything else the shell uses — `Services`, `VectorcraftApp`, `picks::{PickRequest, PickAnswer}`,
`prefs_dialog::restore`, `graphics::GraphicsLoss`, `place_filters`, `catch_panic`,
`set_app_font_dir`, `set_platform_font_files` — is still present at `0.18.0` with the same shape.
`write_atomic` moved from `crates/engine` to `crates/format`, but it is still re-exported through
`engine::cmd::fileio`, so the shell's import is unchanged.

**Verified:** `cargo check -p vectorcraft-android --target aarch64-linux-android` exits 0 against
v0.19.0, and the shell crate in this repository is byte-identical to the one the APK was built from.

---

## 1. The shell crate

`vectorcraft_ui_egui::Services` is a plain struct of `Option<Box<dyn Fn…>>` fields that derives
`Default`: a platform supplies what it can and omits the rest. The desktop binary and the browser
shell fill it in their own way; this is the third one. Once it exists, the real engine and the real
egui interface run unmodified.

Load-bearing details, the same in every port of this set:

- **`[lib] name = "vectorcraft"` + `crate-type = ["cdylib"]`** — cargo-apk turns that into
  `lib/arm64-v8a/libvectorcraft.so` and points the injected `android.app.lib_name` at the artifact
  name (`-` → `_`). A mismatch gives an APK that installs and then dies on launch.
- **`options.android_app = Some(app)`** — `eframe::run_native` errors without it.
- **A file logger in the app's own data directory** — `logcat` needs adb, and a sideloaded build has
  none.

## 2. Fonts — two registrations the desktop makes in its own binary

Both are silent when missing, and both are one-shot, so the shell does them *before* the window opens:

- **The platform's font files.** `system_font_dirs()` branches macOS / Windows / else-Linux, and
  Android falls into the Linux branch: `/usr/share/fonts`, `$XDG_DATA_HOME/fonts`, `~/.fonts` — none
  of which exist on a tablet. Missing directories are skipped without a word. The shell registers the
  real ones from `/system/fonts` (with `/product/fonts`, `/system_ext/fonts`, `/vendor/fonts`) via
  `vectorcraft_text::set_platform_font_files`, which takes a plain `fn` and is re-called per scan.
- **VectorCraft's own *Fonts* folder**, via `set_app_font_dir`. Without it the app's font-install path
  fails with the message that names the cause — *"VectorCraft's Fonts folder isn't set here: the
  desktop app and vectorcraft-cli set it"*. It is pointed at the **USB-visible**
  `<App media>/Fonts`, not `Android/data`: Android 11+ hides `Android/data` from MTP, so a `.ttf`
  copied in over a cable would silently never arrive.

Bundle fonts are `include_bytes!`d upstream, so text works even with no system font at all; these two
registrations are what make *other* documents render in the typeface they were set in.

## 3. The desktop title bar does not exist on a tablet

VectorCraft draws its own title bar for Windows and Linux — minimize/maximize/close at the right end
of the app bar, plus drag and resize zones — which is meaningless where the surface is always
fullscreen. The shell sets the `custom_titlebar` flag off. That alone is not enough: with it off,
`chrome::system_title_bar` would fall through to a stored preference of the same name and claim the
OS draws the title bar, which drops the brand mark instead. So the shell pins that preference off too
(`prefs.system_title_bar = false`) before the first frame. The app bar — brand, menus, document
switcher — stays; the caption buttons go.

## 4. File ▸ Exit left the app unusable until it was cleared from Recents

Desktop eframe apps answer Exit with `egui::ViewportCommand::Close` and let the window closing end the
process. **On Android that ends nothing**: `egui-winit` turns it into a plain `ViewportEvent::Close`
and never calls winit's window API, which is stubbed on Android anyway. The native thread returns
while the Activity lives on, so the app looks alive, is dead the moment it is touched, and cannot be
reopened until it is swiped away from Recents.

The shell watches the state the app sets when it leaves (`ui.status == "quit"`, set only after nothing
is unsaved), saves the preferences, flushes the log and ends the process:

```rust
fn quit_if_asked(&self) {
    if self.inner.ui.status != "quit" { return; }
    save_prefs(&self.inner);
    log::info!("quitting");
    log::logger().flush();
    std::process::exit(0);
}
```

It is checked in **both** `logic` and `ui`: a menu click runs during `ui`, and no repaint is
guaranteed afterwards, so waiting for the next `logic` is how Exit appears to do nothing until the
user taps again.

## 5. File ▸ Open did nothing — `src/file_picker.rs`

`rfd`, the desktop's dialog library, has no Android backend at all: its `backend.rs` compiles no
module for a target that is neither Linux/BSD, macOS, Windows nor wasm32, and the build fails with
`the trait bound FileDialog: FilePickerDialogImpl is not satisfied`. Android's own picker is not a
substitute — the Storage Access Framework needs `startActivityForResult`, which `android-activity`
0.6 does not expose. So the picker is drawn with egui, between frames, on the seam.

VectorCraft's seam is the **receiver** shape: `Services::start_pick(request)` stashes the request and
returns a `Receiver<Answer>` the app polls once a frame, then re-runs the command the request came
from with the answer. There is one trap in it, and the shell has to know about it: the app consults
`start_pick` only after `picks::can`, which looks at the *synchronous* `pick_open` / `pick_save` /
`pick_open_multi` / `pick_folder` closures — so those are registered too, returning "cancelled", or
the asynchronous picker is never reached at all. They are only asked on an entry where a dialog shown
in line would hold the window.

Details that matter on a tablet, all of them paid for once already: the dialog is sized from
`ctx.content_rect()` (a portrait tablet is narrower than egui's desktop default) with enlarged
padding and spacing; it keeps asking for repaints while open (the app otherwise asks once a second
while waiting, so a tap lags); the folder listing and the readability probe are cached per visit, not
per frame; a root that cannot be read stays tappable and *names the permission and the Settings path*
it needs, instead of sitting greyed out; the filter is the union of every extension the app asked for,
with **"Show all files"** and a count of what the filter is hiding; and a save answer is
`dir.join(name)`, because the app's atomic write needs the parent to exist.

## 6. Permissions, icon, theme

`MANAGE_EXTERNAL_STORAGE` (granted by hand in Settings ▸ Special access — Google Play restricts it to
file-manager-class apps, which does not apply to a sideload), `READ_MEDIA_IMAGES` for Android 13+,
and `READ_EXTERNAL_STORAGE` / `WRITE_EXTERNAL_STORAGE` capped with `max_sdk_version` for older
releases. The APK ships launcher icons at 48/72/96/144/192 px from the app's own hicolor PNGs (Android
scales) and the same two-variant edge-to-edge theme file as its siblings (`res/values/styles.xml`),
needed because `targetSdkVersion 35` puts the app under a transparent status bar on Android 15.

## Gaps — what this port does *not* have

The later ports in this set fixed these; this one has not been reworked since:

- **No two-finger scrolling.** There is no `src/touch_scroll.rs` and no wheel bridge, so a two-finger
  drag does not scroll the canvas. (The modifier tracker this port once lacked is now here, and so is
  the text clipboard; this one has not been done.)
- **Preferences are written on the exit path, not while the app runs.** `save_prefs` runs when the app
  reaches its quit state; a swipe from Recents never gets there. The sibling ports write the
  preferences from a change-detected watcher (`prefs_watch`), which is what makes the recent-documents
  list survivable.
- **One window.** There is no *New Window* support, so two documents cannot be put side by side.
- **No audio.** `Services::audio_out` stays `None`: `cpal` has no Android backend here.

## 2026-10-13 — Modifier keys (the gap this port was published with)

This port shipped with no modifier handling: winit's Android backend never emits `ModifiersChanged`,
and `egui-winit` reads modifiers from that event and nowhere else, so `Ctrl`/`Shift`/`Alt` were
permanently false — `Ctrl+A`, `Ctrl+S`, `Ctrl+Z` and `Shift`-click did nothing. The sibling ports had
already fixed it; this one had not.

`src/modifiers.rs` is now in this crate and applied from the shell's `raw_input_hook`: the key events
do arrive, so the state is tracked from them, folded into the held state, stamped onto every `Key` and
`PointerButton` event, and re-reported once per frame with a `ModifiersChanged`. State is carried
between frames because the press and the click usually land in different ones. The copy here is the
simple variant (no unit tests, no return value) — the shells that synthesise wheel events use the
version whose `apply()` hands the state back.
## Not verified here

The APK attached to **v0.1.1** was built from exactly this tree on 2026-10-13 (upstream pin + the patch + the port crate), and the checks in the README's verification table were run against it. What that does *not* buy is a runtime claim: it was not launched on a device or emulator as part of publishing, so every behavioural statement below rests on the hand-testing the porter did on an arm64 tablet. Unlike its
siblings, this port's fixes have not been exercised against a second round of hand-testing on the
tablet — treat the runtime side of this one as the least confirmed in the set.
