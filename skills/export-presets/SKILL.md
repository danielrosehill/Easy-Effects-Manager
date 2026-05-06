---
name: export-presets
description: Back up Easy Effects presets from the live data dir into the user's library. Optionally git-commit and push if the library is configured as a git repo. Use after creating or tweaking a preset to capture it.
needs_user_vars:
  - name: EE_LIBRARY_PATH
    type: path
  - name: EE_LIBRARY_IS_GIT_REPO
    type: bool
---

# export-presets

## Inputs

- `preset_name` — optional. If omitted, exports **all** presets.
- `channel` — `input | output | autoload | all`. Default: `all`.
- `commit_message` — optional override. Default:
  `"Export presets ($(date -u +%Y-%m-%dT%H:%M:%SZ))"`.

## Steps

### 1. Resolve library + data dir + git flag

```bash
LIB="$(cc-uvar get EE_LIBRARY_PATH)" || { echo "Run setup-library first."; exit 1; }
IS_GIT="$(cc-uvar get EE_LIBRARY_IS_GIT_REPO)" || IS_GIT="false"

if command -v easyeffects >/dev/null 2>&1; then
  EE_DATA="${XDG_CONFIG_HOME:-$HOME/.config}/easyeffects"
elif flatpak info com.github.wwmm.easyeffects >/dev/null 2>&1; then
  EE_DATA="$HOME/.var/app/com.github.wwmm.easyeffects/data/easyeffects"
else
  echo "Easy Effects not installed."; exit 1
fi
```

### 2. Pick channels

```bash
case "$channel" in
  input)   chans=(input) ;;
  output)  chans=(output) ;;
  autoload) chans=(autoload/input autoload/output) ;;
  all|"")  chans=(input output autoload/input autoload/output) ;;
  *) echo "Bad channel: $channel"; exit 1 ;;
esac
```

### 3. Copy

```bash
for c in "${chans[@]}"; do
  src="$EE_DATA/$c"
  dst="$LIB/$c"
  [ -d "$src" ] || continue
  mkdir -p "$dst"
  if [ -n "$preset_name" ]; then
    f="$src/${preset_name}.json"
    [ -f "$f" ] && cp -v "$f" "$dst/"
  else
    cp -v "$src"/*.json "$dst/" 2>/dev/null || true
  fi
done
```

### 4. If git repo: stage + commit + push

Only if `IS_GIT=true`:

```bash
if [ "$IS_GIT" = "true" ] && [ -d "$LIB/.git" ]; then
  ( cd "$LIB" && git add -A && \
    git diff --cached --quiet && echo "No changes to commit." || \
    git commit -m "${commit_message:-Export presets ($(date -u +%Y-%m-%dT%H:%M:%SZ))}" )
  if ( cd "$LIB" && git remote get-url origin >/dev/null 2>&1 ); then
    ( cd "$LIB" && git push )
  fi
fi
```

### 5. Report

- Files copied (list)
- Git status (committed? pushed? remote URL?)
- Library inventory after the export
