<!-- Generated: 2026-07-12 | Files scanned: 42 -->
# Dependencies

## Sibling projects NOT in this repo (required to build)

`MarvinsAIRA.csproj` `<ProjectReference>`s two projects by relative path that are
absent from this repository (verified: `dotnet build` fails with CS0246 on
`IRacingSdk`/`IRacingSdkDatum`/`IRacingSdkEnum`/`HPR` without them):

```
..\IRSDKSharper\IRSDKSharper.csproj  → IRacingSdk, IRacingSdkDatum, IRacingSdkEnum
                                        (shared-memory iRacing telemetry SDK wrapper)
                                        consumed by App.IRacingSDK.cs, App.Spotter.cs,
                                        App.ChatQueue.cs
..\SimagicHPR\SimagicHPR.csproj      → HPR class (Simagic pedal-haptics hardware SDK)
                                        consumed by App.SimagicHPR.cs
```

`MarvinsAIRA.sln` (not the `.csproj`) also references a third sibling, not needed
to build the main exe:

```
..\MarvinsAIRASimHub\MarvinsAIRASimHub.csproj → SimHub plugin DLL, copied into
                                                  {userdocs}\MarvinsAIRA by the
                                                  installer, built/shipped separately
```

See [tutorial-build-and-run-from-source.md](../tutorial-build-and-run-from-source.md)
for the exact build failure this produces and how to resolve it.

## NuGet packages (`MarvinsAIRA.csproj`)

```
SharpDX.DirectInput / SharpDX.DirectSound / SharpDX.XAudio2  → wheel FFB, joystick
                                                                  input, LFE audio capture
NAudio                    → sound cues, ABS pitch-shifting
System.Speech             → text-to-speech (App.Voice.cs)
System.IO.Ports           → Arduino serial (wind simulator)
Newtonsoft.Json           → update-check API payloads
System.Net.Http           → update-check HTTP calls
ModernWpfUI / Extended.Wpf.Toolkit → UI controls
Hardcodet.NotifyIcon.Wpf  → system tray icon
BootMeUp                  → auto-start registry entry (current user, no elevation)
```

## Native / hardware integrations

```
LogitechSteeringWheelEnginesWrapper.dll → Logitech wheel RPM shift-light LEDs
                                            (App.Logitech.cs, LogitechGSDK.cs)
Arduino (serial, MarvinBox/MarvinBox.ino) → wind fan control (App.WindSimulator.cs)
vJoyInterfaceWrap.dll                     → shipped as content, UNUSED (dead code,
                                              see FFBReceiver.cs exclusion)
```

## External network services

```
https://herboldracing.com/wp-json/maira/v1/get-current-version
    → self-update version check (App.Service.cs), NOT GitHub-based
github.com/mherbold/MarvinsAIRA
    → referenced only as the "view source" link in the Contribute tab, not an update channel
```

## Related maps

- [architecture.md](architecture.md) — how these dependencies are wired into startup
