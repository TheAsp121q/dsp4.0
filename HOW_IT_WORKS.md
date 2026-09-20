# How SubWf works — the full tour

This is everything you need to understand, fix, and extend the app without help.
Read it once top to bottom, then poke at the code with this open. ~500 lines of Java total.

---

## 0. The 30-second map

```
┌──────────────────────────── your phone ────────────────────────────┐
│                                                                     │
│  Spotify ─┐                                                         │
│  YouTube ─┤--> [ Android audio mixer ] --> [ SubWf's DSP chain ]     │
│  Games  ──┘            (session 0)               │                  │
│                                        EQ → BassBoost → Limiter     │
│                                                  │                  │
│                                                  ▼                  │
│                                     speaker / wired / Bluetooth     │
└─────────────────────────────────────────────────────────────────────┘

MainActivity  = the one screen. Sliders + dropdown. Only writes numbers.
Effects.java  = creates the DSP chain, pushes the numbers into it.
FxService     = stays alive in background so Android doesn't drop the effects.
BootReceiver  = restarts the service after phone reboot.
prefs         = tiny storage: your curve, bass level, on/off, saved presets.
build.sh      = 6 commands that turn the folder into SubWf.apk.
```

The secret of the whole app is one sentence: **Android lets any app attach audio
effects to "session 0", which is the final mix of ALL sound on the phone** — so one
tiny app shapes Spotify, YouTube, games, speaker, wired, and Bluetooth at once.

---

## 1. Audio sessions and session 0

When any app plays sound, its player gets an "audio session" (an ID number).
Audio effects (equalizer, bass boost, reverb...) attach to a session.

- Attach to session 42 → you affect only that one app's stream.
- Attach to **session 0** → you affect the mixed output of everything. That's what SubWf does:

```java
eq = new Equalizer(0, 0);          // (priority, session 0)
bb = new BassBoost(0, 0);
dp = new DynamicsProcessing(0, 0, config);   // the modern one, Android 9+
```

That requires the `MODIFY_AUDIO_SETTINGS` permission — a normal, no-prompt permission.
No root, no special access. It's an official (if old-school) mechanism.

## 2. Why your bass boost used to get quiet — and the DSP fix

The old `Equalizer` effect lives in the phone's audio hardware layer (HAL). Almost every
HAL does this: **if you raise any band above 0 dB, it lowers the master volume** so the
boosted signal can't clip ("headroom"). So +8 dB bass = quieter everything. Not a bug
in your code — physics policy.

The fix is the modern API, `DynamicsProcessing` (Android 9+, API 28). You build a chain:

```
input → 5-band EQ → limiter → out
```

- The EQ is *yours*, running in the DSP: bands at 60 / 230 / 910 / 3.6k / 14 kHz,
  gain in dB per band (`EqBand(enabled, cutoffHz, gainDb)`).
- The **limiter** is the trick: it's a ceiling that catches peaks automatically.
  Bass boost makes the wave hit the ceiling → limiter gently flattens only the peaks →
  average loudness stays. Volume preserved, bass added. That's the "no mercy" mode.

Look at `Effects.ensure()`: it builds the config (`Config.Builder` with stage flags and
band counts), creates the effect on session 0, and `apply()` pushes slider values in as
`gainDb = millis / 100f`. If `DynamicsProcessing` isn't available (Android 8 or weird OEM),
it falls back to the classic `Equalizer` — same UI, gentler result.

`BassBoost` is a separate, additional effect: it only adds low-frequency emphasis,
controlled by "strength" 0–1000. That's your Bass slider.

## 3. Staying alive

Audio effects live inside your app's process. If Android kills the process, the effects
vanish silently. So the app needs to *exist* while you're playing music:

- **Foreground service** (`FxService`): a service that posts one notification
  ("SubWf is on"). That's the deal with the system: notification in exchange for
  not being killed. `START_STICKY` = if killed anyway, restart it.
- **BootReceiver**: listens for `BOOT_COMPLETED` and restarts the service after reboot
  (only if the switch was on — checked in prefs).
- The switch in the UI = `setEnabled(true/false)` on the effects + start/stop the service.

That's also why the app asks for the notification permission on Android 13+.

## 4. The UI — zero libraries

One `Activity` (a screen) loads `res/layout/activity_main.xml`: a vertical
`LinearLayout` (header → divider → EQ block → PRESET label → dropdown button).
The EQ block is a fixed-height `FrameLayout` (270dp) holding two stacked views:
the grid (drawn behind) and the row of sliders.

**The sliders are a hand-drawn custom View** — this is the most transferable skill in
the whole project. `VerticalSliderView` does two things:

1. `onDraw(Canvas)` — runs every redraw. You paint with `Paint` objects:
   rounded rect = track, darker rounded rect from knob to bottom = fill,
   circle = knob, `drawText` = value and label. No images, pure geometry.
   The value → Y position math is just linear mapping between dB range and pixels.
2. `onTouchEvent` — when a finger drags, map finger Y back to a value,
   snap it to steps of 50 (0.5 dB), redraw, and fire `listener.onChanged(value)`.

The Activity passes a listener that writes prefs and calls `Effects.apply()`.
That one-way flow — **touch → value → storage → engine** — is the entire app logic.

