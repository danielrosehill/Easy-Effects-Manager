---
name: test-input-level
description: Record a short sample from the active input device, compute peak/RMS levels, and recommend gain adjustments. Use to verify a mic preset is producing clean, well-leveled audio for voice dictation or recording.
---

# test-input-level

## Inputs

- `duration_sec` — recording length. Default `5`.
- `source_name` — optional PipeWire/Pulse source name. If absent,
  records from the default source.

## Steps

### 1. Pick a tool

Prefer `parecord` (pulseaudio/pipewire-pulse) since it's near-universal
on modern Linux. Fall back to `pw-record`. Confirm one is installed:

```bash
command -v parecord >/dev/null || command -v pw-record >/dev/null \
  || { echo "Need parecord or pw-record. Install pulseaudio-utils or pipewire-cli."; exit 1; }
command -v sox >/dev/null \
  || { echo "Need sox for level analysis. Install with your package manager."; exit 1; }
```

### 2. Record

```bash
TMP=$(mktemp --suffix=.wav)
trap 'rm -f "$TMP"' EXIT

if [ -n "$source_name" ]; then
  parecord --device="$source_name" --file-format=wav "$TMP" &
else
  parecord --file-format=wav "$TMP" &
fi
REC_PID=$!
sleep "${duration_sec:-5}"
kill "$REC_PID" 2>/dev/null
wait "$REC_PID" 2>/dev/null || true
```

Tell the user when recording starts and stops — they need to be
talking at a normal level during the window.

### 3. Analyze

```bash
sox "$TMP" -n stats 2>&1
```

`sox stats` reports peak (`Pk lev dB`), RMS (`RMS lev dB`),
DC offset, crest factor, etc.

### 4. Recommend

Apply these heuristics:

- **Peak above -3 dB** → too hot. Reduce input gain or autogain target.
- **Peak below -20 dB** → too quiet. Raise gain.
- **Sweet spot for voice dictation:** peak between -12 and -6 dB,
  RMS around -20 to -16 dB.
- **DC offset > 0.01** → check that DC removal / high-pass is on in
  the active preset.
- **Crest factor > 20 dB** → very dynamic; compression / autogain
  may help dictation accuracy.

Report the numbers and the recommendation. Offer to:

1. Re-run after the user adjusts gain in the OS mixer or preset
2. Run `voice-dictation-setup` if no clean preset is loaded yet
