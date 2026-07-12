# Force Feedback, Inputs & Wind Simulator Reference

## What it is

`App.ForceFeedback.cs` (1843 lines — the largest file in the codebase) is the FFB effects engine: it
turns iRacing telemetry into a single DirectInput constant-force signal sent to the wheel.
`App.Inputs.cs` enumerates and polls joysticks/wheel buttons. `App.LFE.cs` mixes an
audio-derived low-frequency signal into the FFB output. `App.WindSimulator.cs` + `Arduino.cs` +
`MarvinBox/MarvinBox.ino` drive Arduino-controlled cooling fans to simulate wind based on speed.

## API / Interface

- `App.InitializeForceFeedback()` / `UninitializeForceFeedback()` / `ReinitializeForceFeedbackDevice()`
  (`App.ForceFeedback.cs:142,244,291`).
- `App.UpdateForceFeedback()` — two overloads: a button/settings-poll version (`:496`) and a
  telemetry-driven private version (`:1091`) that computes and sends the actual force.
- `App.UpdateConstantForce()` (`:993`) — pushes the current magnitude to the DirectInput device via
  a Windows multimedia timer callback (`FFBMultimediaTimerEventCallback`, `:1695`).
- `FFB_*` public properties (`:125-138`) — UI-bound state (e.g. auto-scale readiness, cooldown state).
- `App.DoAutoOverallScaleNow()` / `ResetAutoOverallScaleMetrics()` — auto-calibration of overall
  scale from observed peak on-track torque.
- `App.Inputs.cs`: `InitializeInputs()` (`:29`), `UpdateInputs()` (`:118`),
  `Input_CurrentWheelPosition`/`Input_CurrentWheelVelocity`, `Input_AnyPressedButton`.
- `App.LFE.cs`: `InitializeLFE()` (`:31`), `UninitializeLFE()` (`:120`), `LFEThread()` (`:166`).
- `App.WindSimulator.cs`: `InitializeWindSimulator()` (`:23`), `UpdateWindSimulator()` (`:46`),
  `GetWindForce()` (`:107`), `UpdateFanPowers()` (`:146`).
- `Arduino` class (`Arduino.cs`): constructor performs port handshake, `SendMessage(string)`, `Close()`.

## FFB effect catalog

Each of the following is computed independently in `App.ForceFeedback.cs` and additively mixed into
one Newton-meter torque scalar before a single curve-shaping/clamp step:

| Effect | Description | Approx. location |
|---|---|---|
| Base constant force | Telemetry-derived steering torque, sent via a single DirectInput `ConstantForce` effect object | `:993` |
| Overall/Detail scale blend | Steady-state vs. impulse (delta-torque) blend; algorithm differs above/below 100% detail scale | `:1384-1406` |
| LFE mix-in | Audio-derived low-frequency magnitude added when `LFEToFFBEnabled` | `:1416-1441` |
| Soft lock | Synthetic end-stop torque near max steering angle | `:1216-1225` |
| Crash protection | Zeroes/reduces overall scale for a configurable duration after a G-force spike | `:1227-1258` |
| Curb protection | Reduces detail scale after a shock-velocity spike (curb strike) | `:1260-1279` |
| Parked/speed scale | Reduces force below ~5 m/s | `:1281-1290` |
| Understeer | 4 selectable waveforms (sine, ramp, steady-state-scaled, static), driven by a yaw-rate factor (steering angle × speed / yaw rate) | `UFF_ProcessYawRateFactor :1010`, effect `:1449-1479` |
| Oversteer | Same 4-style pattern, driven by lateral velocity, with a softness fade when counter-steering | `:1122-1152`, `:1481-1509` |
| Auto-center | Two algorithms (bang-bang velocity-target vs. quadratic position curve), used off-track/in replay | `:852-957` |
| Auto-overall-scale calibration | Tracks peak on-track torque to auto-set `OverallScale` | `DoAutoOverallScaleNow :407` |
| Min/max force curve shaping | Power-curve (`FFBCurve`) applied between `MinForce`/`MaxForce` | `:1512-1524` |
| Cooldown | Ramps torque to 0 over ~1s when leaving the track ("sore thumbs" prevention) | `:1770-1787` |
| Record/playback | Raw torque sample buffer for capturing/replaying a session's FFB signal | `:64-67`, `:1192-1200` |

Visual aids: a clip indicator and "pretty graph" oscilloscope view (input vs. output waveform)
render into a `WriteableBitmap` (`:1560-1674`, `:1749-1766`) bound to `MainWindow`'s Force Feedback
tab.

Physical wheel buttons can, at runtime, bump Overall/Detail/LFE scale, force a device reinit, trigger
or clear auto-scale, and adjust understeer thresholds (`:496-660`, mapped via
`App.Inputs.MappedButtons`).