The dropdown is a `PopupWindow` containing rows built in code (no XML needed).
Two details worth stealing:
- it measures the free space under the button and **opens upward** if there's no room
  (that was the "can't reach the menu" bug), and
- it caps its height so it scrolls when the list gets long.

## 5. State — SharedPreferences everywhere

`SharedPreferences` = a private key→value file that survives reboots. The app keeps:

| key | meaning |
|---|---|
| `gains` | `"700,300,0,0,100"` — dB×100 per band, comma-separated |
| `bass` | 0–1000 strength |
| `on` | power switch |
| `preset` | name shown on the dropdown button |
| `user_presets` | one saved preset per line: `name\|gains\|bass` |

Rule the code follows: **every change writes immediately** (slider drag, toggle, save),
and the UI is always rebuilt *from* storage. Nothing lives only in memory, so there's
no "save state" code anywhere.

The preset saver is 4 tiny methods: `loadUserPresets`/`persistUserPresets` (text format
above), `saveUserPreset` (reads current gains+bass, appends or overwrites by name),
`deleteUserPreset` (removes the line; if it was selected, flip button to "Custom").
The dropdown merges built-ins and saved ones behind `nameFor/gainsFor/bassFor/totalPresets`
so the list code doesn't care where a preset comes from.

## 6. The build — what build.sh actually does

Gradle/Android Studio do exactly these steps; here they are in the open:

```
1. aapt2 compile + link   resources (XML, icons, strings) + AndroidManifest
                          → compiled resources AND R.java (the R.id.xxx constants)
2. javac                  all .java → .class files (checked against android.jar,
                          the stub of the Android API)
3. d8                     .class → classes.dex (Android bytecode, "DEX")
4. zip                    put classes.dex + resources into one .zip = the APK body
5. zipalign               align data to 4 bytes (the system requires it)
6. apksigner sign         sign with your private key — Android refuses unsigned APKs.
                          SAME KEY = new version installs OVER the old one.
```

`android.jar` is just the API *stubs* (no code) — the real implementations are on the
device. `--min-sdk-version 21` says "must run on Android 5+", and runtime guards like
`Build.VERSION.SDK_INT >= 28` handle newer-only features.

## 7. Doing it yourself next time

**One-time setup (~20 min):**
1. Install JDK 11+ and the Android "command line tools".
2. `sdkmanager --sdk_root=/opt/android "platforms;android-34" "build-tools;34.0.0"`
   (accept licenses). That's the whole SDK you need — no Studio required.
3. Any editor (VS Code is fine). Copy `eqapp/` as your skeleton.

**The dev loop:**
```
edit code → bash build.sh → adb install -r SubWf.apk → open app on phone
                                      └→ adb logcat shows crashes with the line number
```
Enable Developer options + USB debugging on the phone once; `adb` is in build-tools'
sibling `platform-tools`.

**When something breaks, error → meaning:**
| error | where | usually means |
|---|---|---|
| `resource not found` / `AAPT2` | step 1 | typo in XML, wrong `@color/name` |
| `cannot find symbol` | step 2 | missing import, wrong method name |
| `Unsupported class version` | step 2 | wrong JDK flags — keep `-source 1.8` |
| `Uncaught TranslationError` | step 3 | syntax OK but weird code — read the class named |
| `INSTALL_PARSE_FAILED...` / install fail | install | manifest or signature problem (key changed?) |
| app crashes on open | logcat | read the `Caused by:` line — it names file + line |

**Where to look things up** (this is 90% of "not needing AI"):
- `developer.android.com/reference` — every class documented with copy-paste examples
  (search e.g. "DynamicsProcessing android").
- For effects specifically: the `android.media.audiofx` package page lists them all.
- `adb logcat` tells you the truth about crashes. Read the first `Caused by:`.

**Ideas to grow it (each is a small step from this code):**
- per-device profiles → `AudioDeviceCallback`, re-apply on device connect
- more bands → change `CUTOFFS` in Effects.java (that's literally it)
- a mute/auto-off timer → `Handler.postDelayed` in the service
- rename presets → pre-fill the save dialog's EditText with the old name

## 8. File map

| file | job | lines |
|---|---|---|
| `AndroidManifest.xml` | declares app, permissions, activity/service/receiver | 45 |
| `res/layout/activity_main.xml` | the screen skeleton | 100 |
| `res/values/colors.xml`, `strings.xml`, `styles.xml` | theme: black, white text, orange | 40 |
| `src/.../MainActivity.java` | screen, sliders wiring, dropdown, presets, saver | ~430 |
| `src/.../VerticalSliderView.java` | the hand-drawn fader | ~170 |
| `src/.../EqGridView.java` | faint dB grid + axis labels | ~70 |
| `src/.../TriangleView.java` | the little caret | ~35 |
| `src/.../Effects.java` | DSP chain (DynamicsProcessing + BassBoost, legacy fallback) | ~140 |
| `src/.../FxService.java` | foreground service + notification | ~65 |
| `src/.../BootReceiver.java` | restart after reboot | ~25 |
| `build.sh` | the 6-step pipeline | ~45 |
| `eq.keystore` | your signing key — BACK THIS UP or updates won't install over | — |
