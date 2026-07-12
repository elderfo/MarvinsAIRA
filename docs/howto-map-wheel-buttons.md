# How to Map Wheel Buttons to App Actions

This shows you how to assign a physical button on your wheel, wheelbase, or a separate button box to
one of MarvinsAIRA's in-app actions — like nudging the force feedback scale up or down without
touching the mouse — using the app's Button Mapping dialog.

## Prerequisites

- MarvinsAIRA installed and running — see
  [tutorial-first-time-setup.md](tutorial-first-time-setup.md) if you haven't done this yet.
- A wheel, wheelbase, or button box connected and recognized by Windows as a DirectInput device.
  Button mapping isn't limited to your force-feedback wheelbase — MarvinsAIRA enumerates *every*
  attached DirectInput/joystick device (`App.Inputs.cs`, `InitializeInputs()`), so a separate USB
  button box works just as well as buttons on the wheel itself.

## What you can map

Every mappable action has its own small icon button next to the control it affects — there's no
single "add a mapping" screen. Clicking one of these icons is how you both choose the action *and*
open the mapping dialog for it:

| Icon | Where to find it | Action | What it does when triggered |
|---|---|---|---|
| **R** | Force Feedback tab, next to the Wheelbase Device dropdown | Reinitialize force feedback | Re-initializes the force-feedback device connection |
| **A** | Force Feedback tab, Overall Scale row | Trigger auto overall scale | Runs the auto-overall-scale calibration once (only has an effect once auto-scale is "ready," same as clicking the on-screen AUTO button) |
| **C** | Force Feedback tab, Overall Scale row | Clear auto overall scale | Resets the auto-overall-scale calibration data collected so far |
| **−** / **+** | Force Feedback tab, Overall Scale row | Decrease / increase overall scale | Steps Overall Scale down/up, plays a click sound, and (after about a second) speaks the new value if voice announcements are configured |
| **−** / **+** | Force Feedback tab, Detail Scale row | Decrease / increase detail scale | Same pattern as overall scale, for Detail Scale |
| **U** | Steering Effects tab → Understeer group box | Calibrate understeer range from feel | Reads the car's yaw-rate factor at the instant you press it and sets the understeer effect's start/end range around that value — it updates the **left** range if you're steering left at that moment, or the **right** range if you're steering right, so press it while actually cornering |
| **−** / **+** | LFE ⮕ FFB tab | Decrease / increase LFE scale | Same pattern as overall/detail scale, for the LFE-to-FFB mix level |

Two notes on finding these icons:

