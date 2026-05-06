---
name: bind-mic-autoload
description: Bind an Easy Effects preset to autoload when a specific microphone (or output device) becomes active. Lists current PipeWire devices, lets the user pick, and writes the autoload JSON Easy Effects expects.
---

# bind-mic-autoload

Easy Effects autoload bindings live as JSON files under
`<data>/autoload/{input,output}/<device-name>:<port>.json`. When the
matching device shows up, the named preset loads automatically.

## Inputs

- `preset_name` — must be already installed (see `install-preset`).
- `channel` — `input | output`. Default `input`.
- `device_node` — optional. If absent, list devices and ask.
- `port_name` — optional. If absent, list and ask.

## Steps

### 1. Resolve data dir

```bash
if command -v easyeffects >/dev/null 2>&1; then
  EE_DATA="${XDG_CONFIG_HOME:-$HOME/.config}/easyeffects"
elif flatpak info com.github.wwmm.easyeffects >/dev/null 2>&1; then
  EE_DATA="$HOME/.var/app/com.github.wwmm.easyeffects/data/easyeffects"
else
  echo "Easy Effects not installed."; exit 1
fi
mkdir -p "$EE_DATA/autoload/$channel"
```

### 2. List candidate devices

For input:

```bash
pw-cli ls Node | grep -A1 -E 'media.class.*=.*"Audio/Source"' | grep node.name | sort -u
```

For output:

```bash
pw-cli ls Node | grep -A1 -E 'media.class.*=.*"Audio/Sink"' | grep node.name | sort -u
```

Show the user the list and ask which `node.name` to bind. Then list
ports for that node:

```bash
pactl list sources short | awk '{print $2}' | grep -F "$device_node"
```

(Common port names: `[Out] Microphone`, `analog-stereo`, `mono-fallback`.
The exact format varies; existing autoload files in `$EE_DATA/autoload/$channel/`
are the best reference for what shape works on this box.)

### 3. Verify the preset exists

```bash
[ -f "$EE_DATA/$channel/${preset_name}.json" ] || {
  echo "Preset '${preset_name}' is not installed in channel '$channel'. Run install-preset first."
  exit 1
}
```

### 4. Write the autoload binding

The filename pattern is `<device_node>:<port_name>.json`; the body is
JSON pointing at the preset. Verify with one of the existing autoload
files (they're authoritative for the schema your Easy Effects version
expects):

```bash
ls "$EE_DATA/autoload/$channel/"
cat "$EE_DATA/autoload/$channel/"*.json | head -50
```

Match the same shape. Typically:

```bash
fname="$EE_DATA/autoload/$channel/${device_node}:${port_name}.json"
python3 - <<PY
import json
out = {
    "device": "$device_node",
    "device-description": "$device_node",
    "device-profile": "$port_name",
    "preset-name": "$preset_name"
}
open("$fname","w").write(json.dumps(out, indent=4))
PY
```

If existing autoload files use different field names, **copy from
those** rather than fabricating — Easy Effects schema has changed
between versions and the working examples on disk are the contract.

### 5. Confirm

```bash
ls -la "$EE_DATA/autoload/$channel/"
```

Tell the user the binding is in place; the preset will load next time
that device+port combination becomes active.
