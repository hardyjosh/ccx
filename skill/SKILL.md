---
name: ccx
description: Manage long-lived Claude Code sessions in tmux with --dangerously-skip-permissions and --remote-control. Use when the user sends `/ccx new <name>`, `/ccx resume`, `/ccx list`, `/ccx close <name>`, or `/ccx status <name>`.
allowed-tools: Bash, Read
---

# ccx — Claude Code session manager for OpenClaw

Spawn, resume, list, and close named Claude Code child sessions running inside tmux. Each session is started with `--dangerously-skip-permissions` and `--remote-control <name>` so it shows up in the [OpenClaw](https://openclaw.ai) UI (or any other Remote Control client) without any inside-pane keystrokes.

This skill runs at the OpenClaw layer — the underlying LLM driving OpenClaw is irrelevant; `cc-mgr` always spawns Claude Code workers because that's what supports `--remote-control`.

## Critical rules

**Follow the steps literally and in order. If anything fails or is ambiguous: STOP, report exactly what went wrong (command, exit code, output), and wait for guidance. Never improvise. Never auto-pick a session to resume.**

The actual work is done by the `cc-mgr` script (must be on PATH). This skill is a thin wrapper that dispatches to it.

## Dispatch on subcommand

`/ccx` arrives with the subcommand and args following. Trim whitespace, then:

- `new <name> [...]`     → run **NEW**
- `resume [name] [...]`  → run **RESUME**
- `list`                 → run **LIST**
- `list-sessions [...]`  → run **LIST-SESSIONS**
- `status <name>`        → run **STATUS**
- `close <name>`         → run **CLOSE**
- anything else / empty  → reply with the usage block from `cc-mgr help` and stop.

Do NOT guess. If the subcommand is missing or malformed, ask the user what they meant.

---

## NEW

`/ccx new <name> [--cwd <dir>] [--rc <rc-name>] [--prompt <text>] [--no-prompt]`

Spawns a fresh Claude Code session in a new tmux session.

1. Run:
   ```sh
   cc-mgr new <name> [--cwd <dir>] [--rc <rc-name>] [--prompt <text>] [--no-prompt]
   ```
2. If the script exits non-zero, STOP and report stdout+stderr verbatim.
3. On success, reply with the script's `OK:` line plus the attach/close hints.

Defaults baked into the script: `cwd=$PWD` (override with `CCX_DEFAULT_CWD`), `rc-name=<name>`.

**Initial prompt behavior:**
- If `--prompt <text>` is supplied, it's passed to `claude` as a positional argument and auto-submitted as the first user message — the session starts working on it immediately and the input box is left empty for follow-ups. Quote multi-word prompts. Multiline prompts are fine.
- If neither `--prompt` nor `--no-prompt` is supplied, the script auto-submits a default onboarding prompt (override with `CCX_DEFAULT_PROMPT`) that tells the new agent to read AGENTS.md / CLAUDE.md and follow any onboarding instructions, then reply "ready".
- If `--no-prompt` is supplied, the session boots silently with no first message.

Free-text shorthand: if the user writes `/ccx new <name> <free text…>` with no `--` flags, treat everything after `<name>` as the prompt and pass it via `--prompt`. If any `--` flag is present, parse strictly — don't auto-collect leftover words into the prompt.

---

## RESUME

`/ccx resume <name> [--cwd <dir>] [--rc <rc-name>] [--uuid <uuid>]`

Resume a previous Claude session inside a new tmux session.

If `--uuid` is supplied, run directly:
```sh
cc-mgr resume <name> --uuid <uuid> [--cwd <dir>] [--rc <rc-name>]
```

If `--uuid` is **not** supplied:

1. Resolve the cwd (default `$PWD` unless `--cwd` was passed).
2. List candidate sessions:
   ```sh
   cc-mgr list-sessions --cwd <cwd>
   ```
3. Show the user the numbered list and ask: "Which session UUID do you want to resume?"
4. **Wait for the user.** Do not pick. Do not auto-pick if there's only one. Accept either a UUID or a number; if number, re-run `list-sessions` to be safe and resolve the row.
5. Run the resume command with the resolved UUID.
6. If the script exits non-zero, STOP and report stdout+stderr.

---

## LIST

`/ccx list`

Show every tmux session and the last 8 lines of each pane:
```sh
cc-mgr list
```
Forward the output verbatim. Do not truncate.

---

## LIST-SESSIONS

`/ccx list-sessions [--cwd <dir>]`

Show the 10 most-recent resumable Claude sessions for `<cwd>`:
```sh
cc-mgr list-sessions [--cwd <dir>]
```

---

## STATUS

`/ccx status <name>`

Last 30 lines of the named tmux pane:
```sh
cc-mgr status <name>
```

---

## CLOSE

`/ccx close <name>`

```sh
cc-mgr close <name>
```

Forward the script's output. The script verifies the session is gone and errors loudly if not.

---

## What NOT to do

- Do **not** run `claude` directly outside the script — go through `cc-mgr` so flags stay consistent.
- Do **not** auto-pick a session UUID for resume.
- Do **not** type `/remote-control` inside a pane — the script passes `--remote-control` on the CLI, so it is already on at boot.
- Do **not** retry on failure. Report and wait.
