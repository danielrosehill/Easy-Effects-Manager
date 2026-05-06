---
name: load-preset
description: Activate an Easy Effects preset by name. Wraps `easyeffects -l <name>` (or the Flatpak equivalent) and verifies it loaded by reading back the last-loaded preset.
---

# load-preset

## Inputs

- `preset_name` — the bare preset name (no `.json`). Must match an
  installed preset (see `list-presets`).

## Steps

### 1. Detect the runner

```bash
if command -v easyeffects >/dev/null 2>&1; then
  EE_RUN="easyeffects"
elif flatpak info com.github.wwmm.easyeffects >/dev/null 2>&1; then
  EE_RUN="flatpak run com.github.wwmm.easyeffects"
else
  echo "Easy Effects not installed."
  exit 1
fi
```

### 2. Confirm the preset exists in the data dir

If the user hasn't run `install-preset` for a library-only preset,
loading it will silently fail. Cross-check before invoking:

```bash
if [ "$EE_RUN" = "easyeffects" ]; then
  EE_DATA="${XDG_CONFIG_HOME:-$HOME/.config}/easyeffects"
else
  EE_DATA="$HOME/.var/app/com.github.wwmm.easyeffects/data/easyeffects"
fi
[ -f "$EE_DATA/input/${preset_name}.json" ] || [ -f "$EE_DATA/output/${preset_name}.json" ] \
  || { echo "No installed preset named '${preset_name}'. Run install-preset first."; exit 1; }
```

### 3. Load it

```bash
$EE_RUN -l "$preset_name"
```

Easy Effects must already be running (or in service mode) for this to
take effect immediately. If it's not, start it first:

```bash
pgrep -x easyeffects >/dev/null || $EE_RUN --hide-window &
sleep 1
$EE_RUN -l "$preset_name"
```

### 4. Verify

```bash
$EE_RUN -s
```

The output should reference `${preset_name}` for the matching channel
(input or output). If not, surface the discrepancy to the user.
