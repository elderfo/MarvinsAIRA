# Voice, Spotter, Chat & Sound Cues Reference

## What it is

Five files implement MarvinsAIRA's audio-feedback layer: text-to-speech announcements
(`App.Voice.cs`), spotter proximity/flag callouts (`App.Spotter.cs`), a spoken-message-to-in-game-chat
relay (`App.ChatQueue.cs`), a `System.Speech` extension that unlocks higher-quality Windows voices
(`SpeechApiReflectionHelper.cs`), and NAudio-driven sound cues (`App.Sounds.cs`). All of it is driven
by the iRacing telemetry update loop.

## API / Interface

- `App.InitializeVoice()` (`App.Voice.cs:11`) — (re)creates `_voice_speechSynthesizer`, injects
  Windows OneCore voices via reflection, builds `Settings.VoiceList`, and selects
  `Settings.SelectedVoice` (falling back to the first installed voice if none is configured).
- `App.Say(string message, string? value = null, bool interrupt = false, bool alsoAddToChatQueue = true)`
  (`App.Voice.cs:86`) — substitutes a `:value:` token into `message`, optionally cancels current
  speech (`SpeakAsyncCancelAll`) when `interrupt` is set, speaks asynchronously, and forwards to
  `Chat()` unless suppressed. `value == string.Empty` short-circuits to a no-op (`:92-95`), letting
  callers pass templated announcements that silently skip when data is unavailable.
- `App.UpdateVolume()` (`App.Voice.cs:123`) — syncs synthesizer volume from
  `Settings.SpeechSynthesizerVolume`.
- `App.UpdateSpotter(float deltaTime)` (`App.Spotter.cs:16`) → `ProcessCarLeftRight` (`:32`) +
  `ProcessSessionFlags` (`:94`).
- `App.Chat(string message)` (`App.ChatQueue.cs:18`, private) and
  `App.ProcessChatMessageQueue()` (`:28`, public — polled every frame).
- `SpeechApiReflectionHelper.InjectOneCoreVoices(this SpeechSynthesizer)` (`:25`, extension method).
- `App.PlayClick()` / `PlayABS()` / `StopABS()` (`App.Sounds.cs:83,101,120`);
  `InitializeSounds()` (`:22`, private, called alongside `InitializeVoice()` from `App.xaml.cs:64-65`).

## Dependencies

`System.Speech.Synthesis` (voice), `NAudio.Wave` / `NAudio.Wave.SampleProviders`
(`WaveOutEvent`, `WaveFileReader`, `VolumeSampleProvider`, `SmbPitchShiftingSampleProvider`, a custom
`LoopStream`), embedded WPF pack resources `click.wav`/`abs.wav`
(`App.Sounds.cs:30,55`), telemetry fields `_irsdk_carLeftRight`, `_irsdk_sessionFlags`,
`_irsdk_playerCarNumber`, `_irsdk_windowHandle`, `_irsdk_brakeABSactive`
(see [reference-telemetry.md](reference-telemetry.md)), and many `Settings.*` properties:
`EnableSpeechSynthesizer`, `SpeechSynthesizerVolume`, `SelectedVoice`, `SpotterEnabled`,
`SpotterCalloutFrequency`, the `Say*Flag`/`SayCarLeft`/etc. phrase-template strings, `EnableChat`,
`EnableClickSound`, `ClickSoundVolume`, `EnableABSSound`, `ABSSoundVolume`, `ABSSoundPitch`.

## Dependents

The per-frame telemetry loop in `MainWindow.xaml.cs` calls `app.UpdateSpotter(deltaTime)` then
`app.ProcessChatMessageQueue()` (`:566,568`). The force-feedback telemetry path calls `PlayABS()` /
`StopABS()` when `_irsdk_brakeABSactive` transitions, and `PlayClick()` when the user presses an FFB
scale-adjust button. Settings-window sliders and ABS-test buttons in `MainWindow.xaml.cs`
(`:1722,1734,1746,1758`) call the same sound methods for live UI preview.

## Edge cases

- `ProcessCarLeftRight` (`App.Spotter.cs:45-67`) debounces chatter: transitions between adjacent
  states (`CarLeft↔TwoCarsLeft`, `CarRight↔TwoCarsRight`) are suppressed so the app doesn't
  re-announce the same side; otherwise a repeat-callout timer (`Settings.SpotterCalloutFrequency`)
  throttles re-announcing a *persistent* left/right car.
- `ProcessSessionFlags` only fires on a 0→1 bit transition per flag (`:98`), so it never repeats
  while a flag stays set.
- `UpdateSpotter` resets state (clears last car-left-right, timer) whenever spotter is disabled or the
  car is off track, so callouts don't fire stale on re-entry.
- `Chat()` requires `Settings.EnableChat`, an active connection, and a known player car number
  (`:20`) — it silently drops the message otherwise.
- The chat queue sends one full message per detected chat-window-open cycle (open → type entire
  string → close), single-threaded and serial, with no dedupe or priority — a burst of `Say()` calls
  (e.g. rapid flag changes) queues messages in order and can visibly lag behind real-time.
- No installed-voice existence check beyond the fallback-to-first-voice logic
  (`App.Voice.cs:65-68`); `InjectOneCoreVoices` failures are caught and swallowed (`:36-39`).

## Design notes

Reflection is required in `SpeechApiReflectionHelper` because
`System.Speech.Synthesis.SpeechSynthesizer` only enumerates classic desktop SAPI voices — Windows
OneCore/Mobile voices (higher quality, used by Narrator) live in a separate registry hive
(`HKEY_LOCAL_MACHINE\...\Speech_OneCore\Voices`) and aren't exposed by any public API. The helper
reaches into private `VoiceSynthesizer`/`_installedVoices`/`ObjectTokenCategory` internals to
manually register them (`SpeechApiReflectionHelper.cs:10-13,25-70`).

Spotter proximity logic treats `Off` as `Clear` (`App.Spotter.cs:38-41`) and uses a state-transition
table rather than raw equality, specifically to avoid flip-flop chatter between adjacent granularity
levels (e.g. one car vs. two cars alongside).

The ABS tone is implemented as a loop-and-pitch-shift (not a one-shot sample) so it can continuously
reflect brake modulation intensity while ABS is engaged; `PitchFactor` and volume are updated live
from `Settings.ABSSoundPitch`/`Volume` on every `PlayABS()` call rather than only at load time.

## Related

- [reference-telemetry.md](reference-telemetry.md) — telemetry inputs (car-left-right, session flags, ABS state)
- [reference-settings.md](reference-settings.md) — the phrase-template and audio-cue settings
- [howto-add-sound-cue.md](howto-add-sound-cue.md) — adding a new sound cue
- [howto-set-up-voice-alerts.md](howto-set-up-voice-alerts.md) — end-user guide to configuring voice, spotter, and chat relay
