<!-- Generated: 2026-07-12 | Files scanned: 42 -->
# Architecture

Single-process Windows desktop app (WPF, .NET 8, `net8.0-windows10.0.26100.0`). No
client/server split, no HTTP layer. One `WinExe` project (`MarvinsAIRA.csproj`).
Legacy 1.x codebase, no longer actively developed — see
[reference-architecture.md](../reference-architecture.md).

## Entry point → composition root

```
App.xaml (StartupUri) → App (App.xaml.cs, partial class : Application)
  → single-instance Mutex("MarvinsAIRA Mutex") acquired in ctor
  → MainWindow constructed automatically by WPF
  → MainWindow.Window_Activated → App.Initialize(windowHandle)   [real subsystem startup]
  → MainWindow.Window_Closing  → App.Stop()                       [teardown]
```

`App` is the composition root AND the ambient service locator — no DI container.
`(App)Application.Current` is reached into from UI code, subsystems, and dialogs
throughout. Each subsystem lives in its own `App.<Subsystem>.cs` partial-class file.

## Subsystem map (`App.*.cs` partial-class files)

```
App.xaml.cs          → bootstrap, lifecycle (Initialize/Stop), power-throttling
App.Console.cs       → file + in-UI logging
App.Settings.cs      → settings load/save, debounced serialization
App.Service.cs       → machine ID, self-update check (herboldracing.com)
App.IRacingSDK.cs    → iRacing SDK connection lifecycle, raw telemetry fields
App.Telemetry.cs     → cross-process telemetry export (memory-mapped file)
App.CurrentCar.cs / App.CurrentTrack.cs / App.WetDryCondition.cs
                     → session-identity tracking for FFB-profile keys
App.ForceFeedback.cs → FFB effects engine (largest file, ~1843 lines)
App.Inputs.cs        → joystick/button enumeration and polling
App.LFE.cs           → low-frequency-effect audio → FFB mixing
App.WindSimulator.cs + Arduino.cs + MarvinBox/MarvinBox.ino
                     → wind fan control over serial
App.Logitech.cs + LogitechGSDK.cs → Logitech wheel RPM shift-light LEDs
App.SimagicHPR.cs    → Simagic HPR pedal haptic vibration
App.Voice.cs + SpeechApiReflectionHelper.cs → text-to-speech synthesis
App.Spotter.cs       → car-left/right and session-flag callouts
App.ChatQueue.cs     → spoken-message → in-game chat relay
App.Sounds.cs        → click/ABS sound cues (NAudio)
MainWindow.xaml(.cs), MapButtonWindow, RunInstallerWindow, NewVersionAvailableWindow
                     → UI, main update loop, dialogs (see frontend.md)
```

## Startup sequence

`App.Initialize(windowHandle)` runs subsystem init in this fixed order (creates
`%MyDocuments%\MarvinsAIRA` first); any subsystem exception aborts the whole
startup (try/catch logs then rethrows):

```
Console → Settings → Service → Voice → Sounds → Inputs → ForceFeedback → LFE
  → HPR → IRacingSDK → WindSimulator → Telemetry
```

## Runtime data flow (two independent update cadences)

```
1) MainWindow.UpdateLoop() — dedicated thread, 100ms System.Timers.Timer
   → UpdateSettings → UpdateInputs → UpdateForceFeedback → UpdateWindSimulator
   → UpdateSpotter → UpdateLogitech → ProcessChatMessageQueue
   → Dispatcher.BeginInvoke (telemetry → UI)

2) App.OnTelemetryData — iRacing SDK event callback (~60 Hz, sim tick rate)
   → telemetry decode → UpdateHPR (~20 Hz, every 3rd tick)
   → pretty-graph sampling (~30 Hz, every 2nd tick)
   → UpdateTelemetry (memory-mapped export, every tick)
```

Delta-time for physics/FFB math = SDK tick delta / TickRate, not wall-clock.

## Cross-process export

`App.Telemetry.cs` writes a `TelemetryData` struct into named memory-mapped file
`Local\MAIRATelemetry` every SDK tick, for external readers (overlay, SimHub
plugin) that don't want to talk to the iRacing SDK directly.

## Known quirks (see reference-architecture.md for detail)

- `FFBReceiver.cs` excluded from build (`<Compile Remove>`) — dead vJoy-sniffing code.
- Settings migration no-op bug in `App.Settings.cs`.
- Shock-velocity copy-paste bug in `App.IRacingSDK.cs:400-401` (reads LR twice, never LF).
- Silent failure paths: corrupt `Settings.xml` → defaults; dropped log writes under lock
  contention; Logitech LED support permanently disables itself per-session on any SDK exception.

## Related maps

- [dependencies.md](dependencies.md) — external projects, NuGet packages, external services
- [frontend.md](frontend.md) — WPF window/dialog/tab tree
