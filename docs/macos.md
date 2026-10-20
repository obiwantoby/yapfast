# Yapfast on macOS

Yapfast runs natively on macOS: an Apple Silicon or Intel binary, a real
`.app` bundle, the standard menu bar, a menu bar status item, and system
notifications. Nothing about the UI is faked — the same Slint interface as
Linux and Windows, in a Mac window with a Mac icon.

A release build on Apple Silicon sits at about 80 MB of physical footprint
on the linking screen, measured with `footprint -p`. The README's 40 MB is
the Linux number. On a Mac the retina framebuffer and CoreAnimation are in
this process.

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

`scripts/macos-env.sh` points `LK_CUSTOM_WEBRTC` at the libwebrtc under
`.local/`, so the build scripts don't race each other to download it.

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
- **Audio** uses a CoreAudio HAL unit for both playback and recording, at the
  device's own rate and channel count, resampled into and out of the 48 kHz
  stereo rings the rest of Yapfast uses. WebRTC still mixes, echo-cancels and
  encodes; this only moves samples.

## Known gaps

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
