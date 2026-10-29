# SubWf — tiny system-wide EQ for Android

A vibecoded ~34 KB equalizer. No libraries, no internet permission, no accounts.
True black, white text, orange faders. One screen: sliders, then a preset dropdown.

## Install
1. Copy `SubWf.apk` to your phone (download, USB, anything)
2. Tap it → allow "install unknown apps" → Install
3. Open it — Bass Boost is already on
4. On Android 13+ allow the notification permission once (keeps the EQ alive)

## What it does
- On Android 9+ it runs a real DSP chain (DynamicsProcessing): 5-band custom EQ + limiter,
  so bass can be boosted hard **without the phone ducking the volume** — the limiter
  catches peaks instead. Falls back to the classic Equalizer+BassBoost on older devices.
- Bass slider adds an extra sub-bass layer (BassBoost) on top of the EQ curve.
- 6 presets: **Bass Boost (default: +7 dB shelf, 75% bass)**, Flat, Rock, Pop, Voice, Dance
- **Speakers row** (under the header): shows the live output count and opens Android's
  Media Output panel — on dual-audio phones (Samsung/Xiaomi/Sony…) you can tick two
  speakers there; the EQ (session 0) colors both of them. Needs the Bluetooth
  permission on Android 12+ (asked once, only used to count outputs).
- Layered background: warm top glow, faint corner glow, film-grain — drawn in code
- **Preset saver**: "+ Save current" button sits at the bottom of the dropdown — name it
  and it joins the list (survives reboot). Each saved preset has an "×" on the right to
  delete it. Saving again with the same name overwrites it.
- Preset picker is a dropdown button; it opens upward when it's near the bottom of the screen
- Drag any slider → dropdown shows "Custom"
- Power switch top-right; state survives reboot and runs in background

## Project layout (`eqapp/`)
- `src/com/pocketeq/app/MainActivity.java` — the screen (faders + dropdown)
- `src/com/pocketeq/app/Effects.java` — global Equalizer + BassBoost wrapper
- `src/com/pocketeq/app/FxService.java` — foreground service keeping EQ alive
- `src/com/pocketeq/app/VerticalSliderView.java` — vertical fader
- `src/com/pocketeq/app/EqGridView.java` — faint dB grid + axis
- `src/com/pocketeq/app/TriangleView.java` — dropdown caret
- `build.sh` — aapt2 → javac → d8 → zipalign → apksigner

## Rebuild
```bash
cd eqapp
bash build.sh   # needs Android SDK build-tools 34 + platform 34; outputs SubWf.apk
```
Signing key: `eqapp/eq.keystore` (alias `eq`, password `eqpass123`) — same key since v1.0,
so updates install straight over the old version.
