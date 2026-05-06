---
name: tweak-preset
description: Open or modify a single value in an installed Easy Effects preset JSON file, then reload it. Useful for quick adjustments (autogain target, threshold, gain) without launching the full GUI. For structural edits, use the GUI.
---

# tweak-preset

## Inputs

- `preset_name` — installed preset (no `.json`).
- `channel` — `input | output`. Default `input`.
- `path` — dot-path into the JSON, e.g.
  `output.autogain.target` or `input.compressor.threshold`.
- `value` — the new value (number, bool, or string).

## Steps

### 1. Locate the preset file

```bash
if command -v easyeffects >/dev/null 2>&1; then
  EE_DATA="${XDG_CONFIG_HOME:-$HOME/.config}/easyeffects"
  EE_RUN="easyeffects"
elif flatpak info com.github.wwmm.easyeffects >/dev/null 2>&1; then
  EE_DATA="$HOME/.var/app/com.github.wwmm.easyeffects/data/easyeffects"
  EE_RUN="flatpak run com.github.wwmm.easyeffects"
else
  echo "Easy Effects not installed."; exit 1
fi
F="$EE_DATA/$channel/${preset_name}.json"
[ -f "$F" ] || { echo "Not found: $F"; exit 1; }
```

### 2. Show the user what's there now

```bash
python3 -c "
import json,sys
d=json.load(open('$F'))
keys='$path'.split('.')
v=d
for k in keys: v=v[k]
print('Current value:', v)
"
```

If the path doesn't resolve, surface the structure (top two levels)
so the user can correct the dot-path:

```bash
python3 -c "
import json
d=json.load(open('$F'))
def walk(d,prefix=''):
  if isinstance(d,dict):
    for k,v in d.items():
      p = f'{prefix}.{k}' if prefix else k
      if isinstance(v,(dict,list)): walk(v,p)
      else: print(p, '=', v)
walk(d)
" | head -60
```

### 3. Apply the edit (with backup)

```bash
cp "$F" "$F.bak"
python3 - <<PY
import json, sys
d=json.load(open("$F"))
keys="$path".split(".")
*head, last = keys
node = d
for k in head: node = node[k]
# coerce numeric/bool from string
raw = "$value"
try:
    new = json.loads(raw)
except Exception:
    new = raw
node[last] = new
json.dump(d, open("$F","w"), indent=4)
print("Set", "$path", "=", new)
PY
```

### 4. Reload

```bash
$EE_RUN -l "$preset_name"
```

### 5. Offer the next step

- Run `test-input-level` (input channel) to verify the change does
  what the user expected.
- Run `export-presets` to back the change up to the library.
- If the change was bad, restore from `$F.bak` and reload.
