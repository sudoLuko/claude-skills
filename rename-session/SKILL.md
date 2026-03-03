---
name: rename-session
description: Manage friendly names for Claude Code session IDs. Use this skill whenever the user wants to save, name, list, resume, or delete named sessions. Trigger when the user says things like "save this session as...", "name this session", "list my sessions", "resume my auth-refactor session", "what sessions do I have saved", "delete session X", "rename session X to Y", "update note on session X", or any reference to session names, session bookmarks, or session aliases. Also trigger when the user mentions resuming a session by a friendly name rather than a raw session ID.
---

# Session Namer

A tool for mapping hard-to-remember Claude Code session IDs to friendly, memorable names.

## How It Works

Session mappings are stored in `~/.claude/session-names.json` as a simple JSON object:

```json
{
  "auth-refactor": {
    "session_id": "abc123-def456-ghi789",
    "created": "2026-03-01T10:30:00Z",
    "note": "Working on OAuth2 integration",
    "project": "/Users/me/my-app"
  }
}
```

## Name Validation Rules

Session names must be:
- Lowercase alphanumeric characters and hyphens only
- Between 2 and 50 characters
- No leading or trailing hyphens
- Pattern: `^[a-z0-9][a-z0-9-]{0,48}[a-z0-9]$`

## Commands

When the user asks you to manage sessions, use Bash to run the python3 scripts below. **All scripts accept arguments via `sys.argv`** — pass the values as command-line arguments, never string-replace inside the script body.

### Save / Name a Session

When the user says something like "save this session as auth-refactor" or "name this session fix-login-bug":

1. If the user did NOT provide a session ID, **auto-detect it** by running the auto-detect script first (see below) and confirm the detected ID with the user
2. Validate the name matches the naming rules above
3. Ask for an optional note describing what the session is about
4. Run the save script

**Auto-detect current session ID:**

```bash
python3 -c "
import json, os, sys

history = os.path.expanduser('~/.claude/history.jsonl')
try:
    with open(history, 'rb') as f:
        # Read last non-empty line by seeking backwards from end
        f.seek(0, 2)
        size = f.tell()
        buf = bytearray()
        pos = size
        while pos > 0:
            pos -= 1
            f.seek(pos)
            ch = f.read(1)
            if ch == b'\n':
                if buf:
                    break
            else:
                buf.append(ch[0])
        buf.reverse()
        line = buf.decode('utf-8').strip()
        if line:
            entry = json.loads(line)
            sid = entry.get('sessionId', '')
            project = entry.get('project', '')
            if sid:
                print(f'SESSION_ID={sid}')
                print(f'PROJECT={project}')
            else:
                print('ERROR=No sessionId found in last history entry')
        else:
            print('ERROR=Empty history file')
except FileNotFoundError:
    print('ERROR=History file not found at ~/.claude/history.jsonl')
except (json.JSONDecodeError, OSError) as e:
    print('ERROR=' + str(e))
"
```

Parse the output: if it starts with `ERROR=`, tell the user and ask them to provide the session ID manually. Otherwise extract `SESSION_ID` and `PROJECT` values.

**Save script** — pass 4 args: name, session_id, note, project:

```bash
python3 - "$NAME" "$SESSION_ID" "$NOTE" "$PROJECT" << 'PYEOF'
import json, os, sys, re, tempfile
from datetime import datetime, timezone

name, session_id, note, project = sys.argv[1], sys.argv[2], sys.argv[3], sys.argv[4]

# Validate name
if not re.match(r'^[a-z0-9][a-z0-9-]{0,48}[a-z0-9]$', name):
    print(f'ERROR: Invalid name "{name}". Must be 2-50 chars, lowercase alphanumeric and hyphens, no leading/trailing hyphen.')
    sys.exit(1)

filepath = os.path.expanduser('~/.claude/session-names.json')
try:
    with open(filepath) as f:
        data = json.load(f)
except (FileNotFoundError, json.JSONDecodeError):
    data = {}

# Check for duplicate session ID under a different name
for existing_name, info in data.items():
    if info.get('session_id') == session_id and existing_name != name:
        print(f'WARNING: Session ID already saved as "{existing_name}".')
        print(f'DUPLICATE={existing_name}')

# Check for existing name
if name in data:
    print(f'WARNING: Name "{name}" already exists (session {data[name]["session_id"][:12]}...).')
    print('OVERWRITE_PROMPT=true')

data[name] = {
    'session_id': session_id,
    'created': datetime.now(timezone.utc).isoformat(),
    'note': note,
    'project': project,
}

# Atomic write
os.makedirs(os.path.dirname(filepath), exist_ok=True)
fd, tmp = tempfile.mkstemp(dir=os.path.dirname(filepath), suffix='.tmp')
try:
    with os.fdopen(fd, 'w') as f:
        json.dump(data, f, indent=2)
    os.replace(tmp, filepath)
except:
    os.unlink(tmp)
    raise

print(f'Saved session "{name}" -> {session_id[:12]}...')
PYEOF
```

