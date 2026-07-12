# How to add a new force-feedback effect

This walks through adding a new torque contribution to the FFB engine, using the existing soft-lock
effect in `App.ForceFeedback.cs` as the worked pattern.

## Prerequisites

- Familiarity with the "FFB effect catalog" section of
  [reference-force-feedback.md](reference-force-feedback.md) — know what effects already exist and
  roughly where they live in the file.
- Familiarity with the "Effect-mixing design" section of
  [explanation-force-feedback-design.md](explanation-force-feedback-design.md) — every effect
  contributes a scalar Newton-meter delta into one additively-summed total; there is no per-effect
  DirectInput object, envelope, or duration primitive to configure.

## The worked example: soft lock

Soft lock is the simplest complete effect in the file — a single settings-gated block that computes
one scalar per update and adds it into the mix. It's a synthetic end-stop torque that resists turning
past the car's maximum steering angle. The pieces:

- **Settings** — `EnableSoftLock`, `SoftLockStrength`, `SoftLockMargin` (`Settings.cs:3451-3548`).
- **Computation** — `App.ForceFeedback.cs:1216-1225`.
- **Mixing** — `App.ForceFeedback.cs:1445`.
- **UI** — the "Soft Lock" tab in `MainWindow.xaml:920-940`.

Follow these same four pieces for a new effect.

## Steps

### 1. Add settings fields for your effect

Every tunable in the FFB engine is a property on `Settings` (`Settings.cs`), following one fixed
pattern — a backing field, a changed-value check, an `App.WriteLine` audit log, then
`OnPropertyChanged()`. `OnPropertyChanged()` both raises `INotifyPropertyChanged` (for the XAML
binding) and calls `App.QueueForSerialization()` (`Settings.cs:4215-4221`), so any UI edit is
auto-saved — you don't need to persist anything yourself. Soft lock's enable flag looks like this
(`Settings.cs:3451-3470`):

```csharp
private bool _enableSoftLock = true;

public bool EnableSoftLock
{
    get => _enableSoftLock;

    set
    {
        if ( _enableSoftLock != value )
        {
            var app = (App) Application.Current;
            app.WriteLine( $"EnableSoftLock changed - before {_enableSoftLock} now {value}" );
            _enableSoftLock = value;
            OnPropertyChanged();
        }
    }
}
```

At minimum you need a boolean `EnableYourEffect` gate. If your effect has a strength/threshold knob,
follow `SoftLockStrength` (`Settings.cs:3474-3495`): clamp the incoming value with `Math.Clamp` inside
the setter, and if you want a slider readout, add the matching `!`-prefixed `*String` shadow property
(see `SoftLockStrengthString`, `Settings.cs:3499-3518`) — the `TextBox`/`Slider` bindings in XAML read
that string form.

### 2. Wire it into the Settings UI (optional but expected)

Add a `CheckBox`/`Slider` group to `MainWindow.xaml`, bound to the properties from step 1. Soft lock's
tab (`MainWindow.xaml:920-940`) is the template: a `CheckBox` bound to `EnableYourEffect`, then a
`Grid` of `Label`/`TextBox`/`Slider` triples per numeric parameter, each `TextBox` wired to
`TextBox_GotKeyboardFocus`/`PreviewTextInput`/`LostKeyboardFocus` and each `Slider` to
`Slider_PreviewKeyDown` (the shared numeric-entry handlers already in `MainWindow.xaml.cs` — you don't
need to write new ones). This step is UI plumbing only; nothing in the FFB math depends on it.

### 3. Decide where your effect's state lives

Two patterns exist in the file, pick whichever matches your effect:

- **Stateless, computed fresh each update.** Soft lock needs no memory between frames — it's a pure
  function of the current steering angle and the current settings, computed as a local `float` inside
  `UpdateForceFeedback()`. Use this if your effect has no notion of "ramping" or history.
- **Persistent instance state.** Effects that ramp, oscillate, or track history over multiple frames
  declare private fields in the `#region Properties` block near the top of the file (e.g. the
  understeer/oversteer wave-angle accumulators `_ffb_understeerEffectWaveAngle` /
  `_ffb_oversteerEffectWaveAngle`, `App.ForceFeedback.cs:111,115`, or the crash-protection timer/scale
  pair `_ffb_crashProtectionTimer`/`_ffb_crashProtectionScale`, `:117-118`). Use this if your effect
  needs to remember something across calls — a decay timer, a running average, an oscillator phase.

