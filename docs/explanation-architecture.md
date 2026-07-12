# Why the Codebase Is Shaped This Way

## The problem

MarvinsAIRA has to read a game's telemetry at up to 360 Hz, compute physically-felt force feedback
in real time, and simultaneously drive a tabbed configuration UI, a handful of independent hardware
peripherals (wheel, pedals, Arduino fans), and voice output — all as a small side project maintained
by (effectively) one person, not a team with a platform to invest in. Every architectural choice here
optimizes for "one developer can hold the whole system in their head and ship a feature by end of
day," not for long-term extensibility.

## The approach: `App` as composition root, one partial-class file per subsystem

Rather than a DI container, service layer, or event bus, the app has a single `App : Application`
class whose implementation is spread across ~18 `App.<Subsystem>.cs` files (`App.xaml.cs`,
`App.ForceFeedback.cs`, `App.Voice.cs`, etc. — see the table in
[reference-architecture.md](reference-architecture.md)). Every subsystem exposes its state as public
fields directly on `App`, and every other subsystem — plus `MainWindow` and its dialogs — reaches
`(App)Application.Current` to read or call into it.

```
        ┌─────────────────────────── App (composition root) ───────────────────────────┐
        │                                                                               │
        │  App.Console   App.Settings   App.Service   App.IRacingSDK   App.Telemetry     │
        │  App.CurrentCar/Track/WetDryCondition   App.ForceFeedback   App.Inputs          │
        │  App.LFE   App.WindSimulator   App.Logitech   App.SimagicHPR                    │
        │  App.Voice   App.Spotter   App.ChatQueue   App.Sounds                           │
        │                                                                               │
        └───────────────────────────────────┬───────────────────────────────────────────┘
                                             │  (App)Application.Current, shared fields
                                  ┌──────────┴──────────┐
                                  │      MainWindow      │  ← UI, main update loop, DataContext=Settings
                                  └──────────┬──────────┘
                          ┌───────────────────┼───────────────────┐
                   MapButtonWindow   RunInstallerWindow   NewVersionAvailableWindow
```

This buys real speed: adding a new peripheral integration means adding one new `App.<Name>.cs` file
and wiring two calls into `Initialize`/`Stop` — no interfaces to satisfy, no registration ceremony.
The `Settings` object is likewise one flat class bound straight to the UI via
`INotifyPropertyChanged`, so a new setting is a new property, not a new layer.

## Trade-offs

- **No testability boundary.** Every subsystem depends on ambient global state
  (`(App)Application.Current`, static fields) rather than injected interfaces, so none of this code
  can be unit tested without a running WPF `Application` and, for most subsystems, a connected
  iRacing sim or physical hardware. There are no automated tests in this repo.
- **No graceful degradation.** `App.Initialize` wraps subsystem init in try/catch that logs then
  rethrows (`App.xaml.cs:74-80`) — a single subsystem failing to initialize (e.g. a missing native
  DLL) can abort the whole app rather than disabling just that feature. In practice, some subsystems
  compensate individually (Logitech permanently disables itself after one exception, see
  [reference-device-integrations.md](reference-device-integrations.md)), but this is inconsistent
  across subsystems, not a shared policy.
- **Settings has no migration discipline.** There's no schema version field — migrations are ad hoc
  code blocks with comments as the only versioning signal (see
  [reference-settings.md](reference-settings.md)). The one migration present in the code is marked
  `TEMPORARY CODE - GET RID OF THIS AFTER ABOUT A MONTH` and, on inspection, mutates the default
  `Settings` instance before it gets overwritten by the loaded one — it likely never actually ran.
  This is a direct cost of not having a settings-versioning convention from the start.
- **Failures are silent by default.** A corrupt `Settings.xml` is discarded without any user-visible
  warning; log writes dropped under lock contention are silently lost. This matches a "don't crash
  the app over a settings file" philosophy, but it also means data loss (a corrupted/reset settings
  file) is invisible to the user who experiences it.
- **No Windows service, no elevation.** Despite the name, `App.Service.cs` is not a Windows service —
  it derives a per-machine GUID for the update-check API. The app runs entirely as a normal
  (non-elevated) user process; Inno Setup installs it with `PrivilegesRequired=lowest`. This keeps
  install/run friction low at the cost of not being able to do anything that genuinely requires
  elevation (e.g. certain low-level HID access patterns some racing-wheel drivers use).

## Alternatives considered (visible in the code)

The clearest discoverable alternative is `FFBReceiver.cs` — a `vJoy`-based virtual-joystick FFB
sniffer in a different namespace (`MarvinsIRFFB`), excluded from the build. It represents an earlier
architecture where the app would have intercepted Windows' own FFB packets aimed at a virtual
joystick, rather than reading iRacing's telemetry directly and computing force itself. That approach
was abandoned — see [explanation-force-feedback-design.md](explanation-force-feedback-design.md) —
but the dead code and its shipped-but-unused `vJoyInterfaceWrap.dll` were never cleaned up, which is
itself characteristic of a small side project's priorities: shipping the working approach mattered
more than removing the abandoned one.

## Related

- [reference-architecture.md](reference-architecture.md) — the concrete file/subsystem map
- [explanation-force-feedback-design.md](explanation-force-feedback-design.md)
- [explanation-telemetry-sync.md](explanation-telemetry-sync.md)
