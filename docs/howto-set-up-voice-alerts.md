# How to set up voice alerts

Configure MarvinsAIRA's spoken text-to-speech announcements, spotter callouts, and iRacing chat relay so the app talks to you (and optionally your chat window) during a session.

## Prerequisites

- MarvinsAIRA is installed and running — see [tutorial-first-time-setup.md](tutorial-first-time-setup.md) if you haven't done first-time setup yet.
- At least one Windows text-to-speech voice is installed. MarvinsAIRA doesn't ship its own voices — at startup it enumerates whatever `System.Speech` voices Windows already has installed (classic SAPI voices plus, via a reflection-based helper, higher-quality Windows OneCore voices normally reserved for Narrator) and builds its voice list from that (`App.Voice.cs:11-77`). Windows installs at least one voice by default, so this normally needs no action; if you want more voice choices, install additional voices via Windows Settings → Time & Language → Speech (outside MarvinsAIRA — there is no in-app voice installer).
- To verify alerts fire (the Verification section below), you'll need iRacing running with MarvinsAIRA connected and your car on track.

## Steps

1. **Turn on voice.** Open the **Settings** tab, then the **Voice** sub-tab. Check **Enable Voice**. This is the master switch (`Settings.EnableSpeechSynthesizer`) — with it off, nothing is spoken, including spotter callouts.
2. **Set volume and pick a voice.** On the same Voice sub-tab, drag the volume slider (1–100, `Settings.SpeechSynthesizerVolume`) and pick a voice from the dropdown (`Settings.SelectedVoice`). The dropdown lists the description of every voice Windows has installed; if the voice you had selected previously isn't found on this machine (e.g. you copied settings to a new PC), MarvinsAIRA silently falls back to the first installed voice (`App.Voice.cs:65-68`).
3. **Reveal the Spotter tab.** The Spotter tab is hidden until you turn on the **Advanced** toggle in the bottom status bar (`MainWindow.xaml.cs:1902-1938`). Turn it on.
4. **Enable the spotter.** Go to the now-visible **Spotter** tab. Check the checkbox in the tab's own header (next to the "Spotter" label) — this is `Settings.SpotterEnabled`; there's no separate enable checkbox inside the tab body. The tab is labeled in-app as "an experimental work in progress" and shows a red warning ("The voice synthesizer is currently off!") if voice isn't enabled, since spotter callouts are spoken only.
5. **Set callout frequency.** Still on the Spotter tab, use the **Callout Frequency** field/slider (1–60, default 20, per-minute — `Settings.SpotterCalloutFrequency`). This throttles how often a *persistent* car-alongside callout repeats; it doesn't affect one-off transitions (car appears/clears) or session-flag callouts.
6. **Enable chat relay (optional).** Go to **Settings → Chat** and check **Enable sending informational messages to the iRacing chat window** (`Settings.EnableChat`). This relays most spoken announcements into iRacing's in-game chat as text — but not spotter callouts (see Verification below).
7. **Customize the phrases (optional).** Go to **Settings → Messages** to edit the text of any spoken/chat phrase, grouped into **General**, **iRacing**, **Sliders**, **Spotter - Car Left Right**, and **Spotter - Session Flags** group boxes. Each row is a free-text template (e.g. Car Left → `Settings.SayCarLeft`); some accept a `:value:` token that gets substituted at speak-time. The two Spotter group boxes are only visible with the Advanced toggle from step 3 turned on.

## Verification

1. Join an iRacing session and get your car on track. `UpdateSpotter` does nothing (and resets its state) while the spotter is disabled or your car isn't on track (`App.Spotter.cs:16-24`), so no callouts fire in the pits or menus.
2. **Car-left/right callout**: get another car alongside you so iRacing's own proximity data changes. `ProcessCarLeftRight` (`App.Spotter.cs:32-92`) speaks immediately on most state changes (e.g. Clear → Car Left); note that transitions between adjacent granularities (Car Left ↔ Two Cars Left, Car Right ↔ Two Cars Right) are deliberately suppressed to avoid chatter, so going from one car alongside to two on the same side may not re-announce right away. If the car stays alongside, it re-announces after `60 / SpotterCalloutFrequency` seconds.
3. **Session-flag callout**: trigger a flag change, e.g. wait for the green flag at a race start. `ProcessSessionFlags` (`App.Spotter.cs:94-132`) only speaks on a 0→1 transition of a flag bit, so it announces once when the flag comes up and stays silent while it remains set — it won't repeat on its own.
4. Confirm you hear the phrase spoken. If chat relay is enabled, confirm general announcements (e.g. the "Connected" message) also appear in the iRacing chat window — but don't expect spotter callouts there (see Troubleshooting).

## Troubleshooting

- **Spotter tab shows a red "voice synthesizer is currently off" warning**: `Settings.EnableSpeechSynthesizer` is off. Since spotter callouts are speech-only, this means zero spotter output, even with chat relay enabled — spotter callouts never route to chat (both `Say()` calls in `ProcessCarLeftRight` and `ProcessSessionFlags` pass `alsoAddToChatQueue: false`, `App.Spotter.cs:80-85,102-126`).
- **Chat relay seems to silently do nothing**: `Chat()` requires `Settings.EnableChat` **and** an active iRacing connection **and** a known player car number, i.e. you must actually be in a car in a session, not just connected to the sim (`App.ChatQueue.cs:18-26`). Any of those missing drops the message with no error shown in the UI.
- **Chat messages lag behind what you hear**: the chat relay sends one full message at a time — it opens iRacing's chat window, types the entire string, then closes it, serially with no dedupe or priority (`App.ChatQueue.cs:28-67`). A burst of announcements (e.g. several flag changes back to back) queues up and can visibly lag behind real time; this is a design limitation, not a bug to "fix" via settings.
- **A callout you expected didn't happen**: check whether it's an adjacent-state suppression (see Verification, step 2) rather than a bug — the state-transition table in `ProcessCarLeftRight` intentionally skips re-announcing when moving between one-car and two-car-same-side states.
- **Callouts don't resume correctly after pitting or a spin**: going off track or disabling the spotter clears its last-known car-left-right state and callout timer (`App.Spotter.cs:18-24`), so callouts restart cleanly (not mid-state) once you're back on track — this is expected, not a fault.
- **Wrong or unexpectedly different voice after moving Settings.xml to another PC**: MarvinsAIRA has no installed-voice existence check beyond falling back to the first installed voice when the saved `SelectedVoice` name isn't present (`App.Voice.cs:65-68`); re-pick your voice on the Voice sub-tab after migrating settings. Failures while registering higher-quality Windows OneCore voices are also caught and swallowed silently (`App.Voice.cs:36-39`), so if those don't show up in the dropdown, there's no in-app error to explain why — this is a Windows/registry-level issue, not a MarvinsAIRA setting.

## Related documentation

- [reference-voice-audio.md](reference-voice-audio.md) — how the voice, spotter, and chat-relay subsystems work internally
- [reference-settings.md](reference-settings.md) — the full list of persisted settings fields
- [tutorial-first-time-setup.md](tutorial-first-time-setup.md) — first-time install and setup