Add your field(s) next to the existing ones in that region if you need persistent state.

### 4. Compute your effect's torque contribution

The private, telemetry-driven `UpdateForceFeedback()` overload (`App.ForceFeedback.cs:1091`) is where
every effect's math lives. It runs once per SDK telemetry tick (see
[reference-architecture.md](reference-architecture.md#main-update-loop) — this is the `App.OnTelemetryData`
cadence, not the 10 Hz UI loop), and delta-time inside it is derived from SDK tick deltas
(`_irsdk_tickRate`), not wall-clock time — see `App.IRacingSDK.cs:490` and the "Timing model" section
of [reference-force-feedback.md](reference-force-feedback.md). If your effect needs a decay/ramp rate,
scale it by `1f / _irsdk_tickRate` the way crash protection does (`App.ForceFeedback.cs:1248`), not by
a wall-clock delta.

Gate your computation on your new `Settings.EnableYourEffect` flag, following soft lock exactly
(`App.ForceFeedback.cs:1216-1225`):

```csharp
var yourEffectTorqueNM = 0f;

if ( Settings.EnableYourEffect )
{
    // ... compute a scalar in Newton-meters from telemetry (_irsdk_*) and/or your effect state ...
    yourEffectTorqueNM = /* ... */;
}
```

Place this either:
- **before** the per-sample loop (around line 1290, alongside soft lock and the crash/curb-protection
  scale calculations) if your effect produces one value per telemetry tick, or
- **inside** the per-sample loop (lines 1337-1685) if it needs to vary sample-to-sample — this is what
  understeer/oversteer do, advancing their wave-angle accumulator once per sample
  (`App.ForceFeedback.cs:1453-1479`) so the oscillation frequency is independent of the 360 Hz sample
  rate.

If your effect needs a strength setting expressed as a fraction of the wheel's max torque (most do),
convert it once outside the loop the way understeer/oversteer do (`App.ForceFeedback.cs:1301-1302`):
`var yourEffectStrengthNM = Settings.WheelMaxForce * Settings.YourEffectStrength / 100f;`

### 5. Mix it into the output

Inside the per-sample loop, `ffbOutputNM` is the single running total that every effect adds to or
subtracts from before the final clamp/curve step. Soft lock's mix-in is a one-line addition
(`App.ForceFeedback.cs:1445`):

```csharp
ffbOutputNM += softLockTorqueNM;
```

