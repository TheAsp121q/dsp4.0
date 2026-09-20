# Adding dual-speaker output to SubWf — honest guide + architecture

## 0. Read this first: who owns what

Streaming to two Bluetooth speakers at once is a **system capability**, not an app one.
Android's audio policy routes the mixed output to ONE active A2DP device by default.
Connecting speaker #2 usually demotes speaker #1. Vendor builds (Samsung "Dual Audio",
Xiaomi, Sony) patched their Bluetooth stacks to allow two simultaneous A2DP streams —
that code is private; no public API toggles it.

So there are three doors:

| Door | What it gives you | Difficulty | Needs |
|---|---|---|---|
| A. Vendor Dual Audio | Perfect sync, system-level, works with ALL apps + your EQ already | trivial (user toggle) + helper UI | Samsung/Xiaomi/Sony-class phone |
| B. App-owned playback | Your app plays music to 2 devices itself via per-track routing | medium | any phone that honors setPreferredDevice() on A2DP |
| C. System-audio capture + dual playback | Like B but grabs other apps' audio too | hard | Android 10+ (MediaProjection capture), still per-device quirks |

Key insight that makes Door A beautiful for THIS project:
SubWf's chain is attached to **session 0 = final mix**. Whatever the system does after
that — including splitting to two speakers — happens AFTER your EQ. So on a Samsung
with Dual Audio on, **both speakers are already EQ'd by your app. Zero code needed.**
Your job there is only: detect the situation and guide the user.

---

## 1. Door A — dual audio the smart way (build this first)

Modules:
- `BluetoothManager` / `BluetoothA2dp` profile proxy → list connected A2DP sinks.
- Permissions: `BLUETOOTH_CONNECT` (API 31+, runtime) + legacy `BLUETOOTH` (below).
- A small "Two speakers" card in the UI:
  - 1 sink connected → "Connect a second speaker, then turn on Dual Audio"
  - 2+ sinks connected → deep-link the vendor toggle:
    Samsung: `adb shell settings` no — user path is Settings ▸ Connections ▸ Advanced ▸ Dual Audio (or Media output panel)
  - Best universal trick: launch the system **Media Output switcher** where Android
    itself offers multi-select when the build supports it:
    `MediaRouter2` / `MediaOutputSwitcher` (API 33:
    `android.media.Routing2`/`MediaOutputSwitcher` intent `com.android.systemui.MEDIA_OUTPUT_DIALOG`)
    — sending that intent opens Google's own device-picker with multi-select checkboxes.
    That single button covers most modern phones with zero privileged code.

Skeleton:

```java
BluetoothManager bm = getSystemService(BluetoothManager.class);
BluetoothAdapter ad = bm.getAdapter();
// proxy the A2DP profile to list sinks
ad.getProfileProxy(ctx, listener, BluetoothProfile.A2DP);
// in listener: ((BluetoothA2dp) proxy).getConnectedDevices().size()

// open the system picker (API 33+, non-exported intent but commonly allowed):
Intent i = new Intent("com.android.settings.panel.action.MEDIA_OUTPUT");
i.putExtra(Intent.EXTRA_PACKAGE_NAME, getPackageName());  // or the SYSTEM UI dialog action
startActivity(i);
```

Success criteria: on a Dual Audio phone, both speakers play, both are EQ'd.
Ship this. It's 1 day of work and it's the only path that is sync-perfect.

## 2. Door B — your app streams to both speakers itself

Idea: YOU become the player (local files / your own sources). You create TWO AudioTracks
and ask each to route to a different device. Per-track device selection is public API:

```java
// 1) enumerate outputs
AudioDeviceInfo[] outs = audioManager.getDevices(AudioManager.GET_DEVICES_OUTPUTS);
// pick the two with getType() == AudioDeviceInfo.TYPE_BLUETOOTH_A2DP
// (match by getProductName() so the user can choose L/R speaker)

// 2) two tracks, same PCM, different preferred devices
AudioTrack left = new AudioTrack.Builder()
    .setAudioAttributes(new AudioAttributes.Builder()
        .setUsage(USAGE_MEDIA).setContentType(CONTENT_TYPE_MUSIC).build())
    .setAudioFormat(new AudioFormat.Builder()
        .setEncoding(ENCODING_PCM_16BIT).setSampleRate(44100).setChannelMask(CHANNEL_OUT_STEREO).build())
    .setTransferMode(MODE_STREAM).setBufferSizeInBytes(buf).build();
left.setPreferredDevice(devSpeaker1);

AudioTrack right = ...; right.setPreferredDevice(devSpeaker2);

// 3) pump the same decoded PCM into both
short[] pcm = decodeNextChunk();
left.write(pcm, 0, pcm.length, WRITE_BLOCKING);
right.write(pcm, 0, pcm.length, WRITE_BLOCKING);
```

EQ on your own streams: attach the SAME DynamicsProcessing chain to YOUR tracks'
sessions instead of session 0 — you already have all the code:

```java
int session = left.getAudioSessionId();
new DynamicsProcessing(0, session, sameConfigYouAlreadyBuild());
```
(Refactor idea: give `Effects.ensure(Context)` an optional session parameter —
session 0 = system mode, your session = split mode. The presets/sliders feed both.)

### The two problems Door B must solve (this is the "somehow synced" part)

