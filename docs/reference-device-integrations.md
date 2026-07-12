# Wheel Device Integrations Reference

## What it is

Two independent, per-vendor hardware-integration modules let MarvinsAIRA drive third-party
peripherals from live iRacing telemetry: **Logitech** RPM shift-light LEDs (`App.Logitech.cs`,
`LogitechGSDK.cs`) and **Simagic HPR** pedal haptic vibration (`App.SimagicHPR.cs`). They are
hand-written, unrelated modules — there is no shared `IWheelDevice`/`IHapticDevice` abstraction,
consistent with this codebase's general pattern of one `App.<Feature>.cs` file per integration
(see [reference-architecture.md](reference-architecture.md)).

## Logitech RPM shift-light LEDs

### API / Interface

- `LogitechGSDK.cs:9` — `static extern bool LogiPlayLedsDInput(nint deviceHandle, float currentRPM, float rpmFirstLedTurnsOn, float rpmRedLine)` — P/Invoke into `LogitechSteeringWheelEnginesWrapper.dll` (Cdecl, Unicode), the vendor's native Logitech G SDK wrapper.
- `App.Logitech.cs:10` — `UpdateLogitech()`, the per-tick entry point. Guarded by
  `Settings.ControlRPMLights`, `_irsdk_connected`, `_ffb_drivingJoystick != null`, and a
  `_logitech_disabled` latch (`App.Logitech.cs:8`).

### Dependencies

The native DLL must be present and the Logitech G SDK installed on the user's machine. Reads
`_irsdk_rpm` and shift-light RPM thresholds from telemetry, and `Settings.ControlRPMLights`
(`Settings.cs:3752`, default `true`).

### Edge cases

No device-presence check beyond `_ffb_drivingJoystick != null`. Any DLL-missing or SDK-not-installed
condition throws; it's caught generically and the feature **permanently disables itself for the
session** (`_logitech_disabled = true`, `App.Logitech.cs:19-31`) after one log line — there is no
retry within a session.

## Simagic HPR pedal haptics

"HPR" pedal hardware provides 3-channel (clutch/brake/throttle) vibration motors. `App.SimagicHPR.cs`
generates 9 candidate haptic "effects" — gear change, ABS, wide RPM band, narrow RPM band, steering
understeer, steering oversteer, wheel lock, wheel spin, clutch slip — each with a frequency
(15–35 Hz, `HPR_MIN/MAX_FREQUENCY`) and amplitude (18–60, `HPR_MIN/MAX_AMPLITUDE`). Per pedal channel,
up to 3 user-configured effect slots (priority order) are read from Settings, and one On/Off
vibration command is sent per channel per tick.

### API / Interface

- `InitializeHPR(bool isFirstInitialization = false)` (`:55`), `UninitializeHPR()` (`:83`, private),
  `StopHPR()` (`:90`, private), `ResetHPR()` (`:95`, private), `UpdateHPR()` (`:105`, private),
  `TogglePedalHapticsPrettyGraph()` (`:76`, public — toggles `_hpr_drawPrettyGraphs`).
- Public fields: `_hpr_writeableBitmaps[3]`, `_hpr_pixels[3][]`, `_hpr_frequency[3]`,
  `_hpr_amplitude[3]`, `_hpr_averageRpmSpeedRatioPerGear[13]` (a per-gear rolling RPM/speed-ratio
  baseline, `:40`), plus `HPR_WRITEABLE_BITMAP_*` / `HPR_PIXELS_BUFFER_*` constants (`:18-27`).

### Dependencies

External sibling project `Simagic` namespace (`using Simagic;`, `App.SimagicHPR.cs:6`), referenced
via `..\SimagicHPR\SimagicHPR.csproj` — **not in this repo**
(see [reference-architecture.md](reference-architecture.md#external-dependencies-not-in-this-repo)).
Types used: `HPR` class (single instance, `:29`), `.Initialize()`, `.Uninitialize()`,
`.VibratePedal(HPR.Channel, HPR.State, float frequency, float amplitude)`, enum
`HPR.Channel {Clutch, Brake, Throttle}` (matches pedal index `i`), enum `HPR.State {On, Off}`.

Settings consumed: `PedalHapticsEnabled` (`Settings.cs:1513`), `SteeringEffectsEnabled`,
`USEffectStyle`/`OSEffectStyle`/`USEffectStrength`/`OSEffectStrength`, and per-pedal
`PedalHaptics{Clutch,Brake,Throttle}Effect{1,2,3}` + matching `EffectStrength{1,2,3}` (`:325-331`).

Also reads live telemetry/FFB state: `_irsdk_rpm`, `_irsdk_gear`, `_irsdk_brake`, `_irsdk_clutch`,
`_irsdk_throttle`, `_irsdk_velocityX`, `_irsdk_brakeABSactive`, `_irsdk_shiftLightsShiftRPM`,
`_ffb_understeerAmount`, `_ffb_oversteerAmount`, `_ffb_drivingJoystick`.

### Dependents

- `MainWindow.xaml.cs:567` calls `app.UpdateLogitech()` in the main UI-loop update cycle.
- `App.IRacingSDK.cs:571` calls `UpdateHPR()`, throttled to every 3rd telemetry tick (~20 Hz);
  `App.IRacingSDK.cs:301` calls `ResetHPR()` on sim disconnect.
- `App.xaml.cs:69,92` calls `InitializeHPR(true)`/`StopHPR()` during app-level `Initialize`/`Stop`.
- `MainWindow.xaml.cs:1513` re-inits HPR from a checkbox handler; `:1517-1539` toggles the pretty-graph
  debug view; `:2050` `UpdatePedalHapticsPrettyGraphs()` renders `_hpr_pixels` into the
  `PedalHapticsClutch/Brake/Throttle_Image` WriteableBitmaps bound in `MainWindow.xaml`.

### Edge cases

`_hpr.Uninitialize()` is called unconditionally before every (re-)`Initialize()` (`:68-73`), so
toggling the feature off then on cleanly resets state. There's no visible try/catch around
`_hpr.Initialize()`/`VibratePedal` — the code assumes `SimagicHPR` internally tolerates a missing or
disconnected device. Off-track or in replay mode forces all 3 channels off (`:117-124`). No
multi-device selection — a single global `HPR` instance/handle.

## Related

- [reference-force-feedback.md](reference-force-feedback.md) — `_ffb_understeerAmount`/`_ffb_oversteerAmount` computed there feed HPR effect selection
- [reference-telemetry.md](reference-telemetry.md) — telemetry fields both integrations read
- [reference-settings.md](reference-settings.md) — the effect-slot and strength settings
