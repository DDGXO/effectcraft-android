# EffectCraft for Android (Unofficial Port)

Android build of **EffectCraft**, the motion graphics / compositing app by
[Storytold](https://github.com/storytold/effectcraft), at tag **v0.18.0**.

This repository contains the upstream EffectCraft source (v0.18.0, see
`README-UPSTREAM.md` and `ATTRIBUTION.md`) plus the Android build
documentation needed to reproduce the APK. It is an unofficial port and is
not affiliated with or endorsed by Storytold.

## Download

Grab the APK from the **Releases** page:

- `effectcraft-0.18.0.apk` — ARM64, signed with a local test key. Install
  directly on an Android device.

## How it was built

The port uses the **[artcraft-android](https://github.com/MostafaXS1/artcraft-android)**
builder by MostafaXS1, which compiles the official Storytold source with
cargo-ndk into a native `.so` and packages it as an APK (GameActivity
runtime, JNI bridges for storage, IME, clipboard and touch/mouse input).

Three small compatibility patches were needed against the current
EffectCraft — they are documented step by step in
**[BUILD-ANDROID.md](BUILD-ANDROID.md)**:

1. Merge `vendor/winit`: storytold/winit fork core + artcraft-android's
   Android backend (IME-reset APIs vs. mouse/scroll backend).
2. Add the fork's no-op `reset_ime` / `reset_ime_with_request` stubs to the
   Android `Window` impl.
3. Add the `ImeResetCompleted` match arms to the vendored egui-winit.

Build machine: an InstaBox Linux VM (4 vCPU / 8 GB RAM, Debian 13).

## Credits

- **EffectCraft** — [storytold/effectcraft](https://github.com/storytold/effectcraft) (upstream, this repo's source)
- **Android builder** — [MostafaXS1/artcraft-android](https://github.com/MostafaXS1/artcraft-android)
- **Build & porting fixes** — Masha AI for DGXO
