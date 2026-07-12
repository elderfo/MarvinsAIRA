# How to Configure Force Feedback

Tune how strongly and in what ways MarvinsAIRA's force feedback engine drives your wheel, using the controls on the **Force Feedback**, **Steering Effects**, **LFE ⮕ FFB**, and **Pedal Haptics** tabs plus the related **Settings** sub-tabs.

## Prerequisites

- MarvinsAIRA installed and running, with iRacing's own force feedback turned off (MarvinsAIRA drives the wheel directly and shows a red warning banner on the Force Feedback tab if iRacing's FFB is still on) — see [tutorial-first-time-setup.md](tutorial-first-time-setup.md).
- Connected to a live iRacing session (status bar reads "Connected" with a car/track shown) — most of the settings below are easiest to judge by feel while actually on track.
- The bottom status bar's **Advanced** toggle switched on. Several controls covered here (Parked Scale, Output Minimum/Maximum, Output Curve, and the entire Steering Effects, LFE ⮕ FFB, and Pedal Haptics tabs) are hidden until Advanced is enabled (`MainWindow.xaml.cs:1902-1924`, `Advanced_ToggleSwitch_Toggled`).

## Steps

1. **Confirm force feedback is enabled and pick your wheelbase.** On the **Force Feedback** tab, make sure the checkbox in the tab header (bound to `ForceFeedbackEnabled`) is checked, and select your wheel from the **Wheelbase Device** dropdown. This enumerates connected DirectInput devices — use the **TEST** button next to it to send a test signal and confirm the right device is selected.

2. **Set Wheelbase Max.** Still on the Force Feedback tab, set the **Wheelbase Max** slider (1–50, in Newton-meters) to your wheelbase's rated maximum torque. Its tooltip: "This is the maximum force your wheelbase can produce in Newton-meters." This is `Settings.WheelMaxForce` and calibrates how the app scales output for your specific hardware.

