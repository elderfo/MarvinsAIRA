# UI, Dialogs & Distribution Reference

## What it is

`MainWindow.xaml`/`.xaml.cs` is the single-window, tabbed WPF UI (minimizes to system tray) used to
configure every runtime subsystem and to drive the app's per-frame update loop and image rendering.
Three small modal dialogs support it: `MapButtonWindow` (button-mapping capture),
`RunInstallerWindow` and `NewVersionAvailableWindow` (self-update flow).

## Tabs / sections (`MainWindow.xaml`)

| Tab | Line | Contents |
|---|---|---|
| Force Feedback | `:32` | Wheelbase device combo, max force, overall/detail/parked scale, output min/max, FFB curve, record/play/reset, pretty-graph oscilloscope, iRacing-FFB-not-enabled warning |
| Steering Effects | `:184` | Understeer and Oversteer group boxes: effect style, strength, curve, yaw-rate-factor/Y-velocity ranges, softness |
| LFE ⮕ FFB | `:329` | Audio device combo + scale slider (requires a virtual audio cable device, e.g. VB-CABLE) |
| Pedal Haptics | `:367` | Simagic HPR clutch/brake/throttle, 3 effect slots each with combo + strength, per-pedal pretty graphs |
| Spotter | `:478` | Callout frequency slider; warns if voice synthesis is off |
| Skid Pad | `:500` | Live telemetry dashboard: wheel image/rotation, speed, pedal bars, gear, ABS, X/Y velocity, yaw rate, yaw-rate factor, speed/RPM ratio, peak G-force/shock velocity, oversteer/understeer graphs |
| Settings (nested tabs) | `:723` | App, Save File (per-wheel/car/track/config/condition toggles), Devices, Auto Centering, Shock Protection (Crash + Curb), Soft Lock, Force Feedback, Steering Effects, Logitech, Audio, Voice, Chat, Messages (editable phrase templates, grouped General / iRacing / Sliders / Spotter Car-Left-Right / Spotter Session-Flags) |
| Help | `:1463` | Documentation/forum links, console log view, "send console log" (copies to clipboard, opens forum PM) |
| Contribute | `:1475` | GitHub repo link, Buy Me a Coffee, Etsy store, hardcoded supporters list (`MainWindow.xaml.cs:274`) |

Bottom status bar: connection/car/track/condition status plus an "Advanced" toggle that shows/hides
advanced tabs and controls (default is a simplified view).

System tray icon (`tb:TaskbarIcon`, `:1514`): context menu Show/Exit; left-click restores the window.

## API / Interface

- `MainWindow` (`MainWindow.xaml.cs:25`) — `partial class MainWindow : Window`; static `Instance`
  property (`:84`) used as a global singleton accessor by `App` and the dialogs.
- `GetVersion()` (`:97`), `CloseAndLaunchInstaller(string)` (`:104`) — sets a "die now" flag and
  closes so `Window_Closed` can `Process.Start` the downloaded installer.
- `ReinitializeForceFeedback()` (`:1006`), `FixRangeSliders()` (`:1406`), `UpdatePrettyGraph()`
  (`:2024`), `UpdatePedalHapticsPrettyGraphs()` (`:2050`) — pushed into by `App` subsystems.
- `WndProc` (`:362`) handles a custom `WM_MAIRA` message (for SimHub/plugin integration — an enum
  covers Overall/Detail/Parked scale, Min/Max force, FFB curve, LFE, LFE-enabled), plus
  `WM_DEVICECHANGE` and `WM_SYSCOMMAND` (minimize → transparency).
