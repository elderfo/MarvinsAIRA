# Why Telemetry Sync Works This Way

## The problem

iRacing exposes two very different kinds of data over its SDK: a slow-changing YAML "session info"
block (car, track, drivers, session structure) and a fast numeric telemetry stream (speed, RPM,
forces, positions) updated at the sim's own tick rate. MarvinsAIRA needs to react to both — driving
real-time force feedback off the fast stream, while keying persisted per-car/per-track settings
profiles and announcing session changes off the slow stream — without missing transitions (a new
session starting, a car swap) or wasting CPU re-parsing session info every tick.

## The approach: fully event-driven, with tick-count throttling instead of wall-clock timers

`App.IRacingSDK.cs` never polls iRacing itself — it calls `_irsdk.Start()` once and lets the external
IRSDKSharper library discover the sim process and raise `OnConnected`/`OnDisconnected`/
`OnSessionInfo`/`OnTelemetryData` events. `OnTelemetryData`, which fires at the sim's own tick rate
(typically 60 Hz), effectively **is** the app's telemetry game loop — there's no separate
`Task.Delay`/`Timer` loop pumping telemetry reads.

Within that single callback, several sub-tasks that don't need to run at full tick rate are
throttled by **counting ticks**, not by wall-clock timers: the pretty-graph sampler runs on every 2nd
tick (`tickCount & 1`, ≈30 Hz) and the Simagic HPR update runs every 3rd tick (≈20 Hz). This keeps
everything synchronized to the same clock (the sim's tick) rather than juggling independent timers
that could drift relative to the telemetry they're consuming.

Delta-time for FFB and physics math is computed as **`(tick delta) / TickRate`**, i.e. simulated time
elapsed according to the sim, not measured wall-clock elapsed time. This keeps force calculations
frame-accurate to what the sim itself believes happened, even if the host machine briefly stalls or
the OS scheduler delays the callback slightly — a wall-clock `Stopwatch`-based delta would instead
reflect real elapsed time including any such hiccups, which is the wrong signal for physics math tied
to the sim's own simulated ticks.

## Why session/car/track identity uses string comparison, not SDK events

iRacing's SDK does not raise a dedicated "car changed" or "track changed" event — session info is
just a YAML blob that happens to contain the current car/track/condition names, refreshed on
`OnSessionInfo`. `App.CurrentCar.cs`/`App.CurrentTrack.cs`/`App.WetDryCondition.cs` each hold a "last
known display name" and compare it against the newly parsed value every time session info updates,
setting a changed-flag (`_car_carChanged`, etc.) when it differs. This is a deliberate, low-tech
substitute for an event the SDK doesn't provide: any consumer that cares about a car/track/condition
change (TTS announcements, per-profile settings selection) checks these flags rather than subscribing
to something more specific.

The trade-off: this only detects a change when session info is actually re-parsed. That's why
`_irsdk.PauseSessionInfoUpdates` is deliberately turned off (forcing a fresh parse) whenever
`SessionNum` changes — practice → qualify → race transitions are exactly the moments a car or track
swap is likely, so the code makes sure session info isn't stale exactly when it matters most, while
otherwise leaving parsing paused on-track to save CPU (session info rarely changes mid-session
outside of a session transition).

## Trade-offs

- **Simplicity over precision**: reusing YAML display-name strings as change-detection keys and as
  FFB-profile save keys means a car/track rename upstream (a new iRacing content patch renaming a
  car) silently creates a "new" profile rather than being recognized as the same car — there's no
  stable ID-based keying.
- **Correctness depends on `PauseSessionInfoUpdates` being un-paused at the right moments.** If a
  future change to iRacing added more session-number transitions without also affecting a car/track
  swap (or vice versa), this coupling could miss an update; the code has no independent "did the car
  actually change" verification beyond the string comparison already described.
- **No explicit reconnect/retry state machine.** `OnDisconnected` does a full reset of telemetry
  state, and `OnConnected` presumably re-runs `Update{Car,Track,WetDryCondition}` once fresh data
  arrives — the "reconnect" behavior is really just "state was reset, and normal event flow will
  repopulate it," rather than a dedicated reconnection sequence.

## Related

- [reference-telemetry.md](reference-telemetry.md) — the concrete API surface
- [explanation-architecture.md](explanation-architecture.md) — the broader architectural context
- [tutorial-first-time-setup.md](tutorial-first-time-setup.md) — seeing this sync in action for the first time
