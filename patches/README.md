# Patch

This port now changes **one** upstream file: `Cargo.toml`, which resolves `winit` from
`vendor/winit` instead of the git revision, so the Android backend forwards trackpad
scroll (`MotionAction::Scroll`) and stylus/mouse hover (`MotionAction::HoverMove`) to
`MouseWheel` and `CursorMoved`. Everything else the port needs still lives in
`apps/vectorcraft-android/`.

`0001-vectorcraft-android-port.patch` is the diff against upstream.