- `UpdateLoop()` (`:537`) — see [reference-architecture.md](reference-architecture.md#main-update-loop).

### Dialogs

- `MapButtonWindow` (`MapButtonWindow.xaml.cs:6`) — modal, constructed with a
  `Settings.MappedButtons` entry; polls `app.UpdateInputs` on a 100 ms timer and
  `app.Input_AnyPressedButton` to capture button presses. Supports a single button or a two-button
  "hold + click" combo (a third press shifts Button1 → Button2). `canceled` public field signals
  whether the caller should persist the mapping.
- `RunInstallerWindow` / `NewVersionAvailableWindow` — trivial modal Yes/No dialogs with
  `installUpdate`/`downloadUpdate` public bool fields, shown from `App.Service.cs`.
- `ClipboardHelper.CreateDataObject`/`CopyToClipboard`/`GetHtmlDataString` — builds CF_HTML clipboard
  format; not currently used by the "send console log" flow, which uses plain `Clipboard.SetText`
  instead (likely intended for a future rich-paste feature).

## Dependencies

Every tab binds through the window's `DataContext`, which is seeded to a fresh `<local:Settings />`
in XAML (`MainWindow.xaml:26-28`) and then overwritten with the real loaded instance once settings
are read from disk (`mainWindow.DataContext = _settings;`, `App.Settings.cs:65`, inside
`InitializeSettings()`). Tabs also call back into `(App)Application.Current` for
`InitializeForceFeedback`/`StopForceFeedback`/`InitializeLFE`/
`InitializeHPR`/`InitializeVoice`/`UpdateVolume`/`DoAutoOverallScaleNow`/`ResetAutoOverallScaleMetrics`/
`PlayClick`/`PlayABS`/`StopABS`/`ScheduleReinitializeForceFeedback`/`QueueForSerialization` — i.e. the
`App.*.cs` subsystems documented in the other reference docs.

## Edge cases

- `Advanced_ToggleSwitch_Toggled` (`:1902`) shows/hides an entire set of tabs and controls; default
  is the simplified (non-advanced) view.
- Record/Play buttons no-op with a log message if the iRacing sim isn't connected (`:1057,1095`);
  recording auto-enables the pretty graph.
- The auto-scale button is disabled until `FFB_AutoOverallScaleIsReady`.
- Numeric text boxes use regex-filtered input (digits + a single decimal point) and sync to a paired
  slider on focus loss (`:940-995`).
- `PauseWhenSimulatorIsNotRunning` auto-disables FFB when iRacing exits, gated by a cooldown check
  (`FFB_IsCoolingDown`).
- First run vs. subsequent run: window position is restored only if previously saved (not `-1,-1`);
  `StartMinimized` + `CloseToSystemTray` on first activation closes straight to the tray.

## Design notes

System tray behavior uses `Hardcodet.Wpf.TaskbarIcon`; closing to tray cancels the close event, hides
the window, sets `ShowInTaskbar = false`, and shows a one-time balloon tip (unless
`HideTrayAlert`). Auto-start on login uses `WK.Libraries.BootMeUpNS` (registry-based, current user,
no elevation).

**Self-update is a custom flow, not GitHub-based**: `App.Service.cs` polls
`https://herboldracing.com/wp-json/maira/v1/get-current-version`, compares version strings, shows
`NewVersionAvailableWindow` if newer, downloads the setup executable into the user's Documents
folder, then `RunInstallerWindow` asks to launch it. `MainWindow.CloseAndLaunchInstaller` sets a
die-now flag so `Window_Closed` (`:138`) can `Process.Start` the installer after full app teardown.
GitHub is referenced only as the source-code repo link
(`GoToGitHub_Button_Click` → `github.com/mherbold/MarvinsAIRA`), not as an update channel.

Button-mapping capture (`MapButtonWindow`) polls raw input every 100 ms rather than being
event-driven, and supports two-button "hold + click" combos.

## Distribution / installation (`InstallScript.iss`)

Inno Setup script. Installs to `{autopf}\MarvinsAIRA` (Program Files), per-user via two directives —
`PrivilegesRequired=lowest` (defaults to a per-user install) and
`PrivilegesRequiredOverridesAllowed=dialog` (lets the user opt into an admin/all-users install from
the setup dialog) — **no admin rights required by default**. Copies published binaries from
`bin\publish\*`, plus a separate `MarvinsAIRASimHub.dll` (a SimHub plugin, not part of this repo —
see [reference-architecture.md](reference-architecture.md#external-dependencies-not-in-this-repo))
into `{userdocs}\MarvinsAIRA`. Creates a Start Menu entry and optional desktop shortcut, offers to
launch the app post-install. Output file `MarvinsAIRA-Setup-{version}.exe`.

## Related

- [reference-architecture.md](reference-architecture.md) — app lifecycle this UI drives
- [reference-force-feedback.md](reference-force-feedback.md), [reference-device-integrations.md](reference-device-integrations.md), [reference-voice-audio.md](reference-voice-audio.md), [reference-settings.md](reference-settings.md) — subsystems each tab configures
