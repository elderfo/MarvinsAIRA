<!-- Generated: 2026-07-12 | Files scanned: 42 -->
# Frontend (WPF UI)

Single-window, tabbed WPF UI (minimizes to system tray). One main window plus
three small modal dialogs. No component framework/router — plain WPF
`Window`/`TabItem` tree with code-behind.

## Window tree

```
MainWindow (MainWindow.xaml / .xaml.cs)         → main tabbed window, Instance singleton
├── MapButtonWindow (MapButtonWindow.xaml.cs)    → modal, button-mapping capture
├── RunInstallerWindow                            → modal Yes/No, launch downloaded installer
└── NewVersionAvailableWindow                     → modal Yes/No, self-update prompt
```

## Tab tree (`MainWindow.xaml`)

```
Force Feedback   :32   → wheelbase device, scales, curve, record/play, oscilloscope
Steering Effects :184  → understeer/oversteer style, strength, curve, calibration ranges
LFE ⮕ FFB        :329  → virtual-audio-cable device + scale slider
Pedal Haptics    :367  → Simagic HPR per-pedal effect slots + pretty graphs
Spotter          :478  → callout frequency
Skid Pad         :500  → live telemetry dashboard (wheel/speed/pedals/gear/G-force/graphs)
Settings         :723  → nested TabControl:
  ├── App                    :728
  ├── Save File              :754   (per-wheel/car/track/config/condition toggles)
  ├── Devices                :765
  ├── Auto Centering         :772
  ├── Shock Protection       :846   (Crash + Curb)
  ├── Soft Lock              :920
  ├── Force Feedback         :941
  ├── Steering Effects       :970
  ├── Logitech                :975
  ├── Audio                  :982
  ├── Voice                  :1012
  ├── Chat                   :1027
  └── Messages               :1035  (editable phrase templates, 5 groups)
Help             :1463 → docs/forum links, console log view, "send console log"
Contribute       :1475 → GitHub link, Buy Me a Coffee, Etsy, supporters list
```

Bottom status bar: connection/car/track/condition + "Advanced" toggle (shows/hides
advanced tabs; default view is simplified). System tray icon at `:1514`.

## State management

No client-side state store. `MainWindow.DataContext` is a `Settings` instance —
seeded empty in XAML, overwritten with the loaded instance once
`App.Settings.cs`'s `InitializeSettings()` runs (`App.Settings.cs:65`). Every tab
binds directly to `Settings` properties. Tabs also call back into
`(App)Application.Current` for subsystem actions:

```
InitializeForceFeedback / StopForceFeedback / InitializeLFE / InitializeHPR /
InitializeVoice / UpdateVolume / DoAutoOverallScaleNow / ResetAutoOverallScaleMetrics /
PlayClick / PlayABS / StopABS / ScheduleReinitializeForceFeedback / QueueForSerialization
```

## UI-driven update loop

`MainWindow.UpdateLoop()` (`:537`) — see
[architecture.md](architecture.md#runtime-data-flow-two-independent-update-cadences).

## Key files

```
MainWindow.xaml.cs        (~2000+ lines: tab logic, WndProc, UpdateLoop, dialogs)
MapButtonWindow.xaml.cs   (button-capture: 100ms poll timer, hold+click combo support)
ClipboardHelper.cs        (CF_HTML clipboard builder — not currently used)
```

## Distribution

Inno Setup (`InstallScript.iss`) → per-user install to `{autopf}\MarvinsAIRA` by
default (no admin required), copies `MarvinsAIRASimHub.dll` separately into
`{userdocs}\MarvinsAIRA`, output `MarvinsAIRA-Setup-{version}.exe`.

## Related maps

- [architecture.md](architecture.md) — subsystems these tabs configure
- [dependencies.md](dependencies.md) — UI toolkit packages (ModernWpfUI, Extended.Wpf.Toolkit, Hardcodet.NotifyIcon.Wpf)
