# How to add a new sound cue

Add a new NAudio-driven sound-effect cue (a played `.wav` file, following the pattern used by the
existing click and ABS cues) to MarvinsAIRA, from asset wiring through the code that triggers and
plays it.

## Prerequisites

- A read of [reference-voice-audio.md](reference-voice-audio.md) — the reference doc for this
  subsystem. This how-to won't repeat what's documented there; it walks through adding one concrete
  new cue.
- Familiarity with the `App.*.cs` partial-class pattern — one `App.<Subsystem>.cs` file per
  subsystem, all extending `partial class App : Application`, communicating through shared fields.
  See [reference-architecture.md](reference-architecture.md#the-appcs-partial-class-pattern).
- No NAudio experience required — the exact API chain to copy is laid out below.

> **Adding a spoken/TTS cue instead?** A callout like the spotter's car-left/right or flag
> announcements, or a chat message, doesn't touch `App.Sounds.cs` at all — it calls
> `App.Say(string message, ...)` (`App.Voice.cs:86-121`), which speaks the text through
> `SpeechSynthesizer` and (unless you pass `alsoAddToChatQueue: false`) forwards it into the
> in-game chat relay in `App.ChatQueue.cs` automatically (`App.Voice.cs:117-120`). See
> `App.Spotter.cs:94-132` for the edge-triggered pattern used for flag callouts, and
> `App.Spotter.cs:32-92` for the debounced pattern used for car-left/right proximity. This how-to
> focuses on the played-file path, which is the more representative "add one of these" case.

## Steps

1. **Add the `.wav` asset to the repo root** and wire it into `MarvinsAIRA.csproj` as an embedded
   WPF resource, matching the existing `abs.wav`/`click.wav` entries. Both a `None Remove` and a
   `Resource Include` entry are required — the project's default glob picks new files up as
   `<None>`, so they must be explicitly removed from that bucket and re-added as `<Resource>`:

   ```xml
   <!-- in the <None Remove="..." /> ItemGroup -->
   <None Remove="yourcue.wav" />

   <!-- in the <Resource Include="..." /> ItemGroup -->
   <Resource Include="yourcue.wav" />
   ```

   Compare to the existing entries at `MarvinsAIRA.csproj:21,24` (`None Remove` for `ABS.wav` /
   `click.wav`) and `:58,61` (`Resource Include` for `abs.wav` / `click.wav`).

2. **Load the sound in `InitializeSounds()`** (`App.Sounds.cs:22-81`), following the click block as
   your template if the cue is a one-shot effect (fields at `App.Sounds.cs:11-13`, load logic at
   `:28-51`):
   - A `readonly WaveOutEvent` field, a `WaveStream?` field, and a `VolumeSampleProvider?` field.
   - `var resourceStream = GetResourceStream( new Uri( "pack://application:,,,/yourcue.wav" ) );`
     (`App.Sounds.cs:30` is the click equivalent) — the pack URI filename must exactly match the
     `Resource Include` from step 1.
   - Wrap it: `new WaveFileReader( resourceStream.Stream )` → `.ToSampleProvider()` →
     `new VolumeSampleProvider( ... )` (`App.Sounds.cs:32,34`).
   - Call `_sounds_yourCueWaveOutEvent.Init( _sounds_yourCueVolumeSampleProvider )` inside a
     `try/catch` that logs on failure (`App.Sounds.cs:38-51`) and set `.Volume = 1` as the initial
     value.

   If the cue needs to loop or vary pitch while active (like the ABS tone, which reflects brake
   modulation continuously rather than playing once), follow the ABS block instead: wrap the
   `WaveFileReader` in a `LoopStream` and optionally a `SmbPitchShiftingSampleProvider`
   (`App.Sounds.cs:15-18,53-78`).

   Add your load block *before* `_sounds_initialized = true;` at `App.Sounds.cs:80` — that flag
   gates every `Play*()` call, so a cue added after that line would never be playable.

3. **Add a `PlayYourCue()` method**, mirroring `PlayClick()` (`App.Sounds.cs:83-99`) for a one-shot
   cue or `PlayABS()`/`StopABS()` (`:101-129`) for a looping one:

   ```csharp
   public void PlayYourCue()
   {
       if ( _sounds_initialized )
       {
           if ( Settings.EnableYourCueSound )
           {
               if ( ( _sounds_yourCueWaveStream != null ) && ( _sounds_yourCueVolumeSampleProvider != null ) )
               {
                   _sounds_yourCueWaveStream.Seek( 0, System.IO.SeekOrigin.Begin );

                   _sounds_yourCueVolumeSampleProvider.Volume = Settings.YourCueSoundVolume / 100f;

                   _sounds_yourCueWaveOutEvent.Play();
               }
           }
       }
   }
   ```

   The `Seek(0, SeekOrigin.Begin)` before `Play()` is what lets the same `WaveOutEvent`/stream pair
   be replayed from the top on every call — there's no re-`Init()` per play.

4. **Add the corresponding `Settings.cs` properties** — `EnableYourCueSound` (`bool`, default
   `false`) and `YourCueSoundVolume` (`int`, e.g. default `75`) — inside the existing
   `#region Settings tab - Audio tab` block (`Settings.cs:3773-3890`). Copy the exact
   backing-field-plus-property shape used by `EnableClickSound`/`ClickSoundVolume`
   (`Settings.cs:3777-3819`); it's not a plain auto-property — it logs the change and calls
   `OnPropertyChanged()`:

   ```csharp
   private bool _enableYourCueSound = false;

   public bool EnableYourCueSound
   {
       get => _enableYourCueSound;
       set
       {
           if ( _enableYourCueSound != value )
           {
               var app = (App) Application.Current;
               app.WriteLine( $"EnableYourCueSound changed - before {_enableYourCueSound} now {value}" );
               _enableYourCueSound = value;
               OnPropertyChanged();
           }
       }
   }
   ```

5. **Wire the trigger** — decide the condition that should fire `PlayYourCue()` and hook it into
   the right place. Two real patterns exist in this codebase:
   - **Telemetry state-transition (edge-triggered).** The ABS tone is triggered from
     `App.IRacingSDK.cs:522-532`, inside `OnTelemetryData`, by comparing
     `_irsdk_brakeABSactive != _irsdk_brakeABSactiveLastFrame` — it fires once on the transition,
     not on every tick the state holds. Use this shape if your cue should announce a *change* in
     telemetry state.
   - **UI/input event.** The click sound is triggered from `App.ForceFeedback.cs:640-659`, inside
     the force-feedback update, when `Settings.IncreaseLFEScaleButtons.ClickCount` shows a button
     press happened that tick.

   Whichever you pick, `InitializeSounds()` itself is called from `App.xaml.cs:65`, right after
   `InitializeVoice()` (`:64`), as part of the fixed startup order documented in
   [reference-architecture.md](reference-architecture.md#startup-sequence).

6. **(Optional) Add a live-preview hook and UI controls.** `MainWindow.xaml.cs:1716-1724` calls
   `app.PlayClick()` directly from the click-volume slider's `ValueChanged` handler purely so
   changing the slider previews the sound; `:1726-1760` do the same for `PlayABS()`/`StopABS()`
   from a press-and-hold test button. Add an analogous slider/checkbox on the Settings window's
   Audio tab bound to your new `Settings.cs` properties if you want the same UX — WPF UI authoring
   itself isn't covered here; see [reference-ui.md](reference-ui.md).

## Verification

- Build and launch the app. `InitializeSounds()` runs inside `App.Initialize()`'s single outer
  `try/catch`, which logs and **rethrows** on any unhandled exception
  (per [reference-architecture.md](reference-architecture.md#startup-sequence),
  `App.xaml.cs:74-80`) — so if the app starts at all, your resource wiring from step 1 didn't
  throw.
- Watch for your `WriteLine` log lines (`"...loading yourcue sound..."` /
  `"...yourcue sound loaded..."`, if you copied the click block's logging style) either in the
  Visual Studio Output window, the in-app Console tab, or `%MyDocuments%\MarvinsAIRA\Console.log`
  (`App.Console.cs:18,43-86`) — they should appear in order right after the click/ABS lines.
- Trigger the actual firing condition (in-sim telemetry, or a button press) and confirm you hear
  it. For a quick bypass of the real trigger while testing, model the ABS-test button's approach —
  call `PlayYourCue()` directly from a temporary button handler the way
  `ABSTest_Button_PreviewMouseLeftButtonDown` calls `app.PlayABS()` (`MainWindow.xaml.cs:1726-1736`).
- Confirm the settings gate actually gates: flip `Settings.EnableYourCueSound` off and re-trigger
  the condition — the cue should go silent even though the trigger still fires, since the check is
  inside `PlayYourCue()` (`App.Sounds.cs:87,105` are the click/ABS equivalents), not at the call
  site.

## Troubleshooting

- **Nothing plays, no error anywhere.** Two independent silent-no-op gates exist in every
  `Play*()` method: `_sounds_initialized` and `Settings.EnableYourCueSound` (see
  `App.Sounds.cs:85-87,103-105`). Both fail silently by design — there's no log line when a
  disabled setting or an uninitialized subsystem swallows a play call. Check both before assuming
  the trigger code itself is wrong. `EnableYourCueSound` defaults to `false` if you copied the
  existing property pattern, so a freshly wired cue is disabled until something (a UI checkbox or
  a manually edited `Settings.xml`) turns it on.
- **A bad or missing resource wiring can take down the whole app, not just your cue.** The
  wav-loading calls — `GetResourceStream`, `new WaveFileReader(...)`, wrapping in sample providers
  (`App.Sounds.cs:30-34,55-61`) — are **not** wrapped in try/catch; only the subsequent
  `WaveOutEvent.Init()` calls are (`:38-51,65-78`). A mismatched pack URI or a missing
  `Resource Include` in the `.csproj` throws unhandled out of `InitializeSounds()`, which propagates
  to `App.Initialize()`'s outer try/catch and aborts the entire startup sequence — including
  subsystems that come after Sounds (Inputs, ForceFeedback, IRacingSDK, etc.), and including the
  already-working click/ABS cues, since they share the same `InitializeSounds()` call.
- **Watch for stale log strings if you copy-paste a block as a template.** The existing ABS
  `Init()` catch handler at `App.Sounds.cs:77` logs `"Failed to create click wave out event: ..."`
  — a leftover copy-paste string that says "click" even though it's in the ABS code path. If you
  copy either block verbatim, double-check every log string (including inside catch blocks) says
  your new cue's name, so failures are traceable to the right cue later.
- **The cue "stutters" or restarts continuously.** `PlayClick()`/`PlayABS()` (and your new method)
  have no internal debounce — `Play()` after `Seek(0)` simply restarts playback every call. The ABS
  trigger avoids this by comparing this-frame vs. last-frame state
  (`_irsdk_brakeABSactive != _irsdk_brakeABSactiveLastFrame`, `App.IRacingSDK.cs:522`) so it only
  fires on a transition. If your trigger condition is a sustained telemetry value checked every
  tick without an edge/transition comparison, the cue will restart on every tick it's true (up to
  ~60 Hz for a telemetry-driven trigger, ~10 Hz for a `MainWindow.UpdateLoop`-driven one) instead
  of playing once. For a spoken cue with real throttling/debounce to model instead, see the
  state-transition table and callout-frequency timer in `App.Spotter.cs:32-92`.

## Related documentation

- [reference-voice-audio.md](reference-voice-audio.md) — the reference doc for `App.Sounds.cs`,
  `App.Voice.cs`, `App.Spotter.cs`, and `App.ChatQueue.cs`
- [reference-architecture.md](reference-architecture.md) — the `App.*.cs` partial-class pattern,
  startup sequence, and main update loop this how-to hooks into
- [reference-settings.md](reference-settings.md) — how `Settings.cs` properties persist once
  `OnPropertyChanged()` fires
