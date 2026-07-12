# Build and Run MarvinsAIRA from Source

In this tutorial you'll build MarvinsAIRA from source on Windows, launch the compiled app, and make
a one-line change to its logging output that you'll watch show up the next time you run it. By the
end you'll have a working `MarvinsAIRA.exe`, you'll know exactly why the first build attempt fails on
a fresh checkout, and you'll have proven to yourself that you can edit the code and see the result.

> **Status:** this is the legacy 1.x codebase and is no longer actively developed. The maintained
> successor is [MAIRA Refactored (2.x)](https://github.com/mherbold/MarvinsAIRARefactored). This
> tutorial builds the code as it exists in this repo — see
> [reference-architecture.md](reference-architecture.md) and
> [explanation-architecture.md](explanation-architecture.md) for the bigger picture.

## What you'll need

- **Windows.** MarvinsAIRA is a WPF app targeting `net8.0-windows10.0.26100.0`
  (`MarvinsAIRA.csproj:5`) — it only builds and runs on Windows.
- **A .NET SDK that can build that target framework.** The officially referenced SDK is .NET 8. This
  tutorial was verified with `dotnet --version` reporting `10.0.109`, which builds
  `net8.0-windows10.0.26100.0` without issue — a .NET 10 SDK builds older `net8.0`-family targets
  fine, so you don't need to install .NET 8 side-by-side if you already have a newer SDK.
- **Two sibling repositories you don't have yet.** `MarvinsAIRA.csproj` references two projects by
  relative path that live *outside* this repository and are not included in it:
  - `..\IRSDKSharper\IRSDKSharper.csproj`
  - `..\SimagicHPR\SimagicHPR.csproj`

  These are external projects (per
  [reference-architecture.md's "External dependencies not in this repo"](reference-architecture.md#external-dependencies-not-in-this-repo)),
  not something published in this repository or fetchable via NuGet. You need your own checkouts of
  both, placed as siblings of this repo's folder, i.e. given this repo at `C:\dev\MarvinsAIRA`, you'd
  need `C:\dev\IRSDKSharper` and `C:\dev\SimagicHPR`. Step 1 below shows you exactly what happens if
  you skip this — that failure is itself part of the tutorial, not something to route around.

## Step 1: build on a fresh checkout, and read the failure

From the repo root, build just the app project (not the `.sln` — the solution file also references a
third sibling, `..\MarvinsAIRASimHub`, that isn't needed to build the main app):

```
dotnet build MarvinsAIRA.csproj -c Debug
```

NuGet package restore succeeds — no `NU*` errors. Compilation then fails. Here's a trimmed excerpt of
the real output (46 errors total; the pattern below repeats for most of the fields
`App.IRacingSDK.cs` declares):

```
C:\dev\MarvinsAIRA\MarvinsAIRA.csproj : warning NU1701: Package 'BootMeUp 1.2.0' was restored using
'.NETFramework,Version=v4.6.1, ...' instead of the project target framework
'net8.0-windows10.0.26100'. This package may not be fully compatible with your project.
C:\Program Files\dotnet\sdk\10.0.109\Microsoft.Common.CurrentVersion.targets(2189,5): warning MSB9008:
The referenced project ..\IRSDKSharper\IRSDKSharper.csproj does not exist. [C:\dev\MarvinsAIRA\MarvinsAIRA.csproj]

C:\dev\MarvinsAIRA\App.ChatQueue.cs(5,7): error CS0246: The type or namespace name 'IRSDKSharper' could
not be found (are you missing a using directive or an assembly reference?) [...wpftmp.csproj]
C:\dev\MarvinsAIRA\App.SimagicHPR.cs(6,7): error CS0246: The type or namespace name 'Simagic' could not
be found (are you missing a using directive or an assembly reference?) [...wpftmp.csproj]
C:\dev\MarvinsAIRA\App.IRacingSDK.cs(24,11): error CS0246: The type or namespace name 'IRacingSdk'
could not be found (are you missing a using directive or an assembly reference?) [...wpftmp.csproj]
C:\dev\MarvinsAIRA\App.IRacingSDK.cs(26,11): error CS0246: The type or namespace name 'IRacingSdkDatum'
could not be found (are you missing a using directive or an assembly reference?) [...wpftmp.csproj]
... (repeats for every IRacingSdkDatum field, lines 26-59)
C:\dev\MarvinsAIRA\App.IRacingSDK.cs(71,10): error CS0246: The type or namespace name 'IRacingSdkEnum'
could not be found (are you missing a using directive or an assembly reference?) [...wpftmp.csproj]
C:\dev\MarvinsAIRA\App.Spotter.cs(4,7): error CS0246: The type or namespace name 'IRSDKSharper' could
not be found (are you missing a using directive or an assembly reference?) [...wpftmp.csproj]
C:\dev\MarvinsAIRA\App.Spotter.cs(10,20): error CS0246: The type or namespace name 'IRacingSdkEnum'
could not be found (are you missing a using directive or an assembly reference?) [...wpftmp.csproj]
C:\dev\MarvinsAIRA\App.SimagicHPR.cs(29,20): error CS0246: The type or namespace name 'HPR' could not
be found (are you missing a using directive or an assembly reference?) [...wpftmp.csproj]

Build FAILED.
    46 Error(s)
```

This is a real, informative first result, not a dead end. Every one of the 46 `CS0246` errors is the
compiler failing to find a type — `IRSDKSharper`/`IRacingSdk`/`IRacingSdkDatum`/`IRacingSdkEnum`, or
`Simagic`/`HPR` — that only exists in one of the two missing sibling projects. NuGet restore had
nothing to do with it (those packages all resolved fine); the `MSB9008` warning names the actual
cause: `..\IRSDKSharper\IRSDKSharper.csproj` doesn't exist on disk. Once you have both sibling repos
checked out next to this one, every one of these errors disappears, because the types they're
missing get supplied by those projects' own compiled output.

## Step 2: build again with the sibling projects in place

**Precondition:** you have your own checkouts of `IRSDKSharper` and `SimagicHPR` sitting next to this
repo (`..\IRSDKSharper\IRSDKSharper.csproj` and `..\SimagicHPR\SimagicHPR.csproj` both resolve). This
tutorial can't walk you through obtaining those two projects — they're external and not part of this
repository — but once they exist at those paths, rebuild the same way:

```
dotnet build MarvinsAIRA.csproj -c Debug
```

This time restore and compilation both succeed, and MSBuild produces the executable using the
standard SDK-style output pattern derived from this project's `<OutputType>WinExe</OutputType>` and
`<TargetFramework>net8.0-windows10.0.26100.0</TargetFramework>` (`MarvinsAIRA.csproj:4-5`):

```
bin\Debug\net8.0-windows10.0.26100.0\MarvinsAIRA.exe
```

That's the file you'll run in Step 3.

## Step 3: run the app and make a small, visible change

### Run it

```
.\bin\Debug\net8.0-windows10.0.26100.0\MarvinsAIRA.exe
```

The main window opens whether or not iRacing is running. WPF constructs and shows `MainWindow`
automatically via `StartupUri="MainWindow.xaml"` (`App.xaml:6`) as soon as `App`'s constructor
finishes acquiring its single-instance mutex (`App.xaml.cs:19,25`) — that happens before any
subsystem is touched. Real subsystem startup, including the iRacing SDK connection, only begins
afterward, from `MainWindow.Window_Activated` calling `app.Initialize(_win_windowHandle)`
(`MainWindow.xaml.cs:198`). And that connection attempt doesn't block or throw if iRacing isn't
running: `InitializeIRacingSDK` (`App.IRacingSDK.cs:132-147`) just registers callbacks and calls
`_irsdk.Start()`, which "is now waiting for the iRacing Simulator" (`App.IRacingSDK.cs:146`) rather
than requiring it to already be there. So you'll see the window, the tray icon, and a live-updating
Console tab — you just won't see telemetry-driven force feedback or spotter callouts without iRacing
actually running.

### Make a change

Open `App.Console.cs`. Every log line the app writes — to its in-window Console tab and to
`%UserProfile%\Documents\MarvinsAIRA\Console.log` (`App.xaml.cs:12,14`, `App.Console.cs:18`) — is
built from one line inside `WriteLine`:

```csharp
// App.Console.cs:47
var messageWithTime = $"{blankLine}{DateTime.Now}   {message}";
```

Add a marker to it:

```csharp
var messageWithTime = $"{blankLine}{DateTime.Now}   [tutorial] {message}";
```

Save, close the running app (if it's still open), and rebuild:

```
dotnet build MarvinsAIRA.csproj -c Debug
```

Run it again:

```
.\bin\Debug\net8.0-windows10.0.26100.0\MarvinsAIRA.exe
```

Look at the Console tab in the window — every line now reads `... [tutorial] ...`. Close the app and
open `%UserProfile%\Documents\MarvinsAIRA\Console.log` in a text editor; the same marker is there
too, since both destinations are built from the same `messageWithTime` string
(`App.Console.cs:59,81`). That's the full loop: source change → rebuild → observable, verifiable
effect, in both places the app logs to.

## What you built

You now have a working `MarvinsAIRA.exe` built from source, you know precisely why a bare checkout
doesn't compile (two external sibling projects supplying SDK/hardware types), and you've confirmed
you can change a line of code and see it take effect in a running instance of the app.

For the bigger picture of how the app is put together, read
[reference-architecture.md](reference-architecture.md) (subsystem layout, startup sequence, main
update loop) and [explanation-architecture.md](explanation-architecture.md) (why it's structured this
way). When you're ready to make a real change rather than a logging tweak, the how-to guides below
walk through the common cases:

- [howto-add-wheel-device-integration.md](howto-add-wheel-device-integration.md)
- [howto-add-force-feedback-effect.md](howto-add-force-feedback-effect.md)
- [howto-add-sound-cue.md](howto-add-sound-cue.md)

## Related documentation

- [reference-architecture.md](reference-architecture.md) — subsystem layout, startup sequence, external dependencies
- [explanation-architecture.md](explanation-architecture.md) — why the codebase is shaped this way
- [howto-add-wheel-device-integration.md](howto-add-wheel-device-integration.md) — add a new wheel device integration
- [howto-add-force-feedback-effect.md](howto-add-force-feedback-effect.md) — add a new force-feedback effect
- [howto-add-sound-cue.md](howto-add-sound-cue.md) — add a new sound cue
