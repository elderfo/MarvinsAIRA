# Tutorial: First-Time Installation and Setup

In this tutorial you'll install MarvinsAIRA, launch it for the first time, and confirm it is
receiving live telemetry from iRacing — the moment the status bar changes from "iRacing Simulator
is Not Running" to "Connected" and starts showing your car and track, you'll know the app is wired
up correctly. From there you'll turn on force feedback so the wheel actually responds to what's
happening on track.

This tutorial covers installation and the first successful telemetry connection only. Tuning force
feedback strength/curves, voice alerts, and button mapping are separate how-to guides linked at the
end.

> **A note on this guide's limits:** this documentation was written by reading the installer script
> and application source code, not by running the installer or the app. MarvinsAIRA is a
> Windows-only WPF desktop app that requires the iRacing sim to be meaningful, and the two
> conditions needed to build and exercise it end-to-end (a Windows machine, and two sibling
> projects — IRSDKSharper and SimagicHPR — checked out alongside this repo) aren't available in the
> environment this doc was written in. Every step below is grounded in what `InstallScript.iss` and
> the application code actually do, not a screenshot-by-screenshot walkthrough. Where something
> depends on iRacing itself (window titles, exact in-sim menu wording) that can't be verified from
> this repo's code, it's called out explicitly.

## What you'll need

- **Windows.** MarvinsAIRA is a WPF desktop app built for `net8.0-windows`; there is no macOS/Linux
  build.
- **iRacing installed, with an active membership**, and the iRacing sim client itself set up and
  able to join a session. MarvinsAIRA has no telemetry or force feedback to show you until iRacing
  is running and you're in a car.