3. **Set Overall Scale — the main strength control.** The **Overall Scale** slider (1–100%) converts the computed Newton-meter torque into DirectInput force units; this is the primary "how strong does everything feel" knob (`Settings.OverallScale`, reference-force-feedback.md's "Overall/Detail scale blend"). Raise it until the wheel feels strong enough without clipping (see Verification below), or use the **AUTO** button next to it — it auto-calibrates Overall Scale from the peak on-track torque it observes (`App.DoAutoOverallScaleNow`) and only becomes enabled once enough data has been sampled. Right-click **AUTO** to clear a previous auto-scale result.

4. **Set Detail Scale.** The **Detail Scale** slider (0–500%) controls how much fine detail — tire chatter, curb bumps, and similar texture — comes through, where 100% is "as-is from iRacing" (`Settings.DetailScale`). Raise it for more road feel, lower it to smooth the signal out.

5. **(Advanced) Set Parked Scale, Output Minimum/Maximum, and Output Curve.** With Advanced enabled, three more Force Feedback tab controls appear:
   - **Parked Scale** (0–100%) reduces force while stationary or moving very slowly (`Settings.ParkedScale`; reference-force-feedback.md's "Parked/speed scale" — the effect applies below roughly 5 m/s).
   - **Output Minimum** (0–2) boosts forces below this value up to the minimum, useful on weaker wheels that can't reproduce very small forces (`Settings.MinForce`).
   - **Output Maximum** (3–50) clamps forces above this value, useful on stronger wheels (`Settings.MaxForce`).
   - **Output Curve** (0.5–2.0) applies a power curve to compress or expand the final signal (`Settings.FFBCurve`; reference-force-feedback.md's "Min/max force curve shaping").

6. **(Advanced) Turn steering effects on/off and tune Understeer/Oversteer.** On the **Steering Effects** tab, the checkbox in the tab header (`SteeringEffectsEnabled`) is the master on/off switch. Before enabling it, the tab itself warns: *"You must calibrate the YR Factor L and YR Factor R sliders for your car, before you turn on the steering effects feature."* Within the **Understeer** and **Oversteer** group boxes you can independently set, for each:
   - **Fx Style** — a set of radio buttons choosing the waveform: Sine Wave Buzz, Sawtooth Wave Buzz, Reduce Force, Increase Force, or Pedal Haptics (matches the 4-style effect catalog in reference-force-feedback.md, plus the pedal-haptics routing option).
   - **Fx Strength** (0–100%) — how strongly the effect feels.
   - **Fx Curve** (0.25–4.0) — the curve applied to that effect.
   - **YR Factor L / YR Factor R** (Understeer) and **Y Velocity** (Oversteer) range sliders — the calibration ranges that drive when each effect kicks in, plus **Softness** (Oversteer only) for fading the effect out while counter-steering.

7. **(Advanced) Enable LFE ⮕ FFB if you want audio-derived low-frequency effects mixed in.** This requires a virtual audio cable device (e.g. VB-CABLE) already installed — the tab states this explicitly. Check the header checkbox (`LFEToFFBEnabled`), pick the virtual cable device from the **Device** dropdown, and set **Scale** (0–200%) for how strongly the audio-derived signal is added to the force feedback output (`Settings.LFEScale`).

8. **(Advanced) Enable Pedal Haptics if you have Simagic HPR hardware.** On the **Pedal Haptics** tab, check the header checkbox (`PedalHapticsEnabled`), then for each pedal (Clutch, Brake, Throttle) assign up to 3 effect slots from the effect dropdown (gear change, ABS, RPM band, understeer/oversteer, wheel lock/spin, clutch slip) and set each slot's strength slider (0–1).

9. **Configure Soft Lock and Crash/Curb Protection.** Under **Settings → Soft Lock**: check **Enable Soft Lock**, then set **Strength** (0–100%, end-stop resistance near max steering angle) and **Margin** (1–90°, how many degrees before the limit it starts applying). Under **Settings → Shock Protection**: **Crash Protection** (check **Enable Crash Protection**, then **G-Force** threshold, **Duration** in seconds, and **OS Reduction** — how much Overall Scale drops while active) reduces force after a G-force spike; **Curb Protection** (check **Enable Curb Protection**, then **Shock velocity** threshold, **Duration**, and **DS Reduction** — how much Detail Scale drops) reduces detail after a curb strike.

10. **Wind Simulator — not currently exposed in the UI.** `Settings.cs` has a full `#region Wind simulator tab` with `WindSimulatorEnabled` and a per-speed fan-power table (`WindForce1`–`WindForce8`), and the backend subsystem (`App.WindSimulator.cs` + `Arduino.cs`) is fully wired into the update loop. However, no `WindSimulator_TabItem` exists in `MainWindow.xaml` — the only trace left in the UI code is a commented-out line, `//WindSimulator_TabItem.Visibility = visibility;` (`MainWindow.xaml.cs:1913`). In this build there is no in-app control to turn on or tune the wind simulator; it isn't reachable through the UI described in this guide.

## Verification

- **Feel it in the wheel** while connected to an iRacing session and on track — Overall Scale and Detail Scale changes should be immediately noticeable (settings are read live every UI-loop tick, ~10 Hz).
- **Watch the pretty-graph oscilloscope.** Click **Enable Pretty Graph** at the bottom of the Force Feedback tab to show a live input-vs-output waveform. A **CLIPPING** banner appears (for 3 seconds) if the DirectInput magnitude hits its ±10000 hard clamp — a sign Overall Scale (or Output Maximum) is set too high.
- **Check the console log** (Help tab). Every settings change logs a line such as `OverallScale changed to 45` (the pattern every `Settings.cs` property setter follows), so you can confirm a slider edit actually took effect.
- Changes save automatically: any edit is written to `Settings.xml` after about a 1-second debounce (`App.QueueForSerialization`, `App.Settings.cs:71-171`) — there is no explicit Save button.

## Troubleshooting

- **iRacing's own FFB warning banner won't go away.** Force feedback will feel doubled or fight itself if iRacing's built-in FFB is still enabled — turn it off in iRacing's own options (see the first-time-setup tutorial).
- **Some controls are missing.** Parked Scale, Output Minimum/Maximum, Output Curve, and the entire Steering Effects / LFE ⮕ FFB / Pedal Haptics tabs are hidden unless the **Advanced** toggle in the bottom status bar is on.
- **A setting looks different than you left it on another car/wheel/track.** Under **Settings → Save File**, independent toggles (`SaveSettingsPerWheel`, `SaveSettingsPerCar`, `SaveSettingsPerTrack`, `SaveSettingsPerTrackConfig`) control whether Overall/Detail Scale (and steering-effect calibration) are shared globally or saved per profile. If per-car/per-track saving is on, `App.UpdateSettings` upserts a separate `ForceFeedbackSettings`/`SteeringEffectsSettings` entry for each combination, so the slider position you see can legitimately change when you switch cars or tracks.
- **Switching Wheelbase Device briefly cuts and restarts force feedback.** Selecting a different device in the Wheelbase Device dropdown, or clicking outside a slider after an edit, can trigger `InitializeForceFeedback` again (`FFBDevice_ComboBox_SelectionChanged`, `MainWindow.xaml.cs:1032-1040`) — this is a quick device reinitialization, not a full app restart, but expect a brief interruption in force output.
- **Steering effects feel wrong or don't engage.** The tab explicitly warns that YR Factor L/R (Understeer) and the Y Velocity range (Oversteer) must be calibrated per car before enabling the effect — see the website documentation referenced in-app for calibration steps.
- **Force feedback stops when you tab out of iRacing.** If **Automatically Disable Force Feedback When iRacing Simulator Is Not Running** (Settings → Force Feedback tab, `PauseWhenSimulatorIsNotRunning`) is checked, FFB pauses when the sim isn't running and re-enables when it is — this is expected, not a bug.
- **A settings change seems to have "reset."** A missing or corrupted `Settings.xml` is discarded silently and the app falls back to defaults with no warning (see reference-settings.md's "Persistence details") — check `%MyDocuments%\MarvinsAIRA\Settings.xml` if this happens repeatedly.
- **Wind Simulator can't be found anywhere in the app.** As covered in Step 10, this is expected in the current build — the settings fields and backend exist, but the UI tab for them does not.

## Related documentation

- [reference-force-feedback.md](reference-force-feedback.md) — the FFB effect catalog, API, and timing model these settings drive
- [reference-settings.md](reference-settings.md) — the underlying `Settings.cs` fields and persistence behavior
- [explanation-force-feedback-design.md](explanation-force-feedback-design.md) — why FFB is computed from telemetry as a single summed signal instead of per-effect DirectInput objects
- [tutorial-first-time-setup.md](tutorial-first-time-setup.md) — installing MarvinsAIRA and confirming your first telemetry connection
