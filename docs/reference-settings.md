# Settings & Persistence Reference

## What it is

`Settings` (`Settings.cs:11`) is a single flat `INotifyPropertyChanged` class holding every
user-configurable option in the app — roughly 25 `#region`s covering General, Force Feedback,
Steering Effects, LFE-to-FFB, Pedal Haptics, Wind Simulator, Spotter, per-tab UI settings, Advanced,
Lists, and version info (`Settings.cs:31-4229`). It is bound directly to the UI
(`MainWindow.DataContext = _settings`, `App.Settings.cs:65`) and read directly by every subsystem —
there is no separate view-model layer.

## API / Interface

- `App.Settings` (`App.Settings.cs:16-19`) — the single live `Settings` instance for the process.
- `App.InitializeSettings()` — loads `Settings.xml` (or creates defaults), called once from
  `App.Initialize`.
- `App.UpdateSettings(float deltaTime)` (`App.Settings.cs:71-171`) — polled every UI-loop tick
  (~10 Hz). On a 1-second debounce timer, it upserts per-wheel/car/track `ForceFeedbackSettings` and
  per-car `SteeringEffectsSettings` list entries, then calls `Serializer.Save`.
- `App.QueueForSerialization()` (`App.Settings.cs:173-179`) — marks settings dirty; every property
  setter in `Settings.cs` calls this via `OnPropertyChanged()`, so any UI edit schedules a save.
- `Serializer.Load<T>(string path)` / `Serializer.Save(string path, object obj)` (`Serializer.cs`) —
  generic `XmlSerializer`-based load/save, used only for `Settings` today.
- `SerializableDictionary<TKey, TValue>` (`SerializableDictionary.cs`) — hand-rolled `IXmlSerializable`
  dictionary, needed because `XmlSerializer` cannot serialize `Dictionary`/`SortedDictionary`
  natively. Used for per-car/per-track/per-condition settings maps.

## Options / Configuration

Every property in `Settings.cs` follows the same pattern: a backing field, a changed-value check, an
`App.WriteLine` audit-log call, then `OnPropertyChanged()`. Categories include (non-exhaustive —
see `Settings.cs` region headers for the authoritative list):

- **General**: start with Windows, start minimized, close to tray, topmost, window opacity, update
  checking.
- **Save-file scoping**: independent toggles for per-wheel, per-car, per-track, per-track-config,
  per-wet/dry-condition force-feedback and steering-effect profiles.
- **Force feedback**: overall/detail/parked scale, min/max force, FFB curve, auto-scale calibration
  tolerance, update frequency, pause-when-sim-not-running.
- **Steering effects**: understeer/oversteer effect style, strength, yaw-rate-factor and Y-velocity
  ranges, softness.
- **Shock protection**: crash-duration and curb-detection thresholds.
- **Pedal haptics** (Simagic HPR): per-pedal (clutch/brake/throttle) up to 3 effect slots with
  strength, from a fixed effect catalog (gear change, ABS, wide/narrow RPM band, understeer/oversteer,
  wheel lock/spin, clutch slip).
- **Voice / Spotter / Chat**: TTS enable/volume/voice selection, spotter callout frequency, per-phrase
  message templates (large "Messages" settings tab — one editable string per spoken/chat phrase),
  chat relay enable.
- **Audio cues**: click and ABS sound enable/volume/pitch.
- **Logitech**: RPM shift-light LED enable.
- **Wind simulator**: per-speed fan-power table.

## Examples

Load-then-fallback pattern used at startup (`App.Settings.cs`):

```csharp
var settings = Serializer.Load<Settings>(settingsPath);
if (settings != null)
{
    _settings = settings;
}
// otherwise _settings keeps the `new Settings()` default instance
```

Every settings property setter follows this shape (representative, not verbatim):

```csharp
public float OverallScale
{
    get => _overallScale;
    set
    {
        if (_overallScale == value) return;
        _overallScale = value;
        App.WriteLine($"OverallScale changed to {value}");
        OnPropertyChanged();
    }
}
```

`OnPropertyChanged()` both raises `INotifyPropertyChanged` (for WPF binding) and calls
`App.QueueForSerialization()`.

## Persistence details

- Format: plain `XmlSerializer` XML.
- Location: `%MyDocuments%\MarvinsAIRA\Settings.xml`.
- Save trigger: debounced — any property change schedules a save that fires after ~1 second of
  quiescence, rather than saving synchronously on every change.
- Load failure handling: `Serializer.Load` swallows all deserialization exceptions and returns
  `null`; a missing or corrupt `Settings.xml` silently falls back to a fresh `Settings()` with
  defaults — there is no user-visible warning that saved settings were lost.
- No schema/version field drives migrations. `Settings.Version` (`Settings.cs:4231`) reflects the
  app's build version for display only. Data migrations, where they exist, are ad hoc code blocks in
  `InitializeSettings` gated on old-value patterns, documented only by inline comments (see the
  known no-op migration bug in [reference-architecture.md](reference-architecture.md#known-quirks-and-dead-code)).

## Related

- [reference-architecture.md](reference-architecture.md) — how `App.Settings.cs` fits into startup
- [explanation-architecture.md](explanation-architecture.md) — why settings persistence has no
  versioning discipline
