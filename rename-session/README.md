# rename-session

A Claude Code skill for managing friendly names for session IDs. Stop copy-pasting UUIDs — just say "save this session as auth-refactor" and resume it later by name.

## Installation

Copy or symlink the `SKILL.md` file into your Claude Code skills directory:

```
~/.claude/skills/rename-session/SKILL.md
```

Claude Code will automatically discover and activate the skill.

## Usage

Use natural language inside any Claude Code session:

| What you say | What happens |
|---|---|
| "save this session as auth-refactor" | Saves current session with a friendly name |
| "list my sessions" | Shows all named sessions |
| "resume auth-refactor" | Prints the `claude --resume` command (and copies it to clipboard) |
| "rename auth-refactor to oauth-migration" | Renames a saved session |
| "update note on auth-refactor" | Changes the note on an existing session |
| "delete session auth-refactor" | Removes a saved session |

## Features

- **Auto-detect session ID** — Reads `~/.claude/history.jsonl` so you never need to find your session ID manually
- **Name validation** — Names must be lowercase alphanumeric + hyphens, 2-50 chars (like git branch names)
- **Fuzzy matching** — Typo in a session name? Get "did you mean..." suggestions
- **Duplicate detection** — Warns if a session ID is already saved under a different name
- **Clipboard integration** — Resume commands are auto-copied via `pbcopy` (macOS) or `xclip`/`xsel` (Linux)
- **Project path tracking** — Records which directory each session was started in
- **Atomic writes** — Uses temp file + `os.replace()` to prevent data corruption
- **Error handling** — Every command handles missing files, bad JSON, and missing sessions gracefully

## Data Storage

Sessions are stored in `~/.claude/session-names.json`:

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

## Name Rules

Session names must match: `^[a-z0-9][a-z0-9-]{0,48}[a-z0-9]$`

- Lowercase letters, digits, and hyphens only
- 2-50 characters
- No leading or trailing hyphens

Good: `auth-refactor`, `fix-cart-bug`, `v2-migration`
Bad: `-leading-hyphen`, `Has Spaces`, `../../path-traversal`

## Important: Resuming Sessions

You **cannot switch sessions from within a running session**. When you say "resume auth-refactor", the skill will print the `claude --resume <id>` command and copy it to your clipboard — but you still need to:

1. Exit your current Claude Code session (type `/exit` or press `Ctrl+C`)
2. Paste the command into your terminal to start the saved session

This is a limitation of Claude Code itself, not the skill.

## Requirements

- Python 3 (used by all scripts internally)
- macOS (`pbcopy`) or Linux (`xclip`/`xsel`) for clipboard support (optional)
