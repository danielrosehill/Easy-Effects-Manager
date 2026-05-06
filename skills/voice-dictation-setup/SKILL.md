---
name: voice-dictation-setup
description: Opinionated end-to-end setup for clean voice dictation input — confirm Easy Effects is running, ensure a noise-reduction + autogain preset is active, run a level test, and report whether the chain is dictation-ready.
needs_user_vars:
  - name: EE_LIBRARY_PATH
    type: path
---

# voice-dictation-setup

A one-shot guided flow for "make my mic dictation-ready." Composes
other skills in this plugin; doesn't reimplement them.

## Steps

### 1. Confirm Easy Effects is running

```bash
if command -v easyeffects >/dev/null 2>&1; then
  EE_RUN="easyeffects"
elif flatpak info com.github.wwmm.easyeffects >/dev/null 2>&1; then
  EE_RUN="flatpak run com.github.wwmm.easyeffects"
else
  echo "Easy Effects not installed."; exit 1
fi
pgrep -f easyeffects >/dev/null || $EE_RUN --hide-window &
sleep 1
```

### 2. Identify the active mic

```bash
pactl get-default-source
```

Confirm with the user that this is the mic they want to use for
dictation. If wrong, point them at `pavucontrol` or KDE / GNOME sound
settings to switch the default, then resume.

### 3. Ensure a noise-reduction preset is loaded

Run the `list-presets` skill (mentally — don't recursively invoke;
just call its commands inline) and look for a preset whose name
suggests it includes:

- An RNNoise / NoiseTorch / DeepFilter / Speex stage (denoise)
- A high-pass at ~80–100 Hz
- An autogain or compressor stage

Common names: "Speech Processing", "Voice Dictation", "Clean Voice",
"NR + AutoGain". If none exists in `$EE_DATA/input/`:

- Check the user's library (`$LIB/input/`) — if a candidate is there,
  call `install-preset`.
- If nothing suitable exists anywhere, surface the gap to the user
  and offer to create a basic preset. (Authoring presets is best done
  in the Easy Effects GUI; the plugin doesn't try to synthesize one.)

Once a candidate is identified, load it via the `load-preset` skill
pattern:

```bash
$EE_RUN -l "$preset_name"
$EE_RUN -a input
```

### 4. Test levels

Run the `test-input-level` skill flow with `duration_sec=6`. Ask the
user to read a sentence aloud during the window.

### 5. Verdict

Tell the user one of:

- **Ready** — preset loaded, peak in [-12, -6] dB, RMS in [-20, -16]
  dB, no DC offset issue. Dictation should perform well.
- **Loaded but too hot/quiet** — preset is fine, but gain is wrong.
  Suggest the specific adjustment (autogain target, OS mixer level).
  Offer to re-test.
- **No suitable preset** — user needs to author one in the GUI.
  Sketch the chain (HPF → RNNoise → Autogain) and stop.

### 6. Optional: bind it to the mic

If the user is happy and wants this to load automatically for that
mic from now on, offer to call `bind-mic-autoload` with the chosen
preset and the active source's `node.name` + port.
