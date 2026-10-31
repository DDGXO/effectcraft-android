# Building EffectCraft for Android

Reproduces `effectcraft-0.18.0.apk` from the official
[storytold/effectcraft](https://github.com/storytold/effectcraft) source
using the [artcraft-android](https://github.com/MostafaXS1/artcraft-android)
builder by MostafaXS1.

## Requirements

- Git
- Rust + Cargo, with the Android target: `rustup target add aarch64-linux-android`
- Python 3 **with Pillow** (`sudo apt install python3-pil` or `pip install pillow`) —
  the launcher step fails without it ("Adaptive launcher icons require Pillow")
- A JDK 17–23 (the builder can bootstrap Temurin 21 by itself)
- Android SDK + NDK components (the builder discovers or bootstraps them)

## Steps

```sh
git clone https://github.com/MostafaXS1/artcraft-android
cd artcraft-android
# apply the three patches below
bash build.sh -effectcraft        # add -main to build upstream main instead of the latest stable tag
```

The builder clones `https://github.com/storytold/effectcraft.git` itself
(latest stable `v*` tag by default) into `source/effectcraft/`, vendors its
Android compatibility crates into that checkout, compiles the native
library with cargo-ndk (arm64-v8a), generates the GameActivity launcher,
and packages a locally signed APK at `output/effectcraft-<version>.apk`.

## Why patches are needed

EffectCraft pins a matched pair of Storytold forks in `[patch.crates-io]`
(identical in tag v0.18.0 and on main):

- `egui` / `egui-winit` **0.37.0** — `storytold/egui` rev `6d518fc`
- `winit` **0.30.13** — `storytold/winit` rev `a517b384`

The builder instead vendors its own pair: `vendor/egui-winit` **0.36.2**
and `vendor/winit` 0.30.13 (upstream base). The 0.36.2 egui-winit patch is
silently ignored by Cargo (version does not satisfy `^0.37`), so the build
mixes generations and dies with E0599:

```
error[E0599]: no variant named `ImeResetCompleted` found for enum `winit::event::WindowEvent`
error[E0599]: no method named `reset_ime_with_request` found for reference `&winit::window::Window`
```

Storytold's egui-winit calls IME-reset APIs that only exist in Storytold's
winit fork. The three patches below make the vendored stack consistent
again.

### Patch 1 — merge vendor/winit (in the artcraft-android clone)

Replace the core of `vendor/winit` with the Storytold winit fork (the rev
EffectCraft itself pins), but keep artcraft-android's Android backend
directory, which carries the mouse/scroll/hover input work:

```sh
git clone https://github.com/storytold/winit winit-fork
cd winit-fork && git checkout a517b3846ee8457d84ee4975110bfda6797c44dc && cd ..

cp -a artcraft-android/vendor/winit/src/platform_impl/android /tmp/harness-android-backend
rm -rf artcraft-android/vendor/winit
cp -a winit-fork artcraft-android/vendor/winit
rm -rf artcraft-android/vendor/winit/.git \
       artcraft-android/vendor/winit/src/platform_impl/android
cp -a /tmp/harness-android-backend \
      artcraft-android/vendor/winit/src/platform_impl/android
```

### Patch 2 — reset_ime stubs in the Android backend

The fork's core calls two methods on the Android `Window` that the
artcraft-android backend does not implement. Upstream's own Android
backend implements them as no-ops; add the same stubs to
`vendor/winit/src/platform_impl/android/mod.rs`, inside `impl Window`
(right after `set_ime_purpose`):

```rust
    pub fn reset_ime(&self) {}

    #[inline]
    pub fn reset_ime_with_request(&self, _request_id: u64) {}
```

### Patch 3 — exhaustive matches in the vendored egui-winit

With the fork core in place, the vendored egui-winit 0.36.2 no longer
compiles: its `WindowEvent` matches miss the fork's new
`ImeResetCompleted` variant (E0004, non-exhaustive patterns). In
`vendor/egui-winit/src/lib.rs`:

- in `on_window_event`, add the variant to the "completely ignore" arm
  group:

```rust
            | WindowEvent::ImeResetCompleted { .. } => EventResponse {
                repaint: false,
                consumed: false,
            },
```

- in the event-name match, add:

```rust
        WindowEvent::ImeResetCompleted { .. } => "WindowEvent::ImeResetCompleted",
```

(The event is never emitted on Android — the backend methods are no-op
stubs — so ignoring it is correct.)

## Result

`output/effectcraft-0.18.0.apk` (~99 MB, ARM64, local test signature).
Release builds from the builder are optimized; use `build-debug.sh` for a
debug APK.
