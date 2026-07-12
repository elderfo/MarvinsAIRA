# MarvinsAIRA
 
This is the repo for the original classic (1.x) Marvin's Awesome iRacing App.  This app is no longer being updated.

The repo for the latest app, MAIRA Refactored (2.x), can be found here -

https://github.com/mherbold/MarvinsAIRARefactored

## Documentation

Documentation for this legacy codebase lives in [`docs/`](docs/). For a fast,
token-lean structural overview (good for AI context loading), see
[`docs/CODEMAPS/`](docs/CODEMAPS/): [architecture](docs/CODEMAPS/architecture.md),
[dependencies](docs/CODEMAPS/dependencies.md), [frontend](docs/CODEMAPS/frontend.md).

**Tutorials** — learning by doing, start to finish:

- [Build and run from source](docs/tutorial-build-and-run-from-source.md) — for developers: clone, build, run, make a verified change
- [First-time installation and setup](docs/tutorial-first-time-setup.md) — for end users: install, launch, confirm iRacing telemetry is flowing

**How-to guides** — task-oriented, assumes basic familiarity:

- [Add a new wheel/peripheral device integration](docs/howto-add-wheel-device-integration.md)
- [Add a new force-feedback effect](docs/howto-add-force-feedback-effect.md)
- [Add a new sound cue](docs/howto-add-sound-cue.md)
- [Configure force feedback](docs/howto-configure-force-feedback.md)
- [Set up voice alerts](docs/howto-set-up-voice-alerts.md)
- [Map wheel buttons to app actions](docs/howto-map-wheel-buttons.md)

**Reference** — complete technical description:

- [Architecture reference](docs/reference-architecture.md) — entry point, subsystem map, startup/update-loop, external dependencies, known quirks
- [Settings & persistence reference](docs/reference-settings.md)
- [iRacing SDK & telemetry reference](docs/reference-telemetry.md)
- [Force feedback, inputs & wind simulator reference](docs/reference-force-feedback.md)
- [Wheel device integrations reference](docs/reference-device-integrations.md) (Logitech, Simagic HPR)
- [Voice, spotter, chat & sound cues reference](docs/reference-voice-audio.md)
- [UI, dialogs & distribution reference](docs/reference-ui.md)

**Explanation** — design rationale:

- [Why the codebase is shaped this way](docs/explanation-architecture.md)
- [Why force feedback works this way](docs/explanation-force-feedback-design.md)
- [Why telemetry sync works this way](docs/explanation-telemetry-sync.md)
