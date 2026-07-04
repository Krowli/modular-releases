---
name: modular-emulator-android
description: >
  Control an Android emulator / device from inside Modular using the `modular` CLI.
  Use for listing/booting AVDs, taps, swipes, typing, hardware buttons (incl. Back
  and Recents), rotation, app install/launch, runtime permissions, the accessibility
  tree, and logcat — driving a real adb-connected device or emulator. Cross-platform
  (Windows, Linux, macOS). Complements the modular-emulator (iOS) and modular-cli skills.
license: Apache-2.0
---

# Modular Emulator — Android (adb / emulator powered)

Drive an Android emulator or adb-connected device **from within Modular** using
`modular emulator ...` commands. The Android backend shells out to the Android SDK
(`adb`, `emulator`, `avdmanager`) that Android Studio installs, so it works on
Windows, Linux, and macOS — unlike the iOS backend (`modular-emulator`), which is
macOS-only. Device control uses `adb shell input`, so it works without any extra
streaming server.

> **Status:** device discovery + lifecycle + full input/capability control are
> live. The embedded 60fps **visual pane** (scrcpy/H.264) is in development — for
> now, watch the device in Android Studio's emulator window while you drive it
> from the CLI.

## When to use

- List, boot, and target Android emulators/AVDs and physical devices.
- **Tap, swipe, type, press hardware buttons (home/back/recents/power/volume),
  rotate** a running Android device.
- **Install** an APK, **launch** an app, **grant/revoke** runtime permissions.
- Read the **accessibility tree** (`uiautomator`) or capture **logcat**.
- Run an arbitrary `adb shell` command via `exec`.

## When NOT to use

- iOS simulators → use the `modular-emulator` skill (macOS only).
- Building the app → use Gradle / `./gradlew assembleDebug`, then `install`.
- Camera/sensor injection → not supported yet (Android virtual-scene is out of
  scope for now).
- Remote/SSH device control → out of scope; the SDK + device are local to the host.

## Prerequisites (surfaced by Modular)

- **Android Studio / Android SDK** installed, with `ANDROID_HOME` (or
  `ANDROID_SDK_ROOT`) set. Modular also checks the per-OS default location
  (`%LOCALAPPDATA%\Android\Sdk`, `~/Library/Android/sdk`, `~/Android/Sdk`).
- `adb` + `emulator` on the SDK path; at least one **AVD** (create in Android
  Studio ▸ Device Manager) or a connected device with USB debugging.
- A device that is **booted and `adb`-visible** for input/capability commands
  (an AVD that is still shutdown can be listed but must be booted first).

Modular returns a clear message when the SDK is missing
(`Android SDK not found. Install Android Studio and set ANDROID_HOME.`).

## Mental model

```
┌────────────────────────┐
│ modular CLI (agents)      │  e.g. modular emulator tap 0.5 0.7 --device emulator-5554
└───────────┬────────────┘
            │ RPC
            ▼
┌────────────────────────┐   resolves backend by device
│ EmulatorBridge (router)│ ─────────────────────────────► AndroidEmulatorBackend
└────────────────────────┘                                  │ adb / emulator / avdmanager
                                                            ▼
                                                   Android emulator / device
```

Modular owns backend routing and the per-worktree active-device registry. The
Android backend converts Modular's normalized 0–1 coordinates to device pixels and
issues `adb shell input` events; AVD names resolve to running adb serials.

## Common operations

Use `--json` for agent-friendly output. Coordinates are **normalized 0..1**
(top-left origin) — never pixels; Modular converts using the live screen size.

| Goal                       | Command                                                        | Notes |
|----------------------------|----------------------------------------------------------------|-------|
| List devices + AVDs        | `modular emulator devices --json`                                 | Cross-platform; shows iOS + Android with a platform column, booted vs shutdown. |
| Single tap                 | `modular emulator tap <x> <y> --device <serial>`                  | Normalized 0..1. Preferred for single taps. |
| Swipe / gesture            | `modular emulator gesture '<json>' --device <serial>`             | adb approximates the path by its endpoints (start→end). |
| Type text                  | `modular emulator type "user@example.com" --device <serial>`      | US ASCII; spaces handled. No newlines. |
| Hardware button            | `modular emulator button back --device <serial>`                  | home, back, recents, power, volume_up, volume_down. |
| Rotate                     | `modular emulator rotate landscape_left --device <serial>`        | Sets user_rotation (disables auto-rotate). |
| Install an APK             | `modular emulator install ./app-debug.apk --reinstall --device <serial>` | `--reinstall` passes `-r`. |
| Launch an app              | `modular emulator launch com.acme.app --activity .MainActivity --device <serial>` | Omit `--activity` to launch the default LAUNCHER activity. |
| Grant a permission         | `modular emulator permissions grant com.acme.app android.permission.CAMERA --device <serial>` | grant / revoke / reset. |
| Accessibility tree         | `modular emulator ax --device <serial> --json`                    | `uiautomator dump` parsed to a node tree. |
| Logcat (one-shot)          | `modular emulator logcat --lines 200 --device <serial>`           | Dumps recent lines; parsed to entries. |
| Raw adb shell              | `modular emulator exec --command "getprop ro.build.version.sdk" --device <serial>` | Runs `adb -s <serial> shell <command>`. |

## Critical gotchas (teach agents)

- **All coordinates are normalized 0..1** (top-left origin), never pixels — Modular
  scales to the device's live resolution.
- **Target a running device by its adb serial** (e.g. `emulator-5554`) shown in
  `modular emulator devices`. An AVD name resolves only once that AVD is booted.
- The device must be **booted and adb-visible** before input/capability commands;
  a shutdown AVD is listed with `state: shutdown` and must be started first
  (Android Studio, or `emulator @<avd>`).
- `type` uses `adb shell input text` — US ASCII, spaces are handled, newlines are
  not. For unicode-heavy input, use the app UI directly.
- `gesture` is a straight swipe between the first and last point (adb limitation);
  fine for scroll/swipe, not for true multi-touch paths.
- Capability verbs (`install/launch/permissions/ax/logcat`) are **Android-only**;
  running them against an iOS device fails with `emulator_unsupported`.
- No camera/sensor injection yet.

## Targeting devices & worktrees

- Explicit device: `--device <serial>` (recommended for Android today) or an AVD
  name once booted.
- `modular emulator devices` is global (lists every backend's devices); other verbs
  target the resolved device's backend automatically.
- `--worktree <selector>` scopes to a worktree's active device once the
  attach/active flow lands for Android.

## Examples (agent-friendly)

```sh
modular emulator devices --json
modular emulator tap 0.5 0.85 --device emulator-5554 --json
modular emulator type "hello world" --device emulator-5554 --json
modular emulator button recents --device emulator-5554 --json
modular emulator install ./app-debug.apk --reinstall --device emulator-5554 --json
modular emulator launch com.acme.app --device emulator-5554 --json
modular emulator permissions grant com.acme.app android.permission.CAMERA --device emulator-5554 --json
modular emulator ax --device emulator-5554 --json
modular emulator logcat --lines 100 --device emulator-5554 --json
```

## Next action

Run `modular emulator devices --json` to find a booted device, then drive it with
`--device <serial>` while watching the emulator window.

See also: `modular-emulator` (iOS, macOS-only), `modular-cli` (terminals, worktrees,
built-in browser), `computer-use` (desktop UI outside the emulator).
