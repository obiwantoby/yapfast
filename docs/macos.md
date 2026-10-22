# Yapfast on macOS

Yapfast runs natively on macOS: an Apple Silicon or Intel binary, a real
`.app` bundle, the standard menu bar, a menu bar status item, and system
notifications. Nothing about the UI is faked — the same Slint interface as
Linux and Windows, in a Mac window with a Mac icon.

A release build on one M3, idle on the linking screen, was reported at 77 MB
of physical footprint, and 87 MB on a second run (`footprint -p`). The README's
40 MB is the Linux claim for doing nothing. The published Discord-only
Linux sample, software renderer, channel open, is about 66 MiB PSS and 91 MiB
RSS at 1120×760. Physical
footprint is a different measure. The window prefers 1120×760. At 2× its
software pixel buffer is 13.0 MiB, and CoreAnimation's share of the window
is in this process.

## Build it

Requires Xcode (or the Command Line Tools), Rust, and a network connection for
the first build, which downloads a prebuilt libwebrtc and prebuilt Skia.

```sh
git clone https://github.com/yapfast/yapfast
cd yapfast
./scripts/build-macos
```

That produces `target/dist/macos/Yapfast.app` and a zipped copy of it. Add
`--fast` for a quicker, unoptimised build while developing, `--dmg` for a disk
image, and `--universal` for a single binary that runs on both Apple Silicon
and Intel (it needs both libwebrtc downloads).

To run it straight out of the target directory without packing a bundle:

```sh
source scripts/macos-env.sh
cargo run --no-default-features --features desktop,discord
```

`scripts/macos-env.sh` points `LK_CUSTOM_WEBRTC` at one libwebrtc under
`.local/`. `scripts/fetch-libwebrtc-mac` unpacks it before Cargo runs. The
Go Live codec shims compile against that tree and do not download it.
`webrtc-sys-build` locks its own download, which does not fill the directory
for a build script that only reads it.

The first launch from a download asks macOS's permission for the microphone
when you join a call. Camera access is asked for when you turn a camera on.

## What's Mac-specific

- **Menu bar.** Application menu with About, Settings…, Hide and Quit; an Edit
  menu so the standard Cut, Copy, Paste, Undo and Select All shortcuts work in
  the message box; a Window menu. Settings… and Quit carry ⌘, and ⌘Q.
- **Status item.** The Yapfast mark in the menu bar, drawn as a template so it
  follows the light and dark appearance. Clicking it opens the window;
  right-clicking gives Open, Disconnect when a call is up, and Quit.
- **Closing the window hides it.** Yapfast keeps running in the menu bar.
  Clicking the Dock icon brings the window back, the same job the tray icon
  does on Linux and Windows.
- **Notifications** go through `UserNotifications`, and clicking one opens the
  conversation it came from.
- **Start at sign-in** uses `SMAppService`, so the switch in
  **Settings → Advanced** registers Yapfast as a login item rather than
  writing a launch agent.
- **Paste from Finder** reads the file URLs on the pasteboard, so copying a
  file in Finder and pasting into a message attaches it.
- **H.264 decodes through VideoToolbox.** The archive's FFmpeg carries no
  video decoders on the Mac, so `native/videotoolbox.c` decodes H.264 for both
  Go Live streams and video files, and takes the hardware's NV12 apart into
  the I420 planes the rest of Yapfast works in. OpenH264 is in the archive, but
  only its encoder, so there was nothing else to use.
- **Audio** uses a CoreAudio HAL unit for both playback and recording, at the
  device's own rate and channel count, resampled into and out of the 48 kHz
  stereo rings the rest of Yapfast uses. WebRTC still mixes, echo-cancels and
  encodes; this only moves samples.

## Known gaps

- **AAC sound plays silently.** The archive's FFmpeg decodes FLAC, MP3, Opus,
  PCM and Vorbis, and not AAC, so an MP4 whose sound is `mp4a` — a screen
  recording, a clip from a phone — plays without sound. Opening it says so by
  giving none, and the picture plays on regardless. Two tests fail for this,
  both on `video-aac.mp4`: `video::tests::plays_pictures_and_sound_and_moves_back`
  and `video_player::tests::plays_a_video_as_it_downloads`. The Mac has AAC:
  AudioToolbox's `AudioConverter`, behind the sound path the way
  `native/videotoolbox.c` is behind the picture one.
- **Go Live shares the camera, not the whole screen.** Screen CaptureKit
  capture isn't wired up yet, so screen sharing in a Go Live call doesn't
  offer the Mac's display.
- **Notifications need the bundle.** `UserNotifications` requires a real
  `CFBundleIdentifier`, so notifications don't arrive when running the bare
  `cargo run` binary. Use `target/dist/macos/Yapfast.app`.
- **Keychain prompts while re-building.** Yapfast's Discord token is stored in
  the keychain under an access list bound to the code-signing identity, and
  local builds are ad-hoc signed, so a new build asks to use the item the
  previous one stored. "Always Allow" clears it until the next rebuild. A
  Developer ID signature makes this a one-time prompt.
- **A duplicate JPEG symbol at the link.** Skia and libwebrtc each carry a
  libjpeg, and both define `jpeg_natural_order`, so every link warns and keeps
  one of them. The table is the JPEG standard's zigzag order, the same 64
  bytes either way, so nothing can tell which it got. Windows has the same
  collision and renames Skia's copy in `nix/prepare-windows-skia.py`; macOS
  has no such step yet and the warning is safe to ignore.
- **`__eh_frame` is over the compact unwind limit.** The same link reports
  that section too large for the compact unwind table, so unwinding falls back
  to DWARF at runtime. Nothing fails; it's the cost of libwebrtc, Skia and the
  noise-suppression models in one binary.
- **Two checks still fail for reasons of their own.**
  `custom_emoji::tests::skia_draws_the_glyphs_in_colour` can't get a typeface
  out of `FontMgr::new().new_from_data` on macOS, so the generated colour
  emoji font is never drawn in the test. And
  `ui::tests::the_composers_menu_cuts_copies_and_pastes_through_the_application`
  finds the draft unchanged after Select All then Copy, so what Copy puts on
  the pasteboard in a headless test never comes back.
- **Releases aren't notarized here.** The build ad-hoc signs so Gatekeeper
  accepts the app on the machine that built it. Distributing it needs a
  Developer ID and `--notarize`-style notarization, which needs an Apple
  developer account.
- **Intel is untested.** Everything is built and checked on Apple Silicon;
  `--universal` and the `x86_64-apple-darwin` target are wired up but nobody
  has run them.
- **The development scripts still assume Nix.** `scripts/check`,
  `scripts/dev` and the test stack want the Linux toolchain. Building and
  running the app on macOS doesn't.