**Problem 1 — asymmetric latency.** Each speaker buffers differently (codec, chip):
speaker A might play 180 ms behind, speaker B 260 ms. Same sample written now
audibly staggers → comb-filter echo. Fix: give each track its own delay buffer —
write to B from a buffer that lags the data you give A:

```java
// per-track ring buffer; delaySamples = measured ms * sampleRate / 1000
writeToTrack(track, ringA.read());            // leader, no delay
writeToTrack(track2, ringB.read(delaySamples)); // follower, delayed
```

Measure the delay per speaker (pick one):
- Manual (simplest, ship this): play a click/chirp to each speaker, the user holds
  the phone mic? No — simplest is a **slider per speaker, 0–500 ms**, user aligns
  by ear walking between speakers. Honest and it works.
- Auto (later): MIC impulse measurement — play a click to speaker A, record via
  `AudioRecord`, find onset time; repeat for B; delay = difference. Needs
  RECORD_AUDIO + a quiet room. This is the AmpMe trick in miniature.

**Problem 2 — clock drift.** The two speakers' DAC clocks differ by ~tens of ppm.
After 10 minutes, the follower is a few ms off → smear. Countermeasures:
- cheap: every N seconds, drop/insert ~10 ms into the follower's ring (tiny click, rare)
- proper: nudge the follower track's clock with `right.setPlaybackRate(44100 * (1 ± 0.0005))`
  based on ring-buffer fill level (a poor man's PLL). ~30 lines.

### Reality checks for Door B
- Some OEM audio policies IGNORE setPreferredDevice() for A2DP (both tracks go to the
  active device, or the second connect fails). Test on the target phone early —
  doorto-2-speakers feasibility test is 50 lines; write it before building anything.
- Both speakers must be *connected* first. Connecting a second A2DP device is done in
  system Settings — your app can detect but generally cannot initiate A2DP connects.
- Battery: decoding + 2 streams + BT radios — expect the phone to warm up.

## 3. Door C — capturing OTHER apps' audio (Android 10+)

If you want the two speakers to play whatever Spotify plays (not your own files):

```java
MediaProjectionManager mpm = getSystemService(MediaProjectionManager.class);
// startActivityForResult(mpm.createScreenCaptureIntent()) → user consents
MediaProjection mp = result.getData().getParcelableExtra(EXTRA_MEDIA_PROJECTION);
AudioPlaybackCaptureConfiguration cfg = new AudioPlaybackCaptureConfiguration.Builder(mp)
    .addMatchingUsage(AudioAttributes.USAGE_MEDIA).build();
AudioFormat f = new AudioFormat.Builder().setEncoding(ENCODING_PCM_16BIT)
    .setSampleRate(44100).setChannelMask(CHANNEL_IN_STEREO).build();
AudioRecord rec = new AudioRecord.Builder()
    .setAudioFormat(f)
    .setBufferSizeInBytes(65536)
    .setAudioPlaybackCaptureConfig(cfg).build();
rec.startRecording();
// now read() PCM in a loop and feed BOTH tracks from section 2
```

Caveats: apps can opt out of capture (`setAllowedCapturePolicy`), Spotify currently
ALLOWS it; the projection notification is permanent while capturing; latency grows
(+1 buffer) but it's the same for both speakers so sync between them is unaffected.
This + per-speaker delay + drift PLL = your own AmpMe-style "party mode" that works
with any source — with the caveat it only exists on Android 10+.

## 4. Suggested milestones (do them in order)

- [ ] M0 (50 lines): feasibility probe — connect 2 speakers, two AudioTracks with
      setPreferredDevice, play a test tone to each. THIS tells you if Door B is open
      on your phone. Do not write anything else before this.
- [ ] M1: "Two speakers" card (device list via BluetoothA2dp + button that opens the
      system Media Output panel). Free win on Dual Audio phones.
- [ ] M2: own-file player with split output + your EQ chain attached to the two
      session IDs. Manual delay slider per speaker. Test tone button.
- [ ] M3: drift PLL (setPlaybackRate trim from ring-buffer level).
- [ ] M4: MediaProjection capture (Door C) so any app's audio can be split.
- [ ] M5 (optional, hardest): auto delay measurement with the mic.

## 5. New modules you'll add (file map)

| new file | job |
|---|---|
| `BtDevices.java` | A2DP profile proxy, list sinks, names, BLUETOOTH_CONNECT plumbing |
| `SplitPlayer.java` | decoder (MediaExtractor/MediaCodec) → two ring buffers → two AudioTracks, per-track device + delay + rate trim |
| `SplitCapture.java` | MediaProjection AudioRecord loop (Door C, later) |
| `SyncCalib.java` | chirp player + (later) mic onset detection |
| UI: card + 2 delay sliders | reuse your VerticalSliderView! range 0–500 ms, step 5 ms |
| `Effects.java` change | ensure(Context, int sessionId) overload so the same chain serves session 0 AND your split sessions |

## 6. Glossary for the research you'll do
- A2DP — the Bluetooth profile that streams stereo audio; "sink" = speaker
- AVDTP delay reporting — how a sink tells the phone its buffering latency
- AudioPolicyManager — the system service deciding which device gets which stream
- MediaProjection — the consented screen/audio capture mechanism
- ppm drift — clocks differ; why two DACs can't stay sample-locked for long
- LE Audio / Auracast — the future "correct" way: broadcast to many sinks natively
  (Android 14+ on LE Audio hardware) — worth watching, no hacks needed then
