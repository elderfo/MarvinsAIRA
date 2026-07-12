# World Model

MarvinsAIRA is a Windows desktop WPF companion app for the iRacing sim: it reads live iRacing
telemetry and drives a direct-drive/wheel's force feedback plus optional peripherals (Arduino wind
fans, Simagic HPR pedal haptics, Logitech shift-light LEDs) and voice/audio feedback (TTS spotter
callouts, chat relay, sound cues). It is the **legacy 1.x codebase and is no longer actively
developed** — the maintained successor is a separate repo,
[MAIRA Refactored (2.x)](https://github.com/mherbold/MarvinsAIRARefactored). This file exists so
that any future contributor or delegated worker who picks up a unit of work here with no spec has
enough standing context — vocabulary, architecture, constraints, landmines — to make small,
in-scope decisions without guessing or re-deriving it from source.

## Holds

### Domain vocabulary / ubiquitous language

- **FFB** — force feedback: the computed torque signal sent to the wheel's motor via DirectInput.
- **LFE** — "low-frequency effect": an audio-derived low-frequency signal captured from a virtual
  audio cable device and mixed additively into the FFB torque (`App.LFE.cs`), gated by
  `Settings.LFEToFFBEnabled`.
- **HPR** — Simagic's pedal-haptics hardware line ("Simagic HPR"); `App.SimagicHPR.cs` drives 3-channel
  (clutch/brake/throttle) vibration motors via the external `HPR` class (sibling project
  `..\SimagicHPR`).
- **Spotter** — the subsystem (`App.Spotter.cs`) that announces car-left/car-right proximity and
  session-flag changes (yellow, checkered, etc.) via TTS/chat.
- **Save name** — the string key (`_car_carSaveName`, `_track_trackSaveName`,
  `_track_trackConfigSaveName`, `_wetdry_conditionSaveName`) used to scope a persisted
  `ForceFeedbackSettings`/`SteeringEffectsSettings` entry to a specific car/track/track-config/
  wet-dry condition. Constants `ALL_CARS_SAVE_NAME`/`ALL_TRACKS_SAVE_NAME`/etc. (`"All"`) are the
  fallback key when a given scoping toggle is off in Settings.
- **Understeer/oversteer effect style** — one of 4 selectable FFB waveforms (sine, ramp,
  steady-state-scaled, static) driven by a yaw-rate factor (understeer) or lateral velocity
  (oversteer); the same computed amounts (`_ffb_understeerAmount`/`_ffb_oversteerAmount`) also drive
  HPR pedal-haptic effect selection.
- **Soft lock** — a synthetic end-stop torque effect near maximum steering angle.
- **Crash protection** / **curb protection** — temporary reductions of overall/detail FFB scale
  triggered by a G-force spike or a shock-velocity spike (curb strike), respectively.
- **Cooldown** — the FFB ramp-to-zero state machine (~1s) that fires when leaving the track, to avoid
  a torque jolt ("sore thumbs prevention").
- **Pretty graph** — the oscilloscope-style `WriteableBitmap` visualization of FFB input/output
  waveform (and, separately, per-pedal HPR waveforms) rendered on the UI tabs.
- **`_ST` suffix** — telemetry fields sampled at the sim's high-rate "sub-tick" buffer (e.g.
  `_irsdk_steeringWheelTorque_ST[6]` at 360 Hz, shock-velocity `_ST` arrays), distinct from the
  once-per-tick (~60 Hz) scalar telemetry fields.
- **Tick-count throttling** — the pattern of gating sub-tasks inside the once-per-sim-tick
  `OnTelemetryData` callback by counting ticks (e.g. `tickCount & 1` for ~30 Hz pretty-graph
  sampling, a 3-tick accumulator for ~20 Hz HPR updates) rather than using wall-clock timers.
- **Session info vs. telemetry** — iRacing's SDK exposes a slow-changing YAML "session info" block
  (car/track/driver metadata, re-parsed on `OnSessionInfo`) separately from the fast per-tick numeric
  "telemetry" stream (`OnTelemetryData`); car/track/condition identity is derived from the former.

### Architectural decisions and rationale

- **`App` as composition root, one `App.<Subsystem>.cs` file per concern.** There is no DI
  container; `App : Application` (`App.xaml.cs`) is a single partial class spread across ~18 files
  (e.g. `App.ForceFeedback.cs`, `App.Voice.cs`, `App.SimagicHPR.cs`), each exposing its state as
  public fields directly on `App`. Every subsystem and the UI reach `(App)Application.Current` as an
  ambient service locator rather than depending on injected interfaces. See
  [explanation-architecture.md](explanation-architecture.md) and the file/subsystem table in
  [reference-architecture.md](reference-architecture.md). Rationale: optimizes for one developer
  holding the whole system in their head, not long-term extensibility — adding a peripheral means
  adding one new `App.<Name>.cs` file and two calls into `Initialize`/`Stop`, no registration
  ceremony.
