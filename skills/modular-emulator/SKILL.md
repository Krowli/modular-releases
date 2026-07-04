---
name: modular-emulator
description: >
  Control a mobile (iOS) emulator / simulator stream from inside Modular using the `modular` CLI.
  Use for taps, gestures, typing, hardware buttons, camera injection, permissions, accessibility tree, and more — all while seeing the live view in Modular's emulator pane.
  Prefer this over raw `npx serve-sim` or direct simctl when running agents inside Modular (the modular surface handles device scoping, helper lifecycle, and worktree context).
  Complements the modular-cli skill for terminals, worktrees, and the built-in browser.
license: Apache-2.0
---

# Modular Emulator (serve-sim powered)

Drive an Apple Simulator (iOS / iPad / Watch) **from within Modular** using `modular emulator ...` commands (or `modular emulator exec` for raw power). This wraps the excellent [serve-sim](https://github.com/EvanBacon/serve-sim) open-source tool so agents get a consistent Modular-native CLI surface, automatic helper management, and seamless integration with Modular's live emulator pane (the visual "preview" surface).

The underlying serve-sim helper captures the real simulator framebuffer (via private SimulatorKit / IOSurface for low-latency 60fps H.264 or MJPEG) and exposes a WebSocket control channel. Modular's bridge owns the helper processes and per-worktree "active emulator" state so unqualified commands "just work" on whatever device/pane is current for the worktree.

## When to use

- The user/agent wants to **tap, swipe, drag, pinch, or press hardware buttons** on a running iOS simulator while seeing the live result in Modular.
- You want **camera injection** (placeholder, webcam, or file loop) for testing camera flows.
- You need to **grant/revoke app permissions** (camera, photos, notifications, location, etc.) or read the **accessibility tree**.
- Rotate the device, simulate memory warnings, toggle CoreAnimation debug overlays, etc.
- You are inside an Modular worktree/terminal and want the emulator to be **workspace-scoped** (like browser tabs) with explicit targeting when needed.
- The agent should use Modular's preview pane instead of external Simulator.app or raw serve-sim URLs.

**When NOT to use**
- Android emulators → use the `modular-emulator-android` skill (same `modular emulator` namespace, cross-platform via adb/emulator).
- Building or installing the app itself → use `xcodebuild`, `xcrun simctl install`, `expo run:ios`, etc. (launch the app, then use `modular emulator` to drive it).
- In-app debugging (state, network, views) → use the app's own tools or the browser pane if it's a webview.
- Remote/SSH worktrees for emulator control (currently out of scope / unsupported; simulator hardware is local to a Mac).

## Prerequisites (enforced / surfaced by Modular)

- macOS host (with Xcode Command Line Tools: `xcrun --version`).
- A booted simulator (`xcrun simctl list devices booted` or let Modular/attach help boot one).
- Node available (for the serve-sim bits; Modular bundles the CLI surface).
- macOS 14+ recommended for full camera injection features.

Modular will give clear errors if these are missing (e.g. "emulator commands require macOS + Xcode tools").

An active emulator "session" for the worktree is required for most commands. Use `modular emulator list` / `attach` or open the emulator pane in the UI.

## Mental model

```
┌────────────────────┐
│ Modular worktree      │
│  - active emulator │◄── modular emulator tap / type / ...
│  - live pane (UI)  │
└─────────┬──────────┘
          │ (registers active stream)
          ▼
┌────────────────────┐   WS / control   ┌─────────────────┐  framebuffer  ┌──────────────┐
│ Modular EmulatorBridge│ ───────────────► │ serve-sim-bin   │ ────────────► │ iOS Simulator│
│ (main process)     │ (or exec serve-sim) (per-device)   │               └──────────────┘
└────────────────────┘                  └─────────────────┘
          ▲
          │ (state + lifecycle)
┌────────────────────┐
│ modular CLI (agents)  │  e.g. modular emulator tap 0.5 0.7
│ modular-emulator skill│
└────────────────────┘
```

Modular owns:
- Starting/stopping the serve-sim helper (via --detach or direct).
- Per-worktree "active" emulator (like active browser tab).
- Explicit targeting with `--worktree`, `--device`, `--emulator <id>`.
- The visual live pane (renderer uses serve-sim-client for the stream).

Agents use the `modular` / `modular-dev` binary (on PATH in Modular terminals; `modular-dev` for dev builds) and never have to manage PIDs, state files in /tmp, or raw WS URLs themselves.

**For `pnpm dev` testing:** run `pnpm build:cli` first (rebuilds the CLI + ensures the `modular-dev` shim points at *this* worktree). Then inside the dev app use `modular-dev emulator ...` (or the direct `./config/scripts/modular-dev.mjs emulator ...` from the repo root). The orchestration preambles and dev launchers automatically select the dev command name so the CLI reaches your in-memory EmulatorBridge / runtime. Plain `modular` reaches a packaged install instead.

## Common operations

Use `--json` for agent-friendly output. Commands are workspace-scoped by default (current worktree's active emulator).

| Goal                        | Command                                      | Notes |
|-----------------------------|----------------------------------------------|-------|
| List available / running   | `modular emulator list [--worktree <sel>]`     | Shows Modular-managed + raw serve-sim streams. Use output for explicit --device/--emulator. |
| Attach / make active       | `modular emulator attach "iPhone 16 Pro" [--worktree <sel>] [--focus]` | Starts helper if needed (serve-sim --detach). Sets active for unqualified commands. --focus optional (does not auto-steal UI focus by default). |
| Single tap                 | `modular emulator tap <x> <y> [--device <id>]` | Normalized 0..1 coords. **Preferred over gesture for simple taps.** |
| Multi-step gesture         | `modular emulator gesture '<json>'`            | See gestures reference (begin/move/end). Use tap for singles. |
| Type text                  | `modular emulator type "text" [--device <id>]` | US ASCII only. Supports stdin/file via exec if needed. |
| Hardware button            | `modular emulator button home [--device <id>]` | home, swipe_home, app_switcher, lock, siri, side_button. |
| Rotate device              | `modular emulator rotate landscape_left`       | Remembers orientation for subsequent gestures. |
| Camera injection           | `modular emulator camera com.acme.App --webcam` | Or --file, placeholder. Hot-swap with switch. May (re)launch app. |
| Permissions                | `modular emulator permissions grant camera com.acme.App` | grant/revoke/reset/list. See full subcommand help. |
| Accessibility tree         | `modular emulator ax [--device <id>]`          | Or via exec for raw endpoint. |
| Raw / advanced             | `modular emulator exec --command "tap 0.5 0.7"` | Or "ca-debug blended on", "memory-warning", full serve-sim subcommands (no "serve-sim" prefix needed in the command string). Bridge injects active device context. |
| Stop                       | `modular emulator kill [--device <id>]`        | Or let pane close / Modular quit clean up. |

Most support `--worktree <selector>` and explicit `--device <udid|name>` or `--emulator <id>` (from list) for targeting.

## Critical gotchas (teach agents)

- **Prefer `tap` over `gesture` for single taps** (same as raw serve-sim). Separate gesture begin/end can be interpreted as long-press due to WS overhead. The modular wrapper uses the reliable quick sequence.
- All coords normalized 0..1 (top-left origin). Never pixels.
- One "active" emulator per worktree for unqualified commands (like active browser tab). Discover ids with `list`, use explicit flags for multi-device or cross-worktree.
- Type = US keyboard only. Unsupported chars error clearly.
- Camera injection often requires (re)launching the target app bundle.
- The visual pane and CLI share the same underlying stream/helper. Closing the pane can stop the stream (configurable).
- Stale helpers / state are cleaned by Modular on quit, but agents should `kill` when done.
- Private APIs under the hood (SimulatorKit etc.) — version sensitive (Xcode updates can affect).

## Targeting devices & worktrees

- Default: current worktree's active emulator (resolved from shell cwd or Modular context).
- Explicit worktree: `--worktree id:abc123` or `--worktree active`.
- Explicit device: `--device "iPhone 16 Pro"` or `--device <udid>` (after `list`).
- Modular-generated emulator id (for stability, like browserPageId): use `--emulator <id>` returned by list (recommended for scripts that persist ids).

`--worktree all` only for listing.

## Integration with the live pane (UI)

- Opening the emulator pane in Modular (or `attach`) makes that stream the "active" one for the worktree → CLI commands target it automatically.
- The pane shows the real 60fps stream (device frame, touch forwarding, toolbar).
- Agents can drive via CLI while the human watches/interacts in the pane.
- No automatic focus steal on CLI attach (use `--focus` if you really want the UI to switch; matches browser behavior).
- Multiple devices: list shows them; pane can grid; CLI uses active or explicit selector.

## Cleanup

```sh
modular emulator kill --device "iPhone 16 Pro"
# or let Modular quit / close the pane
```

Orphans are cleaned by Modular (like agent-browser sessions).

## Examples (agent-friendly)

```sh
modular status --json
modular emulator list --json
modular emulator attach "iPhone 16 Pro" --json
modular emulator tap 0.5 0.8 --json
modular emulator type "user@example.com" --json
modular emulator button home --json
modular emulator camera com.acme.MyApp --file /tmp/test.mp4 --json
modular emulator permissions grant camera com.acme.MyApp --json
modular emulator ax --json
modular emulator exec --command "ca-debug blended on" --json
```

After changes, re-snapshot / wait as needed (analogous to browser snapshot-interact loop).

## Next action

Confirm `modular status --json` and `modular emulator list --json`, then drive the emulator while the live view is visible in Modular.

See also: modular-cli skill (terminals, worktrees, built-in browser), computer-use for desktop outside the simulator.

This skill is the Modular-native replacement for raw serve-sim when you want the visual + control integrated in the IDE.
