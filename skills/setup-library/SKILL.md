---
name: setup-library
description: Onboard Easy-Effects-Manager — capture the user's preset library path (and whether it's a git repo) into the user-var store, then scaffold the directory tree. Run once per workstation before using any other skill in this plugin. Idempotent.
needs_user_vars:
  - name: EE_LIBRARY_PATH
    type: path
    prompt: "Where do you want to keep your Easy Effects preset library? (filesystem path)"
    default: "~/easy-effects-library"
  - name: EE_LIBRARY_IS_GIT_REPO
    type: bool
    prompt: "Is that library a git repo we should commit + push to after exports?"
    default: false
---

# setup-library

Captures two user-vars via the Bridge `cc-uvar` CLI (path pinned in
SessionStart) and scaffolds the on-disk tree.

## Steps

### 1. Verify Easy Effects is installed

```bash
if command -v easyeffects >/dev/null 2>&1; then
  EE_KIND=native
elif flatpak info com.github.wwmm.easyeffects >/dev/null 2>&1; then
  EE_KIND=flatpak
else
  echo "Easy Effects is not installed. Install via your package manager or 'flatpak install flathub com.github.wwmm.easyeffects'."
  exit 1
fi
echo "Detected: $EE_KIND"
```

### 2. Resolve `EE_LIBRARY_PATH`

```bash
LIB="$(cc-uvar get EE_LIBRARY_PATH)" || LIB=""
```

If empty, ask the user:

> "Where do you want to keep your Easy Effects preset library? (default: `~/easy-effects-library`)"

Expand `~` to `$HOME` if present, then persist:

```bash
LIB="${ANSWER/#\~/$HOME}"
cc-uvar set EE_LIBRARY_PATH "$LIB"
```

### 3. Resolve `EE_LIBRARY_IS_GIT_REPO`

```bash
IS_GIT="$(cc-uvar get EE_LIBRARY_IS_GIT_REPO)" || IS_GIT=""
```

If empty, ask:

> "Is that library a git repo we should commit + push to after exports? (y/n)"

```bash
case "$ANSWER" in
  y|yes|true) cc-uvar set EE_LIBRARY_IS_GIT_REPO true ;;
  *)          cc-uvar set EE_LIBRARY_IS_GIT_REPO false ;;
esac
```

### 4. Scaffold the library tree

```bash
mkdir -p "$LIB"/{input,output,autoload/input,autoload/output,backups}
```

If `EE_LIBRARY_IS_GIT_REPO=true` and the directory isn't a git repo
yet, ask the user before running `git init`:

> "The library at $LIB is not a git repo. Initialize one? (y/n)"

If yes:

```bash
( cd "$LIB" && git init -q && [ ! -f .gitignore ] && printf 'backups/*.tar.gz\n' > .gitignore )
```

Don't create a remote or commit — that's the user's call.

### 5. Report

Summarize:

- Easy Effects install kind (native / flatpak) and data dir
- Library path
- `is_git_repo` flag
- Existing preset counts in both the live data dir and the library
  (input + output + autoload)

```bash
ls -1 "$LIB/input" 2>/dev/null | wc -l
ls -1 "$LIB/output" 2>/dev/null | wc -l
```

## Notes

- Re-running this skill is safe — values already in the store are
  reused, no re-prompting unless `cc-uvar unset` was used.
- To change the library location later, run `cc-uvar unset
  EE_LIBRARY_PATH` then re-run this skill.
