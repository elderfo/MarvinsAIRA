# Architecture Reference

MarvinsAIRA is a Windows desktop app (.NET 8, WPF, `net8.0-windows10.0.26100.0`) that connects to the
iRacing sim, reads live telemetry, and drives a wheel's force feedback plus optional peripherals
(Arduino-based cooling fans, Simagic HPR pedal haptics, Logitech shift-light LEDs) and voice/audio
feedback (spotter callouts, TTS, sound cues).

> **Status:** this is the legacy 1.x codebase and is no longer actively developed. The maintained
> successor is [MAIRA Refactored (2.x)](https://github.com/mherbold/MarvinsAIRARefactored). This
> documentation describes the code as it exists in this repo, including known dead code and latent
> bugs — see [Known quirks](#known-quirks-and-dead-code) below and
> [explanation-architecture.md](explanation-architecture.md).

## Entry point and process model

- Single WinExe project, `MarvinsAIRA.csproj`, `OutputType=WinExe`, `UseWPF=true`.
- WPF startup: `App.xaml` declares `StartupUri="MainWindow.xaml"`, so WPF constructs `MainWindow`
  automatically after `App`'s constructor runs.
- `App` (`App.xaml.cs:10`) is a `partial class App : Application`. There is no dependency-injection
  container — `App` itself is the composition root, and `(App)Application.Current` is the ambient
  service locator used throughout the codebase (UI code, subsystems, dialogs all reach into it).
- Single-instance enforcement: a named `Mutex("MarvinsAIRA Mutex")` is acquired in the `App()`
  constructor (`App.xaml.cs:19,25`). If another instance already holds it, the new process
  foregrounds the existing window and calls `Environment.Exit(0)` — this runs before any WPF
  machinery starts and is wrapped in a bare `try/catch {}`.
- Real subsystem startup happens later than the constructor: `MainWindow.Window_Activated`
  (`MainWindow.xaml.cs:198`) calls `App.Initialize(nint windowHandle)` once a real HWND exists.
  Teardown happens in `MainWindow.Window_Closing`, which calls `App.Stop()`.

## The `App.*.cs` partial-class pattern

The whole application is organized as **one `App.<Subsystem>.cs` file per subsystem**, all
contributing to the same `partial class App`. There is no interface/abstraction layer between
subsystems — they communicate through shared fields and direct method calls on the single `App`
instance.

| File | Subsystem | Reference doc |
|---|---|---|
| `App.xaml.cs` | Bootstrap, lifecycle (`Initialize`/`Stop`), power-throttling | this doc |
| `App.Console.cs` | File + in-UI logging (`WriteLine`) | this doc |
| `App.Settings.cs` | Settings load/save orchestration, debounced serialization | [reference-settings.md](reference-settings.md) |
| `App.Service.cs` | Per-machine ID, self-update check against herboldracing.com | [reference-ui.md](reference-ui.md) |
| `App.IRacingSDK.cs` | iRacing SDK connection lifecycle, raw telemetry fields | [reference-telemetry.md](reference-telemetry.md) |
| `App.Telemetry.cs` | Cross-process telemetry export (memory-mapped file) | [reference-telemetry.md](reference-telemetry.md) |
| `App.CurrentCar.cs` / `App.CurrentTrack.cs` / `App.WetDryCondition.cs` | Session-identity tracking for FFB-profile keys | [reference-telemetry.md](reference-telemetry.md) |
| `App.ForceFeedback.cs` | FFB effects engine (largest file, 1843 lines) | [reference-force-feedback.md](reference-force-feedback.md) |
| `App.Inputs.cs` | Joystick/button enumeration and polling | [reference-force-feedback.md](reference-force-feedback.md) |
| `App.LFE.cs` | Low-frequency-effect audio → FFB mixing | [reference-force-feedback.md](reference-force-feedback.md) |
| `App.WindSimulator.cs` / `Arduino.cs` / `MarvinBox/MarvinBox.ino` | Wind fan control over serial | [reference-force-feedback.md](reference-force-feedback.md) |
| `App.Logitech.cs` / `LogitechGSDK.cs` | Logitech wheel RPM shift-light LEDs | [reference-device-integrations.md](reference-device-integrations.md) |
| `App.SimagicHPR.cs` | Simagic HPR pedal haptic vibration | [reference-device-integrations.md](reference-device-integrations.md) |
| `App.Voice.cs` / `SpeechApiReflectionHelper.cs` | Text-to-speech synthesis | [reference-voice-audio.md](reference-voice-audio.md) |
| `App.Spotter.cs` | Car-left/right and session-flag callouts | [reference-voice-audio.md](reference-voice-audio.md) |
| `App.ChatQueue.cs` | Spoken-message → in-game chat relay | [reference-voice-audio.md](reference-voice-audio.md) |
| `App.Sounds.cs` | Click and ABS sound cues (NAudio) | [reference-voice-audio.md](reference-voice-audio.md) |
| `MainWindow.xaml(.cs)`, `MapButtonWindow`, `RunInstallerWindow`, `NewVersionAvailableWindow` | UI, main update loop, dialogs | [reference-ui.md](reference-ui.md) |

## Startup sequence

`App.Initialize(windowHandle)` (`App.xaml.cs:50-102`) runs subsystem init in this fixed order,
creating `%MyDocuments%\MarvinsAIRA` first:

```
Console → Settings → Service → Voice → Sounds → Inputs → ForceFeedback → LFE → HPR → IRacingSDK → WindSimulator → Telemetry
```

Each step is wrapped in a try/catch that logs then rethrows (`App.xaml.cs:74-80`), so a failure in
any one subsystem still aborts startup rather than degrading gracefully.

## Main update loop

There is no single "game loop" class — two independent update cadences drive the app:

1. **`MainWindow.UpdateLoop()`** (`MainWindow.xaml.cs:537`) — a dedicated thread gated by an
   `AutoResetEvent` and a 100 ms `System.Timers.Timer`. Each tick calls, in order: `UpdateSettings`,
   `UpdateInputs`, `UpdateForceFeedback`, `UpdateWindSimulator`, `UpdateSpotter`, `UpdateLogitech`,
   `ProcessChatMessageQueue`, then marshals telemetry to the UI via `Dispatcher.BeginInvoke`.
2. **`App.OnTelemetryData`** (`App.IRacingSDK.cs`) — fires at the sim's own tick rate (typically
   60 Hz) via an IRSDKSharper event callback. This is where telemetry decoding, `UpdateHPR` (throttled
   to every 3rd tick, ≈20 Hz), pretty-graph sampling (every 2nd tick, ≈30 Hz), and `UpdateTelemetry`
   (memory-mapped export) happen.

Delta-time for physics/FFB math is computed from **SDK tick deltas divided by `TickRate`**, not
wall-clock time, keeping calculations frame-accurate to the sim rather than to real time.

## External dependencies not in this repo

`MarvinsAIRA.csproj` references two sibling projects by relative path that are **not part of this
repository** and must be checked out alongside it to build:

- `..\IRSDKSharper\IRSDKSharper.csproj` — wraps iRacing's shared-memory telemetry SDK; provides the
  `IRacingSdk` class consumed throughout `App.IRacingSDK.cs`.
- `..\SimagicHPR\SimagicHPR.csproj` — wraps Simagic's pedal-haptics hardware SDK; provides the `HPR`
  class consumed in `App.SimagicHPR.cs`.

The solution file (`MarvinsAIRA.sln`) also references a third sibling, `..\MarvinsAIRASimHub`, a
SimHub plugin (`MarvinsAIRASimHub.dll`) that `InstallScript.iss` copies into
`{userdocs}\MarvinsAIRA` — also not part of this repo.

Notable third-party packages (see `MarvinsAIRA.csproj` for full/versioned list): `SharpDX.DirectInput`
/ `SharpDX.DirectSound` (wheel FFB, joystick input, LFE audio capture), `NAudio` (sound cues, ABS
pitch-shifting), `System.Speech` (TTS), `System.IO.Ports` (Arduino serial), `Newtonsoft.Json` +
`System.Net.Http` (update-check API), `ModernWpfUI` / `Extended.Wpf.Toolkit` (UI controls),
`Hardcodet.NotifyIcon.Wpf` (system tray), `BootMeUp` (auto-start registry entry).

## Cross-process telemetry export

`App.Telemetry.cs` writes a `TelemetryData` struct into a named memory-mapped file
(`Local\MAIRATelemetry`) on every SDK telemetry tick, so external tools (e.g. an overlay or the
SimHub plugin) can read live state without going through the iRacing SDK themselves.

## Known quirks and dead code

Documented here rather than silently omitted, since this is frozen legacy code future readers may
still need to reason about:

- **`FFBReceiver.cs` is excluded from the build** (`<Compile Remove="FFBReceiver.cs" />` in
  `MarvinsAIRA.csproj:16`). It's dead code in namespace `MarvinsIRFFB` that sniffs vJoy FFB HID
  reports for debugging — superseded once the app switched to reading iRacing telemetry directly.
  `vJoyInterfaceWrap.dll` is still shipped as content but unused by any compiled code. See
  [explanation-force-feedback-design.md](explanation-force-feedback-design.md).
- **Settings migration no-op bug**: a one-time migration block in `App.Settings.cs` (marked
  `TEMPORARY CODE - GET RID OF THIS AFTER ABOUT A MONTH`) mutates the default `Settings` instance
  before it gets overwritten by the loaded one, so the migration likely never takes effect.
- **Shock-velocity copy-paste bug**: `App.IRacingSDK.cs:400-401` reads `"LRshockVel_ST"` twice
  instead of `"LFshockVel_ST"` — left-front shock velocity telemetry is never actually captured.
- **Silent failure paths**: a corrupt `Settings.xml` is discarded silently (falls back to defaults);
  log writes under lock contention are dropped silently; Logitech LED support permanently disables
  itself for the session on any SDK exception with no retry.

## Related documentation

- [reference-settings.md](reference-settings.md) — Settings object and persistence
- [reference-telemetry.md](reference-telemetry.md) — iRacing SDK connection and telemetry
- [reference-force-feedback.md](reference-force-feedback.md) — FFB effects, inputs, wind simulator
- [reference-device-integrations.md](reference-device-integrations.md) — Logitech, Simagic HPR
- [reference-voice-audio.md](reference-voice-audio.md) — Voice, spotter, chat, sound cues
- [reference-ui.md](reference-ui.md) — MainWindow, dialogs, distribution
- [explanation-architecture.md](explanation-architecture.md) — why the codebase is shaped this way
- [tutorial-build-and-run-from-source.md](tutorial-build-and-run-from-source.md) — build this codebase and verify a change end-to-end