- **Telemetry-driven, direct-DirectInput FFB**, not a vJoy/HID pass-through. `App.ForceFeedback.cs`
  reads iRacing's raw physics telemetry (steering torque at up to 360 Hz, G-force, yaw rate, shock
  velocities) and computes its own torque scalar, sent via a single DirectInput `ConstantForce`
  effect refreshed by a Windows multimedia timer, Hermite-interpolating between 360 Hz samples. The
  earlier, abandoned design — visible as dead code in `FFBReceiver.cs` (namespace `MarvinsIRFFB`,
  excluded from the build via `<Compile Remove="FFBReceiver.cs" />` in `MarvinsAIRA.csproj:16`) —
  would have registered a `vJoy` virtual joystick and sniffed/decoded iRacing's own HID FFB report
  stream. That approach was abandoned in favor of full creative control over the feel, at the cost of
  MarvinsAIRA owning all effect timing/interpolation itself and requiring users to neutralize
  iRacing's own FFB. See [explanation-force-feedback-design.md](explanation-force-feedback-design.md).
- **Event-driven telemetry with tick-count throttling, not wall-clock timers.**
  `App.IRacingSDK.cs` never polls; it calls `_irsdk.Start()` once and reacts to
  IRSDKSharper's `OnConnected`/`OnDisconnected`/`OnSessionInfo`/`OnTelemetryData` events.
  `OnTelemetryData` (typically 60 Hz) is the de-facto telemetry game loop. Sub-tasks throttle by
  counting sim ticks (not wall-clock time), and FFB/physics delta-time is `(tick delta) / TickRate` —
  frame-accurate to the sim rather than to real elapsed time. Car/track/condition change detection is
  a low-tech string-comparison of session-info display names (there's no SDK "car changed" event);
  `PauseSessionInfoUpdates` is deliberately un-paused on every session-number change to force a fresh
  parse at exactly the moments a car/track swap is likely. See
  [explanation-telemetry-sync.md](explanation-telemetry-sync.md).
- **Two independent, uncoordinated update cadences**: `MainWindow.UpdateLoop()` (a dedicated thread,
  ~10 Hz, drives settings/inputs/FFB-button-polling/wind/spotter/Logitech/chat) and
  `App.OnTelemetryData` (sim tick rate, drives the actual FFB torque computation, HPR, telemetry
  export). There is no unified "game loop" class.
- **No shared device-abstraction layer.** Logitech LED support and Simagic HPR pedal haptics are
  hand-written, unrelated modules (no `IWheelDevice`/`IHapticDevice` interface), consistent with the
  one-file-per-integration pattern.

### Standing conventions and constraints

- No dependency-injection container anywhere in the app.
- No automated test suite exists; nothing in this codebase can be unit tested without a running WPF
  `Application` and, for most subsystems, a connected iRacing sim or physical hardware
  (see [explanation-architecture.md](explanation-architecture.md)).
- No Windows service and no elevation. `App.Service.cs` (despite the name) only derives a per-machine
  GUID for the update-check API; the app runs as a normal non-elevated process, and Inno Setup
  installs with `PrivilegesRequired=lowest`.
- `Settings.xml` (plain `XmlSerializer` XML, `%MyDocuments%\MarvinsAIRA\Settings.xml`) has no
  schema/version field and no real migration discipline — migrations, where present, are ad hoc code
  blocks gated on old-value patterns and documented only by inline comments.
- **Silent-failure-by-default philosophy** is pervasive and intentional, not accidental, in several
  places: a corrupt/missing `Settings.xml` is discarded and replaced with defaults with no
  user-visible warning; log writes dropped under lock contention are silently lost; Logitech LED
  support permanently disables itself for the session on the first SDK exception, with no retry.
  `App.Initialize`, by contrast, wraps subsystem init in try/catch that logs then **rethrows** — a
  single subsystem failing to initialize can abort the whole app rather than degrading gracefully.
  This inconsistency (some subsystems self-disable, startup as a whole does not degrade) is a known,
  documented characteristic of the codebase, not something to "fix" by unifying it without asking.
- Version numbers are **auto-generated from Unix-epoch math** in `MarvinsAIRA.csproj`'s `<Version>`
  property (`1.13.$(days-since-epoch-offset).$(seconds-into-day/60)`), not manually bumped. There is
  no `VERSION` file and no `CHANGELOG.md` — none is expected for this repo. Historical commits are
  almost entirely "Version x.y.z.w updates" bumps (see `git log`), confirming this project has
  historically received little beyond version-bump commits and, most recently, a batch of
  architecture documentation.
- Settings is one large flat `INotifyPropertyChanged` class (`Settings.cs`, ~4200 lines, ~25
  `#region`s) bound directly to the UI (`MainWindow.DataContext = _settings`) — there is no separate
  view-model layer. Every property setter follows the same shape: changed-check, `App.WriteLine`
  audit log, `OnPropertyChanged()` (which both fires WPF binding and queues a debounced ~1s save).

### Key external systems and dependencies

- **`..\IRSDKSharper`** (sibling project, relative path, not in this repo) — wraps iRacing's
  shared-memory telemetry SDK; provides the `IRacingSdk` class used throughout `App.IRacingSDK.cs`.
  Must be checked out alongside this repo to build.
- **`..\SimagicHPR`** (sibling project, not in this repo) — wraps Simagic's pedal-haptics hardware
  SDK; provides the `HPR` class used in `App.SimagicHPR.cs`. Must also be checked out to build.
- **`..\MarvinsAIRASimHub`** (third sibling, referenced only by `MarvinsAIRA.sln` and by
  `InstallScript.iss`, not by the `.csproj`) — a separate SimHub plugin DLL copied into
  `{userdocs}\MarvinsAIRA` by the installer; also not part of this repo.
- Native/vendor SDKs: the Logitech G SDK via P/Invoke into `LogitechSteeringWheelEnginesWrapper.dll`
  (`LogitechGSDK.cs`); `vJoyInterfaceWrap.dll` is still shipped as content but is unused by any
  compiled code (only the excluded `FFBReceiver.cs` referenced it).
- Notable NuGet packages: `SharpDX.DirectInput`/`SharpDX.DirectSound` (wheel FFB, joystick input, LFE
  audio capture), `NAudio` (sound cues, ABS pitch-shifting), `System.Speech` (TTS),
  `System.IO.Ports` (Arduino serial), `Newtonsoft.Json` + `System.Net.Http` (update-check API),
  `ModernWpfUI`/`Extended.Wpf.Toolkit` (UI controls), `Hardcodet.NotifyIcon.Wpf` (system tray),
  `BootMeUp` (auto-start registry entry).
- **Distribution**: Inno Setup (`InstallScript.iss`), per-user install by default
  (`PrivilegesRequired=lowest`), output `MarvinsAIRA-Setup-{version}.exe`.
- **Self-update channel is custom, not GitHub Releases**: `App.Service.cs` polls
  `https://herboldracing.com/wp-json/maira/v1/get-current-version`, and downloads/launches a setup
  executable directly. GitHub is referenced only as the source-repo link in the UI's "Contribute"
  tab, not as an update mechanism.
- Firmware side: `MarvinBox/MarvinBox.ino` (Arduino) speaks a small 9600-baud ASCII protocol over
  serial (`Arduino.cs`) to control wind-simulator fan PWM.

### Product/domain framing

This app is a **sim-racing peripheral-control companion app**, not a game itself: it sits alongside a
running iRacing session, reads its live telemetry, and turns that into (a) wheel force feedback via
DirectInput, (b) Simagic HPR pedal vibration, (c) Logitech RPM shift-light LEDs, (d) Arduino-driven
wind-simulator fan speed, and (e) voice/audio feedback (TTS spotter callouts, in-game chat relay,
click/ABS sound cues). Nearly every feature exists to make the sim feel more physical or to surface
information the driver would otherwise have to glance at a HUD for.

### Known landmines — do not silently "fix" without asking

This is frozen legacy code with unplanned users on 1.x; an unplanned behavior change could surprise
someone relying on the current (buggy) behavior. Flag these, don't silently correct them:

- **Settings-migration no-op bug** (`App.Settings.cs`, marked `TEMPORARY CODE - GET RID OF THIS AFTER
  ABOUT A MONTH`): the migration block mutates the *default* `Settings` instance before it gets
  overwritten by the loaded one, so it likely never actually takes effect.
- **Shock-velocity copy-paste bug** (`App.IRacingSDK.cs:400-401`): the left-front shock-velocity datum
  is looked up from `"LRshockVel_ST"` twice instead of `"LFshockVel_ST"` — left-front shock velocity
  telemetry is never actually captured.
- **Dead code: `FFBReceiver.cs`** — excluded from the build, namespace `MarvinsIRFFB`, a vJoy-based
  FFB HID sniffer superseded by the current telemetry-driven approach. Its dependency
  `vJoyInterfaceWrap.dll` is still shipped as content but unreferenced by compiled code. Leave both in
  place unless specifically asked to clean them up.

### Repo-level fact

This is the **legacy 1.x codebase**, explicitly no longer actively developed (per `README.md` and the
status banner in `docs/reference-architecture.md`). The maintained successor is a separate repo,
[MarvinsAIRARefactored (2.x)](https://github.com/mherbold/MarvinsAIRARefactored). Before starting any
new-feature work here (as opposed to a targeted fix/doc update), a delegated worker should surface the
question "should this go in the 2.x repo instead?" rather than assuming this repo is the right target.

## What goes here, what doesn't

This file holds **durable** facts: things true regardless of which task is currently in flight —
architecture, vocabulary, external dependencies, standing conventions, and known landmines. If a fact
would still be true and useful to a worker picking up an unrelated task six months from now, it
belongs here. Anything **effort-scoped or temporary** — the state of an in-progress task, a single
PR's remaining TODOs, which files a particular worker has touched so far — belongs in
`.swarm/SWARM_STATE.md` instead, not here. Edits to this file should land through a normal PR/review
like any other documentation change, not as a direct unreviewed write during an unrelated task.