Add your contribution the same way, in the same loop, in a spot that makes sense next to the effects
it interacts with (soft lock is added right after the LFE mix-in and before understeer/oversteer, for
instance). Don't create a second DirectInput effect object or a separate `SetParameters` call — see
[explanation-force-feedback-design.md](explanation-force-feedback-design.md#effect-mixing-design) for
why the whole engine deliberately funnels through one scalar and one `ConstantForce` send.

### 6. Let the existing clamp/curve step handle output shaping

You don't need to clamp your own contribution to the final wheel range — `ffbOutputNM` as a whole
passes through the shared min/max-force curve (`App.ForceFeedback.cs:1512-1524`) and is converted to
DirectInput units afterward (`:1535`). Adding an extreme value from your effect just uses up more of
that shared headroom; see Troubleshooting below for what happens when it uses up all of it.

## Verification

Confirm your effect is actually contributing force:

1. **Toggle it and watch the pretty graph.** `TogglePrettyGraph()` (`App.ForceFeedback.cs:959-964`,
   surfaced in the Force Feedback tab) turns on `_ffb_drawPrettyGraph`, which renders the per-sample
   input torque (blue) and shaped output torque (white/red when clipped) into the oscilloscope-style
   `WriteableBitmap` (`:1560-1674`). Flip `Settings.EnableYourEffect` on/off while driving in a
   straight line (so the base steering torque is small and stable) and confirm the output trace shifts
   by roughly the magnitude you expect.
2. **Check `FFB_LastMagnitudeSentToWheel`.** This public property (`App.ForceFeedback.cs:127`,
   backed by `_ffb_lastMagnitudeSentToWheel`) is the actual DirectInput magnitude (-10000..10000) sent
   on the most recent multimedia-timer tick (`:1791`). It should move when your effect's trigger
   condition is true, even with the car stationary if your effect doesn't depend on speed.
3. **Add a temporary `WriteLine`.** Every subsystem logs through `App.WriteLine` to the in-UI/file log
   (see `App.Console.cs` in [reference-architecture.md](reference-architecture.md)); a one-line log of
   your computed `yourEffectTorqueNM` before you delete it is the fastest way to confirm the value is
   non-zero and in a sane range before chasing it on the wheel.
4. **Feel it on the wheel**, with `Settings.ForceFeedbackEnabled` on and iRacing's own FFB neutralized
   (see the explanation doc's note on why iRacing's own FFB must not fight this signal). Isolate your
   effect by temporarily setting `OverallScale`/`DetailScale` low and your effect's own strength high,
   so you're not trying to feel a small contribution buried under normal steering torque.

## Troubleshooting

These are failure modes actually present in the code, not hypothetical ones:

- **Effect has no visible impact even though the settings flag is on.** Check whether your
  computation is being overwritten later in the same loop iteration — mix-in order matters because
  `ffbOutputNM` is a plain running total; an effect added early and then multiplied by something later
  (e.g. `speedScale` at `:1413`, applied before the LFE/soft-lock/steering-effect additions) behaves
  differently than one added after. Soft lock, LFE, and understeer/oversteer are all added *after*
  `speedScale` is applied, so they are not speed-scaled themselves — if you want your effect scaled by
  speed too, you must apply that scale explicitly to your own contribution.
- **Output silently disappears near max steering angle or under heavy combined effects.** The shared
  curve step (`:1512-1524`) clamps to `Settings.MaxForce`, and the final DirectInput send hard-clamps
  to ±`DI_FFNOMINALMAX` (±10000) in `FFBMultimediaTimerEventCallback`
  (`App.ForceFeedback.cs:1755-1766`). Once the sum of all effects saturates that ceiling, adding more
  from your effect has zero further effect on the wheel — it's already maxed. The 3-second clip
  indicator (`_ffb_clippedTimer`, driven from the same clamp) is the visible signal this is happening;
  if you don't see it light up but also don't feel your effect, the problem is upstream of the clamp,
  not the clamp itself.
- **Effect feels like a stair-stepped buzz instead of a smooth force.** This happens if you write
  directly into `_ffb_outputDI` instead of going through the per-sample loop and its Hermite
  interpolation (`InterpolateHermite`, `:1833`, consumed by the multimedia timer callback at `:1743`).
  Always contribute your torque inside the sample loop so it rides the same interpolation path as
  every other effect — see the "Timing model" section of
  [reference-force-feedback.md](reference-force-feedback.md).
- **Effect never triggers, or triggers and never turns off.** Effects gated by a duration/timer
  (crash protection, curb protection, cooldown) count down using `_irsdk_tickRate`-scaled deltas and
  reset only when their trigger condition re-fires or the timer reaches exactly `0f`
  (`App.ForceFeedback.cs:1244-1258`). If your effect uses a similar timer pattern and the trigger
  condition is noisy (flickering true/false near a threshold), the timer will keep re-arming and the
  effect will look "stuck on" — add hysteresis to the trigger condition rather than to the timer.
- **Effect works while stationary in the garage but not on track**, or vice versa. Several existing
  gates zero out the base telemetry input off-track or in replay (`App.ForceFeedback.cs:1204-1212`),
  and `speedScale` (`:1283-1290`) reduces everything below ~5 m/s. If your effect is meant to work
  regardless of on-track state (like auto-center, `:852-957`, which explicitly only runs off-track/in
  replay) or regardless of speed, make sure it isn't accidentally placed after one of those gates or
  folded into `speedScale`.

## Related documentation

- [reference-force-feedback.md](reference-force-feedback.md) — the full FFB effect catalog and timing
  model this how-to builds on
- [explanation-force-feedback-design.md](explanation-force-feedback-design.md) — why effects are
  mixed as one additive scalar instead of separate DirectInput effect objects
- [reference-architecture.md](reference-architecture.md) — overall app structure, the two update
  cadences, and the SDK-tick-based delta-time convention used throughout FFB math
