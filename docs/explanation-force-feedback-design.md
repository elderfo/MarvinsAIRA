# Why Force Feedback Works This Way

## The problem

A racing wheel's force feedback needs to feel like the road: rich, high-frequency, physically
correct torque updates, with zero perceptible lag, driven by a game (iRacing) that most FFB-capable
titles expect to talk to directly over a device driver — not by a third-party companion app.
MarvinsAIRA needs to inject its own computed torque onto the wheel *in addition to* (or instead of)
whatever iRacing itself sends, without fighting the sim or the wheel driver for control of the same
hardware effect.

## The approach that was abandoned: `FFBReceiver.cs` / vJoy sniffing

The dead code in `FFBReceiver.cs` (excluded from the build, namespace `MarvinsIRFFB`) shows the
earlier design: register a virtual joystick (`vJoyInterfaceWrap`) with iRacing, let iRacing send its
own FFB effects to that virtual device, and have the app intercept/decode the low-level HID FFB
report stream (effect create/start/stop, envelope, condition, periodic, ramp, constant, device gain).
This is a legitimate FFB architecture — it's how tools that *pass through* or *modify* a game's own
FFB signal typically work — but it means the app is limited to reshaping what iRacing already
computed, and it depends on decoding an entire vendor-specific HID FFB protocol correctly.

## The approach actually used: telemetry-driven, single constant-force effect

`App.ForceFeedback.cs` takes a different path: it reads iRacing's **raw physics telemetry**
(steering-wheel torque samples at up to 360 Hz, G-force, yaw rate, shock velocities, lateral
velocity) directly from the SDK, computes its own torque value in application code, and sends that
value to the wheel through **one single DirectInput `ConstantForce` effect object**, refreshed via a
Windows multimedia timer. iRacing's own FFB output to the wheel is bypassed/ignored in favor of this
computed signal.

This trades one problem for another:

- **Gained**: full control over the feel — every effect (understeer waveform, curb protection, soft
  lock, auto-centering, etc.) is just application-level math over telemetry, easy to add or tune
  without needing to understand HID FFB report formats at all.
- **Gained**: only one hardware FFB effect object needs to exist and stay alive per session, instead
  of juggling a set of DirectInput effect objects (periodic, condition, ramp, etc.) and their
  lifetimes — effect mixing happens as scalar addition in software before a single `SetParameters`-
  style update.
- **Cost**: MarvinsAIRA now owns the entire force computation, including its timing. The 360 Hz
  telemetry samples don't arrive at the same rate the multimedia timer sends to the device, so the
  app must Hermite-interpolate between samples (`InterpolateHermite`, `App.ForceFeedback.cs:1833`) to
  avoid a stair-stepped, buzzy feel — complexity that a driver-level pass-through approach wouldn't
  need.
- **Cost**: because the app computes torque itself rather than relaying iRacing's, users must have
  iRacing's *own* FFB effectively neutralized or complementary (the UI surfaces a warning label when
  iRacing's own FFB is still enabled — see [reference-ui.md](reference-ui.md)) — a coordination
  requirement the vJoy pass-through approach wouldn't have had in the same way.

## Effect-mixing design

Rather than modeling each FFB effect (understeer, oversteer, soft lock, curb protection, crash
protection, LFE...) as a separate DirectInput effect object layered by the OS/driver, every effect in
`App.ForceFeedback.cs` contributes a scalar torque delta that gets **additively summed into one
Newton-meter value**, which then passes through a single curve-shaping/clamp step
(`FFBCurve`, min/max force) before being sent as one `ConstantForce` magnitude. This is simpler to
reason about (one signal, one clamp, one send) at the cost of not being able to use DirectInput's
own per-effect envelope/duration primitives — all envelope-like behavior (crash protection ramping,
cooldown ramping) is hand-implemented as time-based scale multipliers rather than DirectInput
`Envelope` structures.

## Trade-offs, summarized

| Choice | Gains | Costs |
|---|---|---|
| Telemetry-driven torque vs. vJoy/HID pass-through | Full creative control per effect; simple device lifetime | Must own timing/interpolation; requires user to disable/tune iRacing's own FFB |
| Single summed scalar vs. per-effect DirectInput objects | One clamp/curve step; easy to add new effects | No native envelope/duration primitives; all blending is hand-rolled |
| Multimedia timer decoupled from sim tick, with Hermite interpolation | Smooth output independent of variable sim tick timing | Extra complexity (`InterpolateHermite`) and another thread to manage |

## Related

- [reference-force-feedback.md](reference-force-feedback.md) — the concrete effect catalog and API
- [explanation-architecture.md](explanation-architecture.md) — the broader "one dev, ship fast" ethos this reflects
- [howto-add-force-feedback-effect.md](howto-add-force-feedback-effect.md) — applying this design to add a new effect
