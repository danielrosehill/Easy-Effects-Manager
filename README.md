# Easy-Effects-Manager

Claude-Controller plugin for managing [Easy Effects](https://github.com/wwmm/easyeffects)
on Linux: maintain a preset library, install/export presets, bind autoload to
specific microphones, test input levels, and set up clean voice-dictation chains.

Works with both native and Flatpak installs of Easy Effects (auto-detected).

## Skills

| Skill | What it does |
|---|---|
| `setup-library` | One-time onboarding. Captures your library path + git-repo flag via the Bridge `cc-uvar` user-var store. Run first. |
| `list-presets` | Shows what's loaded now, what's installed, and what's in your library. |
| `load-preset` | Activates an installed preset. |
| `install-preset` | Copies a preset from your library into the live Easy Effects data dir. |
| `export-presets` | Backs presets up from the live data dir into your library. Optional git commit + push. |
| `bind-mic-autoload` | Binds a preset to a specific mic so it auto-loads when that mic is active. |
| `test-input-level` | Records a short sample, reports peak/RMS via `sox stats`, recommends adjustments. |
| `voice-dictation-setup` | Opinionated end-to-end "make my mic dictation-ready" flow. |
| `tweak-preset` | Quick numerical edit to a single value in a preset JSON, then reload. |

## Configuration

Two user-vars (captured automatically by `setup-library`):

- `EE_LIBRARY_PATH` — where preset backups + curated presets live on disk.
- `EE_LIBRARY_IS_GIT_REPO` — whether `export-presets` should `git commit && push` after writing.

Both live in `~/.config/claude-controller/user-vars/<identity>.json` (the
Claude-Controller user-var store, per `decisions/0008-user-vars-contract.md`
in the Claude-Controller repo). Manage directly via:

```bash
cc-uvar list
cc-uvar set EE_LIBRARY_PATH /new/path
cc-uvar unset EE_LIBRARY_PATH    # forces re-prompt next run
```

## Dependencies

- Easy Effects (`easyeffects` package, or `flatpak install flathub com.github.wwmm.easyeffects`)
- `python3` (already installed on every modern Linux)
- For `test-input-level`: `parecord` (pulseaudio-utils or pipewire-pulse) and `sox`
- For `export-presets` with `EE_LIBRARY_IS_GIT_REPO=true`: `git` with a configured remote

## Install

This plugin is consumed via Claude-Controller. Subscribe with:

```
admin-subscribe Easy-Effects-Manager
```

then `bridge-reload-index` to refresh the SessionStart pin.

## License

MIT