- **A force-feedback-capable wheel/wheelbase**, connected and working in Windows (and ideally
  already confirmed working in iRacing's own controls). MarvinsAIRA drives the wheel through
  DirectInput (`SharpDX.DirectInput`) — this documentation set doesn't establish a maintained list
  of specific compatible wheelbases beyond two integrations with dedicated code paths: **Logitech**
  wheels (for RPM shift-light LEDs) and **Simagic HPR** pedals (for pedal haptic vibration) — see
  [reference-device-integrations.md](reference-device-integrations.md). Any DirectInput
  force-feedback wheel should work for the core force-feedback loop this tutorial sets up; the
  Logitech/Simagic integrations are optional extras, not requirements.
- A few minutes and a working internet connection to download the installer.

## Step 1: Download and run the installer

MarvinsAIRA is distributed as a single Inno Setup installer, built from `InstallScript.iss` in this
repo. From that script, here's exactly what the installer does:

- App name shown in the installer UI: **"Marvin's Awesome iRacing App"**.
- Installer file name: **`MarvinsAIRA-Setup-{version}.exe`** (e.g.
  `MarvinsAIRA-Setup-1.2.3.exe`).
- **No administrator rights are required by default** — the script sets
  `PrivilegesRequired=lowest`, so it installs per-user out of the box. It also sets
  `PrivilegesRequiredOverridesAllowed=dialog`, which means the installer's first screen lets you
  opt into an all-users/admin install instead, if you want one — you don't have to.
- Install location: **`{autopf}\MarvinsAIRA`** — Inno Setup's per-user-or-machine "Program Files"
  path (for a per-user install this typically resolves to
  `C:\Users\<you>\AppData\Local\Programs\MarvinsAIRA`; for an all-users install,
  `C:\Program Files\MarvinsAIRA` or `C:\Program Files (x86)\MarvinsAIRA`). The main executable is
  installed there as **`MarvinsAIRA.exe`**.
- A second folder is created at **`{userdocs}\MarvinsAIRA`** — i.e. your
  `Documents\MarvinsAIRA` folder — and the installer copies a SimHub plugin file,
  `MarvinsAIRASimHub.dll`, into it. This is a separate optional SimHub integration, not something
  you need to interact with directly. (This is also the same folder the app itself uses for its
  settings file — more on that in Step 3.)
- Shortcuts: a Start Menu entry (**Start Menu → MarvinsAIRA → MarvinsAIRA**) is always created.
  A desktop shortcut is optional — the installer presents a "Create a desktop icon" checkbox task
  during setup.
- At the end of the wizard, the installer offers to launch the app immediately ("Start Marvin's
  Awesome iRacing App") — leave that checked if you want to jump straight into Step 2.

Run the downloaded `MarvinsAIRA-Setup-{version}.exe`, click through the wizard (default options are
fine for a first install), and let it finish.

## Step 2: Launch the app for the first time

Start MarvinsAIRA from the Start Menu entry, the desktop shortcut (if you created one), or by
letting the installer's "launch now" option do it for you.

Based on the app's startup sequence (see
[reference-architecture.md](reference-architecture.md#startup-sequence)), here's what happens and
what you should expect to see:

- The app enforces **single-instance startup** — if you somehow launch it twice, the second launch
  just brings the existing window to the front instead of opening a duplicate.
- On first activation of the main window, the app creates its data folder at
  **`%MyDocuments%\MarvinsAIRA`** (i.e. `Documents\MarvinsAIRA` — the same folder the installer
  seeded with the SimHub plugin) and initializes subsystems in this fixed order: Console → Settings
  → Service → Voice → Sounds → Inputs → Force Feedback → LFE → HPR → iRacing SDK → Wind Simulator →
  Telemetry.
- **There is no first-run wizard or setup dialog.** The main window opens directly into its normal
  tabbed interface, pre-populated with default settings. The tabs you'll see across the top are:
  **Force Feedback**, **Steering Effects**, **LFE ⮕ FFB**, **Pedal Haptics**, **Spotter**,
  **Skid Pad**, **Settings** (with its own nested sub-tabs: App, Save File, Devices, Auto Centering,
  Shock Protection, Soft Lock, Force Feedback, Steering Effects, Logitech, Audio, Voice, Chat,
  Messages), **Help**, and **Contribute**. By default the view is simplified — an "Advanced" toggle
  in the bottom status bar reveals additional tabs/controls if you need them later, but you don't
  need it for this tutorial.
- At the bottom of the window is a **status bar**. On a fresh install, before iRacing is running,
  it reads **"iRacing Simulator is Not Running"**, with the car and track fields showing **"No
  Car"** / **"No Track"**. That's expected — it will update once you're in a session (Step 4).
- **This is your first settings file being created.** `Settings.xml` is written to
  `Documents\MarvinsAIRA\Settings.xml`. If the file doesn't exist yet (as on a first run) or is
  ever corrupted, the app silently falls back to built-in defaults rather than erroring out — so if
  something looks unexpectedly "reset" later, that's a known, documented behavior, not a bug you
  need to troubleshoot (see [reference-architecture.md](reference-architecture.md#known-quirks-and-dead-code)).
- The app minimizes to the **system tray** rather than the taskbar when closed/minimized; look for
  its icon there if the window disappears.

At this point the app is running but idle — it has nothing to show yet because iRacing isn't
running.

## Step 3: Turn off iRacing's own force feedback

Before you launch iRacing, there's one setting to check inside iRacing itself: **iRacing's built-in
force feedback needs to be off**, because MarvinsAIRA drives the wheel's force feedback directly
and the two will conflict if both are active.

You don't have to hunt for this — MarvinsAIRA tells you. The **Force Feedback** tab has a built-in
check: if it detects that iRacing's own force feedback is currently enabled, it displays a
prominent red warning banner reading:

> **WARNING - Force feedback is currently enabled in iRacing!**

If you see that banner once iRacing is running, go into iRacing's own options and disable its force
feedback, leaving MarvinsAIRA as the only thing driving the wheel. The banner disappears once
iRacing's own FFB is off.

Also confirm the **Force Feedback** checkbox in MarvinsAIRA's own "Force Feedback" tab header is
checked (it's on by default), and that your wheelbase is selected in the **Wheelbase Device**
dropdown on that tab — MarvinsAIRA enumerates connected DirectInput devices there, so your wheel
should appear in the list once Windows recognizes it.

## Step 4: Launch iRacing and join a session

With MarvinsAIRA still running, start iRacing and join any session — a test/practice session is
the easiest way to confirm the connection quickly; you don't need to be racing against anyone.

Per the iRacing SDK connection lifecycle (see
[reference-telemetry.md](reference-telemetry.md#connection-lifecycle)), MarvinsAIRA polls for the
iRacing sim process in the background and connects automatically — there's no "Connect" button to
click. When it picks up the running sim, watch the status bar at the bottom of the MarvinsAIRA
window:

- **"iRacing Simulator is Not Running"** changes to **"Connected"**.
- The car field updates from "No Car" to your current car's name.
- The track field updates from "No Track" to the track name (and configuration, if applicable).

Once you're actually on track, two more things confirm live telemetry is flowing, not just a
process-level connection:

- The **Skid Pad** tab becomes a live dashboard — wheel rotation, speed, pedal inputs, gear, ABS
  state, G-forces, and more update in real time as you drive.
- Steering forces from the car should now be coming through your wheel — if you completed Step 3,
  what you feel is MarvinsAIRA's force feedback engine converting live telemetry into torque.

If the status bar stays on "iRacing Simulator is Not Running" after iRacing is fully loaded and
you're in a session, double check that iRacing launched normally (not through some restricted
mode) — this documentation set can't verify iRacing-side troubleshooting steps beyond what the
MarvinsAIRA code itself reports.

## What you built

You installed MarvinsAIRA per-user with no admin rights required, launched it, disabled iRacing's
conflicting built-in force feedback, and confirmed the two are talking: the status bar shows
"Connected" with your car and track, the Skid Pad tab shows live telemetry, and your wheel is
receiving force feedback driven by that telemetry.

From here, the defaults are deliberately conservative — the real tuning work (force strength and
curves, steering effects, voice alerts, button mappings) happens in the how-to guides:

- [How to configure force feedback](howto-configure-force-feedback.md) — tuning wheel max force,
  overall/detail scale, and the FFB curve for your specific wheelbase.
- [How to set up voice alerts](howto-set-up-voice-alerts.md) — spotter callouts, TTS, and the
  Messages tab's editable phrase templates.
- [How to map wheel buttons](howto-map-wheel-buttons.md) — assigning physical wheel buttons to
  in-app actions like scale adjustments and auto-scale triggers.

## Related documentation

- [reference-ui.md](reference-ui.md) — full tab-by-tab breakdown of the main window and its dialogs
- [reference-architecture.md](reference-architecture.md) — startup sequence, subsystem map, and
  known quirks referenced throughout this tutorial
- [How to configure force feedback](howto-configure-force-feedback.md)
- [How to set up voice alerts](howto-set-up-voice-alerts.md)
- [How to map wheel buttons](howto-map-wheel-buttons.md)
