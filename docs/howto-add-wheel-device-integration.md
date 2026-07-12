# How to add a new wheel/peripheral device integration

Wire a new third-party wheel or peripheral (shift lights, pedal haptics, or similar) into
MarvinsAIRA's telemetry-driven update loop, following the same pattern used for the Logitech
and Simagic HPR integrations.

## Prerequisites

- Familiarity with the `App.*.cs` partial-class pattern — one `App.<Subsystem>.cs` file per
  subsystem, all extending `partial class App : Application`, communicating through shared
  fields rather than an interface. See
  [reference-architecture.md](reference-architecture.md#the-appcs-partial-class-pattern).
- The device vendor's SDK: either a native DLL you'll P/Invoke into (Logitech's approach — see
  `LogitechGSDK.cs`) or a managed sibling `.csproj` referenced by relative path (Simagic's
  approach — see `..\SimagicHPR\SimagicHPR.csproj` in
  [reference-architecture.md](reference-architecture.md#external-dependencies-not-in-this-repo)).
  Decide which shape your device's SDK needs before you start.
- A read of [reference-device-integrations.md](reference-device-integrations.md) — the reference
  doc for this subsystem. This how-to won't repeat what's documented there; it tells you where to
  hook in.
- Basic familiarity with the telemetry fields you'll need (e.g. `_irsdk_rpm`, `_irsdk_gear`) — see
  [reference-telemetry.md](reference-telemetry.md).

## Steps

1. **Create `App.<YourDevice>.cs`** in the repo root, following the existing partial-class
   pattern: `namespace MarvinsAIRA { public partial class App : Application { ... } }`. Compare
   the two existing shapes before picking one:
   - Logitech's is a single P/Invoke-backed method with no init/teardown (`App.Logitech.cs:1-36`)
     because the vendor SDK is stateless per call.
   - Simagic HPR's is a fuller subsystem with a single instance field (`_hpr` at
     `App.SimagicHPR.cs:29`), `Initialize*`/`Uninitialize*`/`Stop*`/`Reset*`/`Update*` methods
     (`App.SimagicHPR.cs:55-105`), and its own constants/state fields declared at the top of the
     file (`App.SimagicHPR.cs:12-53`).
   
   If your device's SDK is a native DLL with simple stateless calls, follow Logitech. If it's a
   managed SDK with a device handle/session that needs opening and closing, follow Simagic HPR.

2. **If P/Invoking a native DLL**, declare the imports in their own class rather than inline,
   mirroring `LogitechGSDK.cs:1-13`:

   ```csharp
   [DllImport( "YourVendorSdkWrapper", CharSet = CharSet.Unicode, CallingConvention = CallingConvention.Cdecl )]
   public static extern bool YourVendorCall( /* ... */ );
   ```

   Note this is a `class`, not part of the `partial class App` — it's a plain static P/Invoke
   surface that `App.<YourDevice>.cs` calls into.

3. **If your SDK is a managed sibling project**, add a `<ProjectReference>` to
   `MarvinsAIRA.csproj` pointing at `..\<YourDeviceSdk>\<YourDeviceSdk>.csproj` (matching how
   `..\SimagicHPR\SimagicHPR.csproj` is referenced — see
   [reference-architecture.md](reference-architecture.md#external-dependencies-not-in-this-repo)),
   and add the corresponding `using <YourVendorNamespace>;` at the top of your `App.<YourDevice>.cs`
   (`App.SimagicHPR.cs:6` does `using Simagic;`). Document the new external dependency in
   reference-architecture.md's "External dependencies not in this repo" table in the same commit.

4. **Add init/teardown calls in `App.xaml.cs`'s `Initialize`/`Stop`.** `Initialize` runs
   subsystems in a fixed order (`App.xaml.cs:61-72`); add your `InitializeYourDevice(...)` call in
   the position that matches your device's dependencies (HPR's init sits right after
   `InitializeLFE()` and before `InitializeIRacingSDK()`, `App.xaml.cs:68-70`). `Stop` calls
   teardown methods in roughly reverse order (`App.xaml.cs:89-95`); add `StopYourDevice()`
   alongside `StopHPR()` (`App.xaml.cs:92`). Both `Initialize` and `Stop` are wrapped in a single
   outer `try/catch` each (`App.xaml.cs:54-80`, `:87-101`) — an exception during your init will
   abort the rest of startup and rethrow, so don't let a missing/uninstalled SDK throw out of your
   `Initialize*` method (see Troubleshooting below for how Logitech avoids this at update-time
   instead).

5. **Add a per-tick update call.** Where you hook this in depends on your device's required
   update rate — there are two independent cadences (see
   [reference-architecture.md](reference-architecture.md#main-update-loop)):
   - **~10 Hz UI-thread loop**: `MainWindow.UpdateLoop()` (`MainWindow.xaml.cs:537`) calls
     subsystem updates in a fixed sequence each tick; `app.UpdateLogitech()` is called at
     `MainWindow.xaml.cs:567`, right after `UpdateSpotter` and before `ProcessChatMessageQueue`.
     Add `app.UpdateYourDevice(...)` alongside it if your device only needs to react to settings
     changes or low-frequency state (that's all Logitech's shift lights need).
   - **Sim tick rate (~60 Hz), telemetry-driven**: `App.OnTelemetryData` in `App.IRacingSDK.cs`
     calls `UpdateHPR()` (throttled to every 3rd tick, ≈20 Hz, `App.IRacingSDK.cs:571`) and
     `ResetHPR()` on sim disconnect (`App.IRacingSDK.cs:301`). Follow this pattern if your device
     needs telemetry-rate updates (e.g. haptics that must track RPM/speed in near-real-time) —
     throttle if you don't need full sim tick rate.

6. **Wire up `Settings.cs` persistence if the device needs config.** Add a backing field plus a
   public property inside a new `#region Settings tab - <YourDevice> tab` block, matching the
   existing per-feature regions (e.g. `#region Settings tab - Logitech tab` at `Settings.cs:3745`,
   `#region Settings tab - Devices` at `Settings.cs:2811`). Follow the existing property shape
   exactly — it's not just a plain auto-property:

   ```csharp
   private bool _yourDeviceEnabled = false;

   public bool YourDeviceEnabled
   {
       get => _yourDeviceEnabled;
       set
       {
           if ( _yourDeviceEnabled != value )
           {
               var app = (App) Application.Current;
               app.WriteLine( $"YourDeviceEnabled changed - before {_yourDeviceEnabled} now {value}" );
               _yourDeviceEnabled = value;
               OnPropertyChanged();
           }
       }
   }
   ```

   This is the same shape as `ControlRPMLights` (`Settings.cs:3752-3769`, gates
   `UpdateLogitech()` at `App.Logitech.cs:12`) and `PedalHapticsEnabled`
   (`Settings.cs:1513-1530`, gates `InitializeHPR()` at `App.SimagicHPR.cs:70`). Gate your
   `Update*`/`Initialize*` method on the new setting the same way. If the feature should
   reinitialize when settings change (like HPR does — see `MainWindow.xaml.cs:1513`), call your
   `InitializeYourDevice()` from the relevant settings-checkbox handler in `MainWindow.xaml.cs`.

7. **Add UI controls if the device needs user-facing settings** — a new tab/section in
   `MainWindow.xaml` bound to the `Settings.cs` properties from step 6. This how-to doesn't cover
   WPF UI authoring in detail; see [reference-ui.md](reference-ui.md) for how existing
   settings tabs are structured.

## Verification

- Confirm your `Initialize*` method's log line appears in the console/log output at startup, in
  order with the others. `WriteLine` timestamps every entry (`App.Console.cs:43-49`) and writes
  to both the file log and `Debug.WriteLine`, so you can watch it live in the Visual Studio Output
  window during a debug run. `InitializeHPR` logs `"InitializeHPR called."` (`App.SimagicHPR.cs:66`)
  as the model to copy.
- Confirm your per-tick update method is actually being invoked: temporarily add a `WriteLine`
  inside it (or breakpoint it) and check it fires at the cadence you expect — once per
  `MainWindow.UpdateLoop` tick (~10 Hz) if wired per step 5's first option, or once per throttled
  telemetry tick (~20 Hz for HPR's 3-tick throttle) if wired per the second option.
  `_irsdk_connected` must be `true` (i.e. iRacing running and telemetry flowing) for the
  telemetry-driven path to fire at all.
- Exercise the actual hardware/vendor SDK call path: toggle the device's enable setting off/on at
  runtime and confirm the physical device responds (LEDs light, motors vibrate, etc.) and that
  toggling off cleanly stops output (compare to HPR's off-track behavior, which forces all
  channels off — `App.SimagicHPR.cs:117-124`).
- If your device has an init/teardown lifecycle, confirm `Stop()` on app exit actually releases
  the device (no dangling handle, no "device busy" on next launch) — `StopHPR()` calls
  `_hpr.Uninitialize()` unconditionally (`App.SimagicHPR.cs:90-93`).

## Troubleshooting

- **A vendor SDK call throws at runtime.** Check how you're handling it against the two real
  precedents in this codebase, which behave differently:
  - Logitech's `UpdateLogitech()` wraps the SDK call in `try/catch ( Exception )` and, on either
    an exception *or* a `false` return from `LogiPlayLedsDInput`, sets `_logitech_disabled = true`
    (`App.Logitech.cs:16-26`). That flag is checked at the top of every subsequent call
    (`App.Logitech.cs:14`), so **the feature permanently disables itself for the rest of the
    session on the very first failure** — there is no retry, and no way to re-enable it short of
    restarting the app. One log line is written the moment this happens (`App.Logitech.cs:30`,
    `"The Logitech G SDK doesn't seem to be working, so we are disabling ... support."`). If your
    device can recover from a transient failure (e.g. device unplugged and replugged), a permanent
    latch like this is probably the wrong model — consider a re-check window instead, but be
    deliberate about diverging from the existing pattern.
  - Simagic HPR has **no try/catch at all** around `_hpr.Initialize()` or `.VibratePedal(...)`
    calls (see `App.SimagicHPR.cs:55-105`, `:105-417`) — the code assumes the `SimagicHPR` sibling
    project tolerates a missing or disconnected device internally. If you follow this model, verify
    your SDK actually degrades gracefully on its own; if it doesn't, you'll get an unhandled
    exception that propagates up into `Initialize`'s or the update loop's `try/catch` (or crashes
    the app, if it's on a thread without one).
- **Your `Initialize*` throws during app startup.** `App.Initialize` has one outer `try/catch`
  that logs the exception message and **rethrows** (`App.xaml.cs:74-80`), aborting the rest of
  startup — later subsystems (`IRacingSDK`, `WindSimulator`, `Telemetry`) never initialize. Don't
  let "SDK/DLL not installed" surface as an exception from your `Initialize*` method; check for the
  condition and log + return instead, the way Logitech defers its failure handling to first
  use rather than to init (it has no `InitializeLogitech()` at all).
- **Device works but doesn't update.** Confirm the gating conditions on your `Update*` method are
  actually true at runtime — Logitech's are `Settings.ControlRPMLights`, `_logitech_disabled ==
  false`, `_irsdk_connected`, and `_ffb_drivingJoystick != null` all at once
  (`App.Logitech.cs:12-14`). A `null` `_ffb_drivingJoystick` (no FFB device selected/initialized)
  will silently no-op the whole method with no log line — there's no "why didn't this run" signal
  beyond reading the code.
- **Settings don't persist or don't reach the running subsystem.** Confirm you followed the
  property-setter shape in step 6 (not a plain auto-property) — `OnPropertyChanged()` is what
  triggers the debounced settings serialization; see
  [reference-settings.md](reference-settings.md) for how that pipeline works.
- **Corrupt or missing `Settings.xml`.** Unrelated to your new code, but worth knowing before you
  chase a phantom bug: a corrupt settings file is discarded silently and defaults are used instead
  — see [reference-architecture.md](reference-architecture.md#known-quirks-and-dead-code).

## Related documentation

- [reference-device-integrations.md](reference-device-integrations.md) — the reference doc for the
  Logitech and Simagic HPR integrations this how-to is modeled on
- [reference-architecture.md](reference-architecture.md) — the `App.*.cs` partial-class pattern,
  startup sequence, and main update loop this how-to hooks into
- [explanation-architecture.md](explanation-architecture.md) — why the codebase is shaped this way
