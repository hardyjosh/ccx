# ccx — Claude Code session manager for OpenClaw

`ccx` is a skill for [OpenClaw](https://openclaw.ai) that lets it spawn, drive, and tear down [Claude Code](https://docs.claude.com/en/docs/claude-code) child sessions on its own. Each child runs in `tmux` with `--dangerously-skip-permissions` and `--remote-control`, so it shows up in the OpenClaw UI alongside your main session immediately — no inside-pane keystrokes, no manual setup.

It ships as two pieces:

- A bash CLI (`cc-mgr`) that handles the tmux + claude wiring.
- An OpenClaw skill (`/ccx`) so OpenClaw can invoke it as `/ccx new <name>`, `/ccx list`, `/ccx resume`, etc.

## What it gives you

- **Spawn child agents.** OpenClaw can fork off named Claude Code workers to handle long-running jobs in parallel, then check on them later via `/ccx list` / `/ccx status`.
- **Auto-onboarding.** New sessions boot with a default first prompt that asks them to read your `AGENTS.md` / `CLAUDE.md` and reply `ready` — so they share the same context OpenClaw has. Override or skip per-call.
- **Named & attachable.** Each child is a named tmux session you can `tmux attach -t <name>` from a terminal, or jump to from the OpenClaw UI.
- **Resumable.** Pick up any past Claude Code session by UUID via `/ccx resume`.

> **Not using OpenClaw?** `cc-mgr` works as a standalone CLI on any machine, and the skill folder is structured as a [Claude Code skill](https://docs.claude.com/en/docs/claude-code) (`SKILL.md` with frontmatter) — so you can drop it into `~/.claude/skills/` and use `/ccx` from a Claude Code session directly. Spawned children show up in [claude.ai/code](https://claude.ai/code) or any other Remote Control client. But the orchestration loop — a platform driving Claude Code workers — is what OpenClaw is built for, and it's where `ccx` earns its keep.

## Requirements

- `bash`, `tmux`
- [Claude Code](https://docs.claude.com/en/docs/claude-code) installed and on PATH (or set `CCX_CLAUDE_BIN` to its absolute path)
- `python3` for `list-sessions` previews (optional — falls back to no preview)

## Install

```sh
git clone https://github.com/hardyjosh/ccx ~/.local/share/ccx
ln -sf ~/.local/share/ccx/bin/cc-mgr ~/.local/bin/cc-mgr
```

Make sure `~/.local/bin` is on your `PATH`.

### Optional: install the Claude Code skill

If you want to invoke `ccx` as `/ccx ...` from inside a Claude Code session:

```sh
mkdir -p ~/.claude/skills
ln -sf ~/.local/share/ccx/skill ~/.claude/skills/ccx
```

Restart your Claude Code session and `/ccx` should appear in `/help`.

## Usage

### Spawn a new named session

```sh
cc-mgr new oracle
```

By default the new session boots with an onboarding prompt that asks it to read `AGENTS.md` / `CLAUDE.md` and follow any onboarding instructions, then reply `ready`. Override with `--prompt`, or skip with `--no-prompt`.

### Spawn with a custom first prompt

```sh
cc-mgr new mybug --prompt "Reproduce and fix the off-by-one in src/foo.ts:42"
```

The prompt is passed to `claude` as a positional argument and auto-submitted on boot.

### Resume a previous session

```sh
cc-mgr list-sessions                   # 10 most-recent for $PWD
cc-mgr resume oracle --uuid <uuid>     # uuid from list-sessions
```

### List active tmux sessions

```sh
cc-mgr list                            # all tmux sessions, last 8 lines each
cc-mgr status oracle                   # last 30 lines of one
```

### Attach / detach / close

```sh
tmux attach -t oracle                  # attach (Ctrl-b d to detach)
cc-mgr close oracle                    # kill the tmux session
```

### From inside Claude Code (with the skill installed)

```
/ccx new oracle
/ccx new mybug --prompt "fix the off-by-one"
/ccx list
/ccx resume oracle
/ccx close oracle
```

## Configuration

Override defaults with environment variables:

| Variable | Default | Purpose |
|---|---|---|
| `CCX_CLAUDE_BIN` | first `claude` on PATH | path to the claude executable |
| `CCX_DEFAULT_CWD` | `$PWD` | default working directory for new sessions |
| `CCX_PROJECTS_DIR` | `$HOME/.claude/projects` | where Claude Code stores per-project session files |
| `CCX_DEFAULT_PROMPT` | onboarding prompt asking the agent to read AGENTS.md / CLAUDE.md | the prompt sent on `new` when neither `--prompt` nor `--no-prompt` is given |

## How resume finds sessions

Claude Code stores resumable session jsonl files at `$CCX_PROJECTS_DIR/<encoded-cwd>/`, where `<encoded-cwd>` is the cwd path with `/` and `.` replaced by `-`. `cc-mgr list-sessions [--cwd <dir>]` lists the most recent ten, with timestamp and a one-line preview of the first user message.

## Caveats

- `--dangerously-skip-permissions` bypasses Claude Code permission prompts. Only use this in environments you trust. The same caveat applies to every session `ccx` spawns.
- Sessions spawned by `ccx` inherit your current shell environment.
- Session names must be valid tmux session names (no `:` or `.`).
- The `claude` CLI is assumed to support the positional-prompt argument and `--remote-control` flag (Claude Code v2.x).

## License

MIT — see [LICENSE](./LICENSE).