When using this script:
1. Set `$NAME` to the user's chosen name
2. Set `$SESSION_ID` to the session ID (auto-detected or user-provided)
3. Set `$NOTE` to the user's note (or empty string `""`)
4. Set `$PROJECT` to the detected project path (or empty string `""`)
5. If output contains `DUPLICATE=`, warn the user that this session ID is already saved under another name and ask if they want to continue
6. If output contains `OVERWRITE_PROMPT=true`, ask the user if they want to overwrite the existing entry

### Validate a Session ID

After obtaining a session ID (auto-detected or user-provided), optionally validate it exists:

```bash
python3 - "$SESSION_ID" << 'PYEOF'
import os, sys, glob

sid = sys.argv[1]
found = False

# Check in projects directory
projects_base = os.path.expanduser('~/.claude/projects')
if os.path.isdir(projects_base):
    for dirpath, dirnames, filenames in os.walk(projects_base):
        for fname in filenames:
            if fname.endswith('.jsonl') and sid in fname:
                found = True
                break
        if found:
            break

# Check in session-env directory
session_env = os.path.expanduser(f'~/.claude/session-env/{sid}')
if os.path.exists(session_env):
    found = True

if found:
    print(f'VALID: Session {sid[:12]}... found')
else:
    print(f'WARNING: Session {sid[:12]}... not found in local history. It may still be valid.')
PYEOF
```

This is a soft check — warn but don't block if the session isn't found locally.

### List Saved Sessions

When the user says "list my sessions", "what sessions do I have?", or "show saved sessions":

```bash
python3 << 'PYEOF'
import json, os

filepath = os.path.expanduser('~/.claude/session-names.json')
try:
    with open(filepath) as f:
        data = json.load(f)
except (FileNotFoundError, json.JSONDecodeError):
    print('No saved sessions yet.')
    exit()

if not data:
    print('No saved sessions yet.')
else:
    for name, info in data.items():
        note = f' -- {info["note"]}' if info.get('note') else ''
        project = f' [{info["project"]}]' if info.get('project') else ''
        print(f'  {name}: {info["session_id"][:12]}...{note}{project} (saved {info["created"][:10]})')
PYEOF
```

Present the output in a clean format to the user.

### Resume a Named Session

When the user says "resume auth-refactor" or "continue my fix-login-bug session":

1. Look up the friendly name in the JSON file
2. Retrieve the session ID
3. Tell the user to run: `claude --resume <session_id>`
4. Copy the command to clipboard if possible

```bash
python3 - "$NAME" << 'PYEOF'
import json, os, sys, subprocess, shutil, difflib

filepath = os.path.expanduser('~/.claude/session-names.json')
name = sys.argv[1]

try:
    with open(filepath) as f:
        data = json.load(f)
except (FileNotFoundError, json.JSONDecodeError):
    print('No saved sessions found.')
    sys.exit(1)

if name in data:
    sid = data[name]['session_id']
    cmd = f'claude --resume {sid}'
    print(f'To resume "{name}", run:')
    print(f'  {cmd}')

    # Try to copy to clipboard
    copied = False
    if shutil.which('pbcopy'):
        try:
            subprocess.run(['pbcopy'], input=cmd.encode(), check=True)
            copied = True
        except (subprocess.SubprocessError, OSError):
            pass
    elif shutil.which('xclip'):
        try:
            subprocess.run(['xclip', '-selection', 'clipboard'], input=cmd.encode(), check=True)
            copied = True
        except (subprocess.SubprocessError, OSError):
            pass
    elif shutil.which('xsel'):
        try:
            subprocess.run(['xsel', '--clipboard', '--input'], input=cmd.encode(), check=True)
            copied = True
        except (subprocess.SubprocessError, OSError):
            pass

    if copied:
        print('(Copied to clipboard!)')
else:
    # Fuzzy match suggestions
    matches = difflib.get_close_matches(name, data.keys(), n=3, cutoff=0.4)
    # Also check substring matches
    substr_matches = [k for k in data.keys() if name in k or k in name]
    suggestions = list(dict.fromkeys(matches + substr_matches))  # dedupe, preserve order

    print(f'No session named "{name}" found.')
    if suggestions:
        print(f'Did you mean: {", ".join(suggestions)}?')
    elif data:
        print(f'Available sessions: {", ".join(data.keys())}')
    else:
        print('No sessions saved yet.')
PYEOF
```

**Important**: You cannot directly launch `claude --resume` from within a running Claude Code session. Instead, provide the user with the exact command to run after exiting the current session, or in a new terminal.

### Delete a Named Session

When the user says "delete session auth-refactor" or "remove the fix-login-bug session":