- The **Steering Effects** and **LFE ⮕ FFB** tabs (and therefore the **U** button and the LFE
  **−**/**+** buttons) are only visible when the **Advanced** toggle in the bottom status bar is on.
  The Force Feedback tab's R/A/C/−/+ buttons are visible either way.
- The Force Feedback tab also has a **record** button and a **play** button next to the Wheelbase
  Device dropdown (round record icon, triangular play icon) — those belong to the FFB
  record/playback feature, not button mapping. Only the small **R** icon after them opens a mapping
  dialog.

## Steps

### 1. Decide which action to map, and click its icon

Find the row for the action you want (see the table above) and click its icon button. This is a
plain button in the main window — no menu or right-click involved.

### 2. The Button Mapping dialog opens and starts listening immediately

Clicking the icon opens a small modal "Button Mapping" window (`MapButtonWindow`) on top of the main
window. There's no separate "start listening" step: the moment the dialog activates, it starts a
100&nbsp;ms polling timer that reads every attached joystick/button box for a new button press
(`Window_Activated` → `OnTimer`, which calls the same input-polling routine the app uses everywhere
else, `App.UpdateInputs`). The dialog shows two instructional labels:

> Press any button on any controller to select it.
>
> If you want to map a button combo, press and release the hold button first, then press and
> release the click button second.

If the action already has a mapping, the dialog opens showing it (e.g. "Button 3 on G923 Racing
Wheel") instead of a blank state — it's pre-loaded with whatever's currently assigned, not empty.

### 3. Press the physical button you want to assign

Press (and release) the button on your wheel or button box. As soon as the dialog's poll detects it,
the label updates to show the device name and button number, e.g. "Button 5 on Simucube Button Box."

**Important — replacing an existing single-button mapping:** if the action already had one button
mapped and you press a *different* button without clicking **Clear** first, the new press does not
replace the old one. Because the dialog always fills the first empty slot, and the old button already
occupies the first slot, your new press goes into the second slot instead — turning what was a single
button into a hold-then-click combo (see step 4) with the old button as the "hold." If you just want
to swap in a different single button, click **Clear** (step 5) before pressing the new one.

### 4. (Optional) Map a two-button hold + click combo

To require a second button be held down before the mapped press registers: press and release the
"hold" button first, then press and release the "click" button. The dialog fills Button 1 (hold) and
Button 2 (click) in that order, and the label changes to show both, e.g. "Button 3 on G923 Racing
Wheel (HOLD) + Button 7 on G923 Racing Wheel (CLICK)". At runtime, the click button only counts while
the hold button is currently pressed down (`App.Inputs.cs`, `UpdateInputs()`).

A few mechanical details worth knowing:

- Pressing the same button that's already assigned (as either Button 1 or Button 2) again does
  nothing — it's ignored rather than duplicated.
- Once both slots are filled, pressing a third, different button shifts the combo forward: the
  current "click" button becomes the new "hold" button, and your new press becomes the "click"
  button. This lets you re-pick the click button without starting over, but it does discard whatever
  the original hold button was.

### 5. Save, clear, or cancel

- **Update** — saves the mapping shown in the dialog and closes it.
- **Clear** — empties both button slots in the dialog (back to "Not set.") without closing it, so you
  can immediately press a fresh button. Click **Update** afterward to actually remove the mapping, or
  **Cancel** to back out without removing anything.
- **Cancel** — closes the dialog and discards whatever you pressed. Closing the dialog any other way
  (including its window-close control) has the same effect as Cancel — only clicking **Update**
  persists your changes (`MapButtonWindow.xaml.cs`, `Window_Closing`/`canceled`).

Clicking **Update** takes effect immediately for the running app (the very next button poll uses the
new mapping) and is written to your settings file on disk about a second later
(`App.QueueForSerialization()`, a 1-second debounced save to `Documents\MarvinsAIRA\Settings.xml` —
see [tutorial-first-time-setup.md](tutorial-first-time-setup.md) for where that file lives). Button
mappings are global app settings, not tied to the currently selected car, track, or per-wheel
profile — a mapping you set once applies everywhere.

While the Button Mapping dialog is open, the main window pauses its normal handling of these mapped
buttons, so pressing a button while you're mapping something else won't also trigger its old action.

## Verification

1. Close the Button Mapping dialog with **Update**.
2. Press the physical button you just mapped.
3. Watch for the effect that matches the action (see the table above):
   - **Overall/Detail/LFE scale −/+**: the corresponding slider on the Force Feedback or LFE ⮕ FFB
     tab should move one step, you should hear a click sound (if enabled under Settings → Audio), and
     about a second later you should hear the new value spoken (if voice announcements are
     configured).
   - **Reset (R)**, **Auto-scale (A)**, and **Clear auto-scale (C)**: these have no slider to watch,
     so the most reliable confirmation is the console log (see below).
   - **Understeer calibrate (U)**: press it while actually cornering on track; the understeer
     Y-Factor L or R range on the Steering Effects tab should update to straddle the yaw-rate factor
     you were feeling at that instant.
4. For any action, open **Help → console log view** and look for a line logged at the moment you
   pressed the button (e.g. "RESET button pressed!", "AUTO-OVERALL-SCALE button pressed!",
   "INCREASE-OVERALL-SCALE button pressed!"). Every mapped-button trigger logs a line here
   (`App.ForceFeedback.cs`, `UpdateForceFeedback()`), which is the one confirmation that works for
   every action, including the ones with no visible on-screen change.

## Troubleshooting

- **Nothing happens when I press the button, and no combo was intended.** Re-open the mapping dialog
  for that action and check what's actually assigned — if a stray extra press turned your intended
  single-button mapping into a hold+click combo (see step 3), the action now needs both buttons
  pressed in the hold-then-click sequence, not just one.
- **A mapping that used to work stops working after I unplug and replug the device (or plug it into
  a different USB port).** MarvinsAIRA re-enumerates all input devices automatically when Windows
  reports a device was added or removed, but only if **Reinitialize when devices are added or
  removed from the system** is checked under Settings → Devices (on by default). Mappings are keyed
  to the device's DirectInput instance GUID plus a button number; if Windows assigns your device a
  different instance GUID after reconnecting — which can happen if it's plugged into a different
  physical port — the stored mapping silently stops matching anything, and the button simply does
  nothing (no error is shown). Re-map it if that happens (`App.Inputs.cs` de-dupes/enumerates devices
  by instance GUID on every re-init).
- **The device dropped out mid-session.** If a joystick throws an exception while MarvinsAIRA is
  polling it (a common symptom of an unplugged device), the app silently removes it from its internal
  device list and logs an exception message — that's the app's disconnect handling; there's no
  separate "device disconnected" dialog. Any buttons mapped on that device stop responding until it's
  reconnected and the app re-initializes inputs (see the point above).
- **Two different actions are mapped to the same physical button.** There's no conflict warning or
  prevention anywhere in the mapping dialog or the button-processing logic — both actions will fire
  every time you press that button. If two actions seem to trigger together, check whether you
  mapped the same button (or the same hold button in two different combos) to both.
- **The icon for an action I want isn't visible.** The Steering Effects and LFE ⮕ FFB tabs (and the
  U/LFE mapping icons on them) are hidden unless the **Advanced** toggle in the bottom status bar is
  turned on.
- **I pressed a button but the dialog's label never updates.** The dialog only recognizes presses on
  buttons/POV controls it can see through DirectInput — confirm the device shows up at all elsewhere
  in the app (e.g. in the Wheelbase Device dropdown on the Force Feedback tab, if it's FFB-capable)
  before assuming the mapping dialog is broken; a device Windows hasn't recognized won't be polled by
  MarvinsAIRA either.

## Related documentation

- [reference-ui.md](reference-ui.md) — full tab-by-tab breakdown of the main window, including the
  `MapButtonWindow` dialog's implementation notes
- [reference-force-feedback.md](reference-force-feedback.md) — how `App.Inputs.cs` fits into the
  force-feedback/input subsystem, and what each mapped action changes under the hood
- [tutorial-first-time-setup.md](tutorial-first-time-setup.md) — install, first launch, and first
  telemetry connection, if you haven't completed initial setup yet