## Timing model

360 Hz telemetry torque samples (`_irsdk_steeringWheelTorque_ST[6]`) are smoothed to the multimedia
timer's send rate via Hermite interpolation (`InterpolateHermite`, `:1833`) — the multimedia timer
runs on its own thread decoupled from the sim's telemetry tick.

## Dependencies

`SharpDX.DirectInput` (wheel FFB output, joystick/button polling), `SharpDX.DirectSound` (LFE audio
capture), `System.IO.Ports` (Arduino serial), `WinApi.TimeSetEvent`/`TimeKillEvent` (multimedia
timer), iRacing telemetry fields from [reference-telemetry.md](reference-telemetry.md)
(`_irsdk_steeringWheelTorque_ST`, `_irsdk_yawRate`, `_irsdk_velocityX/Y`, `_irsdk_gForce`, shock
velocities, `_irsdk_isOnTrack`/`_irsdk_simMode`), and dozens of `Settings.*` fields (see
[reference-settings.md](reference-settings.md)) covering scale factors, steering-effect parameters,
crash/curb thresholds, and wind force/speed tables.

## Dependents

Called from `MainWindow.UpdateLoop()` on the ~10 Hz UI-loop cadence
(see [reference-architecture.md](reference-architecture.md#main-update-loop)).
`MainWindow.WheelForceFeedback_Image` binds to the pretty-graph bitmap; `FixRangeSliders()` in
`MainWindow.xaml.cs` reacts to FFB settings changes in the UI.

## Edge cases

- DirectInput magnitude is hard-clamped to ±10000, with a 3-second clip-indicator timer.
- A cooldown state machine prevents a torque jolt when leaving the track.
- Device exceptions set `_ffb_forceFeedbackExceptionThrown` and schedule a reinit.
- `App.Inputs.cs` joystick enumeration de-dupes by `InstanceGuid` and silently drops any joystick
  whose `GetCurrentState`/`Poll` throws — this is the app's disconnect-handling mechanism (no
  explicit reconnect logic beyond re-enumeration).
- LFE uses a double-buffered magnitude array (index-flip) to avoid tearing between the audio-capture
  thread and the FFB thread.
- `Arduino.ConnectPort` probes every COM port with a handshake+timeout so it won't send wind data to
  an unrelated serial device.
- `MarvinBox.ino` has a 5-second no-data fan shutoff and a 0.1s LED-off — a firmware-side fail-safe
  if the host app dies or disconnects.

## Serial protocol (Arduino.cs ↔ MarvinBox.ino)

9600 baud, ASCII, over `System.IO.Ports`.

- **Handshake**: the app opens each candidate COM port and writes `"wind"` (4 bytes). The firmware,
  on receiving `'w'` then matching `"ind"`, echoes `"wind"` back. A byte-exact echo is the app's
  positive identification that it found the right device.
- **Fan commands**: the app writes `L###` or `R###` (a letter plus exactly 3 ASCII digits, e.g.
  `"L160"`) for the left/right fan. The firmware accumulates the 3 digits into a `powerLevel`
  (0–320) and, if in range, writes it directly into `OCR1A` (pin 9, left fan) or `OCR1B` (pin 10,
  right fan) — Timer1 PWM compare registers (16-bit phase-correct PWM, `ICR1` TOP = 320).
- Any malformed byte resets the firmware's 1-byte parser state machine. `LED_BUILTIN` blinks on every
  byte received as an activity indicator.

## Dead code: `FFBReceiver.cs`

Excluded from the build (`<Compile Remove="FFBReceiver.cs" />` in `MarvinsAIRA.csproj:16`). It's in
namespace `MarvinsIRFFB` (not `MarvinsAIRA`) and uses `vJoyInterfaceWrap` (the vJoy virtual-joystick
SDK) to register a low-level FFB PID callback and decode/log every HID FFB report type — a
debug/reverse-engineering sniffer for a virtual joystick's FFB traffic, not a force generator. It
appears to be leftover code from an earlier/sibling project superseded once the app switched to
reading iRacing telemetry directly and driving the wheel via SharpDX. `vJoyInterfaceWrap.dll` is
still shipped as content (`<None Update>` in the csproj) but unreferenced by any compiled code. See
[explanation-force-feedback-design.md](explanation-force-feedback-design.md) for why this approach
was abandoned.

## Related

- [reference-telemetry.md](reference-telemetry.md) — the telemetry inputs this subsystem consumes
- [reference-settings.md](reference-settings.md) — the settings fields that parameterize every effect
- [reference-device-integrations.md](reference-device-integrations.md) — Simagic HPR pedal haptics,
  which reuse the understeer/oversteer amounts computed here
- [explanation-force-feedback-design.md](explanation-force-feedback-design.md) — design rationale
  and trade-offs
