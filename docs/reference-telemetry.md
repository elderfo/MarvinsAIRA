# iRacing SDK & Telemetry Reference

## What it is

`App.IRacingSDK.cs`, `App.Telemetry.cs`, `App.CurrentCar.cs`, `App.CurrentTrack.cs`, and
`App.WetDryCondition.cs` connect MarvinsAIRA to the running iRacing sim process, decode its
telemetry and session-info streams, and track session-identity state (current car / track /
wet-dry condition) used to key per-profile force-feedback settings.

## API / Interface

### Connection lifecycle

- `App._irsdk` (`App.IRacingSDK.cs:24`) — the `IRacingSdk` instance (from the external `IRSDKSharper`
  project — see [reference-architecture.md](reference-architecture.md#external-dependencies-not-in-this-repo)).
- `App.InitializeIRacingSDK()` / `App.StopIRacingSDK()` (`App.IRacingSDK.cs:132`, `:149`) — called
  from `App.Initialize`/`App.Stop`.
- `_irsdk.Start()` is fire-and-forget — IRSDKSharper internally polls for the sim process and raises:
  - `OnConnected` / `OnDisconnected` — full state reset on disconnect (~40 fields plus shock-velocity
    arrays cleared, ABS sound stopped, car/track/wet-dry state recomputed so the UI reverts to
    "No Car"/"No Track").
  - `OnSessionInfo` — slow-changing YAML block (car/track/driver metadata).
  - `OnTelemetryData` — the sim's per-tick numeric channel update (typically 60 Hz); this is the
    app's de-facto game loop for telemetry-driven work.
  - `OnException`, `OnDebugLog`.

### Decoded telemetry fields (the module's real public surface)

Rather than accessor methods, this subsystem exposes decoded state as public mutable fields on
`App`, consumed directly by sibling partials: `_irsdk_connected`, `_irsdk_isOnTrack`,
`_irsdk_onPitRoad`, `_irsdk_playerCarIdx`, `_irsdk_playerTrackSurface`, `_irsdk_sessionNum`,
`_irsdk_gForce`, `_irsdk_velocity` (and `_irsdk_velocityX/Y`), `_irsdk_brake`/`_irsdk_throttle`/
`_irsdk_clutch`/`_irsdk_rpm`/`_irsdk_gear`/`_irsdk_speed`, `_irsdk_steeringWheelAngle(Max)`,
per-corner shock-velocity arrays, `_irsdk_steeringWheelTorque_ST[6]` (360 Hz torque buffer),
`_irsdk_yawRate`, `_irsdk_brakeABSactive`, `_irsdk_simMode`, `_irsdk_playerCarNumber`,
`_irsdk_carLeftRight`, `_irsdk_sessionFlags`, shift-light RPM thresholds
(`_irsdk_shiftLightsShiftRPM` and related first/blink RPM fields), all in `App.IRacingSDK.cs:64-126`.

### Session-identity tracking

- `App.CurrentCar.cs`: `UpdateCurrentCar()` (private) → `_car_currentCarScreenName`;
  `UpdateCarSaveName()` (public) → `_car_carSaveName`, the key used for per-car FFB profiles.
- `App.CurrentTrack.cs`: `UpdateCurrentTrack()` (private) → `_track_currentTrackDisplayName`;
  `UpdateTrackSaveName()` / `UpdateTrackConfigSaveName()` → `_track_trackSaveName` /
  `_track_trackConfigSaveName`.
- `App.WetDryCondition.cs`: `UpdateCurrentWetDryCondition()` (private) →
  `_wetdry_currentConditionDisplayName`; `UpdateWetDryConditionSaveName()` →
  `_wetdry_conditionSaveName`.
- Constants `ALL_CARS_SAVE_NAME` / `ALL_TRACKS_SAVE_NAME` / `ALL_TRACK_CONFIGS_SAVE_NAME` /
  `ALL_WETDRYCONDITIONS_SAVE_NAME` = `"All"` — the fallback profile key when per-car/track/condition
  scoping is turned off in Settings.

### Cross-process export

- `App.Telemetry.cs`: `UpdateTelemetry()` writes a `TelemetryData` struct into a named memory-mapped
  file (`Local\MAIRATelemetry`) on every `OnTelemetryData` callback, so out-of-process consumers
  (e.g. an overlay, or the separate SimHub plugin) can read live state.

## External types used (IRSDKSharper — not in this repo)

`IRacingSdk`: events `OnException`, `OnConnected`, `OnDisconnected`, `OnSessionInfo`,
`OnTelemetryData`, `OnDebugLog`; methods `Start()`, `Stop()`; properties `IsStarted`, `IsConnected`,
`Data`, `PauseSessionInfoUpdates` (settable), `TickRate`, `TickCount`.

`IRacingSdkData` (via `.Data`): `SessionInfo` (strongly-typed YAML, e.g.
`WeekendInfo.{TrackDisplayName,TrackConfigName,SimMode}`,
`DriverInfo.{Drivers,DriverCarIdx,DriverCarSLFirstRPM,DriverCarSLShiftRPM,DriverCarSLBlinkRPM,DriverCarGearNumForward}`),
`TelemetryDataProperties` (name→`IRacingSdkDatum` lookup, supports indexer and `TryGetValue`), and
typed getters `GetFloat/GetBool/GetInt/GetBitField/GetFloatArray(datum, ...)`.

`IRacingSdkEnum`: `CarLeftRight`, `TrkLoc` (includes `NotInWorld`, `OffTrack`), `Flags`.

## Examples

Datum-caching pattern used throughout `App.IRacingSDK.cs` — telemetry variable handles are looked up
by name once and reused every tick to avoid per-frame dictionary lookups:

```csharp
if (!_irsdk_telemetryDataInitialized)
{
    _irsdk_rpmDatum = _irsdk.Data.TelemetryDataProperties["RPM"];
    // ... more datum lookups, once
    _irsdk_telemetryDataInitialized = true;
}
_irsdk_rpm = _irsdk.Data.GetFloat(_irsdk_rpmDatum);
```

Session-transition handling — session-info parsing is paused while on track (to save CPU) but
forcibly resumed on a session-number change so a new session's car/track data is picked up promptly:

```csharp
if (sessionNum != _irsdk_sessionNum)
{
    _irsdk.PauseSessionInfoUpdates = false; // force a fresh SessionInfo parse
}
```

## Dependents

- `App.ForceFeedback.cs` is the heaviest consumer (G-force, on-track/pit-road state, track surface,
  sim mode, car/track/wet-dry save-names for profile selection).
- `App.Settings.cs` persists FFB/steering-effect settings keyed by car/track/wet-dry save-names.
- `App.WindSimulator.cs`, `App.Spotter.cs`, `App.SimagicHPR.cs` gate on `_irsdk_isOnTrack`/connected.
- `App.Logitech.cs`, `App.ChatQueue.cs` gate on `_irsdk_connected`.
- `MainWindow` reads these fields directly for the status bar and the live "Skid Pad" telemetry
  dashboard.

## Edge cases

- Telemetry datum lookups use `TryGetValue` for car-dependent channels (shock velocity, torque) but
  a throwing indexer for core channels assumed always present.
- A known bug: `_irsdk_lfShockVel_STDatum` is populated from `"LRshockVel_ST"` twice
  (`App.IRacingSDK.cs:400-401`) instead of `"LFshockVel_ST"` — left-front shock velocity is never
  actually captured.
- `OnTelemetryData` throttles sub-tasks by tick-count parity rather than wall-clock timers: pretty
  graph sampling at ~30 Hz (`tickCount & 1`), HPR update at ~20 Hz (3-tick accumulator).
- Delta-time for physics/FFB math is `(tick delta) / TickRate`, not measured wall-clock time.
- SDK callbacks arrive on a background thread; UI-facing updates are marshaled via
  `Dispatcher.BeginInvoke`.

## Related

- [reference-architecture.md](reference-architecture.md) — overall startup/update-loop context
- [reference-force-feedback.md](reference-force-feedback.md) — the primary consumer of this telemetry
- [explanation-telemetry-sync.md](explanation-telemetry-sync.md) — why this subsystem is event-driven
  and uses string-identity change detection instead of SDK-provided change events
- [tutorial-first-time-setup.md](tutorial-first-time-setup.md) — confirming a live telemetry connection for the first time
