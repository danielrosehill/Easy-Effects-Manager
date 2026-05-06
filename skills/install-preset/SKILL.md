---
name: install-preset
description: Copy a preset from the user's library into the live Easy Effects data dir so it can be loaded. Use after pulling new presets into the library or on a fresh workstation.
needs_user_vars:
  - name: EE_LIBRARY_PATH
    type: path
---

# install-preset

## Inputs

- `preset_name` — bare name (no `.json`).
- `channel` — `input` or `output`. Default: `input`.
- Optional `from_file` — absolute path overriding library lookup
  (useful when the user has just been handed a preset by someone else).

## Steps

### 1. Resolve library + data dir

```bash
LIB="$(cc-uvar get EE_LIBRARY_PATH)" || { echo "Run setup-library first."; exit 1; }

if command -v easyeffects >/dev/null 2>&1; then
  EE_DATA="${XDG_CONFIG_HOME:-$HOME/.config}/easyeffects"
elif flatpak info com.github.wwmm.easyeffects >/dev/null 2>&1; then
  EE_DATA="$HOME/.var/app/com.github.wwmm.easyeffects/data/easyeffects"
else
  echo "Easy Effects not installed."; exit 1
fi
mkdir -p "$EE_DATA/$channel"
```

### 2. Locate source

```bash
SRC="${from_file:-$LIB/$channel/${preset_name}.json}"
[ -f "$SRC" ] || { echo "Source preset not found: $SRC"; exit 1; }
```

### 3. Validate it's a JSON Easy Effects preset

```bash
python3 -c "
import json,sys
d=json.load(open('$SRC'))
assert isinstance(d,dict) and ('input' in d or 'output' in d), 'Not an Easy Effects preset'
print('OK schema')
"
```

### 4. Copy (refuse to overwrite without confirmation)

```bash
DEST="$EE_DATA/$channel/${preset_name}.json"
if [ -e "$DEST" ]; then
  echo "Already installed at $DEST. Overwrite? (ask user)"
  # On confirmation:
  cp "$SRC" "$DEST"
else
  cp "$SRC" "$DEST"
fi
```

### 5. Confirm visibility to Easy Effects

```bash
EE_RUN="easyeffects"; flatpak info com.github.wwmm.easyeffects >/dev/null 2>&1 && EE_RUN="flatpak run com.github.wwmm.easyeffects"
$EE_RUN -p | grep -q "$preset_name" && echo "Visible." || echo "Not visible — restart Easy Effects."
```

Tell the user whether they need to restart, and offer to call
`load-preset` next.