```bash
python3 - "$NAME" << 'PYEOF'
import json, os, sys, tempfile, difflib

filepath = os.path.expanduser('~/.claude/session-names.json')
name = sys.argv[1]

try:
    with open(filepath) as f:
        data = json.load(f)
except (FileNotFoundError, json.JSONDecodeError):
    print('No saved sessions found.')
    sys.exit(1)

if name in data:
    del data[name]

    # Atomic write
    fd, tmp = tempfile.mkstemp(dir=os.path.dirname(filepath), suffix='.tmp')
    try:
        with os.fdopen(fd, 'w') as f:
            json.dump(data, f, indent=2)
        os.replace(tmp, filepath)
    except:
        os.unlink(tmp)
        raise

    print(f'Deleted session "{name}".')
else:
    # Fuzzy match suggestions
    matches = difflib.get_close_matches(name, data.keys(), n=3, cutoff=0.4)
    substr_matches = [k for k in data.keys() if name in k or k in name]
    suggestions = list(dict.fromkeys(matches + substr_matches))

    print(f'No session named "{name}" found.')
    if suggestions:
        print(f'Did you mean: {", ".join(suggestions)}?')
    elif data:
        print(f'Available sessions: {", ".join(data.keys())}')
PYEOF
```

### Rename a Session

When the user says "rename auth-refactor to oauth-migration":

```bash
python3 - "$OLD_NAME" "$NEW_NAME" << 'PYEOF'
import json, os, sys, re, tempfile, difflib

filepath = os.path.expanduser('~/.claude/session-names.json')
old_name, new_name = sys.argv[1], sys.argv[2]

# Validate new name
if not re.match(r'^[a-z0-9][a-z0-9-]{0,48}[a-z0-9]$', new_name):
    print(f'ERROR: Invalid name "{new_name}". Must be 2-50 chars, lowercase alphanumeric and hyphens, no leading/trailing hyphen.')
    sys.exit(1)

try:
    with open(filepath) as f:
        data = json.load(f)
except (FileNotFoundError, json.JSONDecodeError):
    print('No saved sessions found.')
    sys.exit(1)

if old_name not in data:
    matches = difflib.get_close_matches(old_name, data.keys(), n=3, cutoff=0.4)
    substr_matches = [k for k in data.keys() if old_name in k or k in old_name]
    suggestions = list(dict.fromkeys(matches + substr_matches))

    print(f'No session named "{old_name}" found.')
    if suggestions:
        print(f'Did you mean: {", ".join(suggestions)}?')
    sys.exit(1)

if new_name in data:
    print(f'ERROR: Name "{new_name}" already exists.')
    sys.exit(1)

data[new_name] = data.pop(old_name)

# Atomic write
fd, tmp = tempfile.mkstemp(dir=os.path.dirname(filepath), suffix='.tmp')
try:
    with os.fdopen(fd, 'w') as f:
        json.dump(data, f, indent=2)
    os.replace(tmp, filepath)
except:
    os.unlink(tmp)
    raise

print(f'Renamed "{old_name}" to "{new_name}".')
PYEOF
```

### Update Note on a Session

When the user says "update note on auth-refactor" or "change the note for my session":

```bash
python3 - "$NAME" "$NEW_NOTE" << 'PYEOF'
import json, os, sys, tempfile, difflib

filepath = os.path.expanduser('~/.claude/session-names.json')
name, new_note = sys.argv[1], sys.argv[2]

try:
    with open(filepath) as f:
        data = json.load(f)
except (FileNotFoundError, json.JSONDecodeError):
    print('No saved sessions found.')
    sys.exit(1)

if name not in data:
    matches = difflib.get_close_matches(name, data.keys(), n=3, cutoff=0.4)
    substr_matches = [k for k in data.keys() if name in k or k in name]
    suggestions = list(dict.fromkeys(matches + substr_matches))

    print(f'No session named "{name}" found.')
    if suggestions:
        print(f'Did you mean: {", ".join(suggestions)}?')
    sys.exit(1)

old_note = data[name].get('note', '')
data[name]['note'] = new_note

# Atomic write
fd, tmp = tempfile.mkstemp(dir=os.path.dirname(filepath), suffix='.tmp')
try:
    with os.fdopen(fd, 'w') as f:
        json.dump(data, f, indent=2)
    os.replace(tmp, filepath)
except:
    os.unlink(tmp)
    raise

if old_note:
    print(f'Updated note on "{name}" (was: "{old_note}").')
else:
    print(f'Added note to "{name}".')
PYEOF
```

## Edge Cases

- If `~/.claude/` directory doesn't exist, it will be created automatically on save
- If the user tries to save a name that already exists, warn them and ask if they want to overwrite
- Names must match `^[a-z0-9][a-z0-9-]{0,48}[a-z0-9]$` — reject anything else with a clear error
- If the exact session name isn't found, suggest close matches before giving up
- If a session ID is already saved under a different name, warn and let the user decide
- Auto-detect reads only the last line of history.jsonl for efficiency
- All writes use atomic temp-file + `os.replace()` to prevent corruption
- Clipboard copy is best-effort — works on macOS (pbcopy) and Linux (xclip/xsel), silently skipped otherwise

## Tips for Users

- Use descriptive names like `auth-refactor`, `fix-cart-bug`, `setup-ci-pipeline`
- Add notes when saving so you remember what you were working on
- The session ID is auto-detected — you usually don't need to find it yourself
- Use "update note" to add context to sessions you saved without a note
- The resume command is automatically copied to your clipboard on macOS
