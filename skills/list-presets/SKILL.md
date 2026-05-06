---
name: list-presets
description: List Easy Effects presets — what's loaded right now, what's in the live data dir (installed but not active), and what's in the user's library. Use to figure out what's available before loading, installing, or exporting.
needs_user_vars:
  - name: EE_LIBRARY_PATH
    type: path
    prompt: "Library path (run setup-library first if unsure)"
---

# list-presets

## Steps

### 1. Detect install + data dir

```bash
if command -v easyeffects >/dev/null 2>&1; then
  EE_RUN="easyeffects"
  EE_DATA="${XDG_CONFIG_HOME:-$HOME/.config}/easyeffects"
elif flatpak info com.github.wwmm.easyeffects >/dev/null 2>&1; then
  EE_RUN="flatpak run com.github.wwmm.easyeffects"
  EE_DATA="$HOME/.var/app/com.github.wwmm.easyeffects/data/easyeffects"
else
  echo "Easy Effects not installed."
  exit 1
fi
```

### 2. Resolve library

```bash
LIB="$(cc-uvar get EE_LIBRARY_PATH)" || {
  echo "Library not configured. Run the setup-library skill first."
  exit 1
}
```

### 3. Show what's loaded right now

```bash
$EE_RUN -s 2>/dev/null || $EE_RUN -a input; $EE_RUN -a output
```

(`-s` prints both last-loaded; `-a input|output` prints individually.
Some builds support one but not the other.)

### 4. List installed (data-dir) presets

```bash
echo "=== Installed input ==="
ls -1 "$EE_DATA/input/" 2>/dev/null
echo "=== Installed output ==="
ls -1 "$EE_DATA/output/" 2>/dev/null
echo "=== Installed autoload (input) ==="
ls -1 "$EE_DATA/autoload/input/" 2>/dev/null
echo "=== Installed autoload (output) ==="
ls -1 "$EE_DATA/autoload/output/" 2>/dev/null
```

### 5. List library presets

```bash
echo "=== Library input ==="
ls -1 "$LIB/input/" 2>/dev/null
echo "=== Library output ==="
ls -1 "$LIB/output/" 2>/dev/null
echo "=== Library autoload ==="
ls -1 "$LIB/autoload/input/" "$LIB/autoload/output/" 2>/dev/null
```

### 6. Report

Summarize as a table for the user: preset name, where it lives
(installed / library / both), and which one is currently active.
