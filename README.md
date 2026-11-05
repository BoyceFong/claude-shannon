# claude-shannon

![License](https://img.shields.io/badge/license-MIT-blue) ![Platform](https://img.shields.io/badge/platform-macOS%20%7C%20Linux-lightgrey) ![Runtime](https://img.shields.io/badge/runtime-tmux%20%2B%20python%20stdlib-green)

A tmux workspace that turns [Claude Code](https://docs.anthropic.com/en/docs/claude-code) into a two-pane terminal UI — a session sidebar on the left, Claude Code on the right. No forks, no patches: official Claude Code, wrapped.

![claude-shannon workspace](docs/images/ui.png)

*Left: every conversation across all your projects, grouped by workspace, with live status dots. Right: Claude Code — the fleet view by default, or any conversation you click, loaded in place.*

Collapsed state — the sidebar shrinks to a slim rail; one click on `▶` brings it back:

![claude-shannon collapsed](docs/images/ui-collapsed.png)

## Why

Claude Code starts in one directory with one conversation, and your history is spread across every project you have ever touched. The built-in fleet view knows about all of it, but you only reach it after launching a session first.

`claude shannon` flips that: the cockpit comes first.

- **One command, the whole picture** — every session the fleet view knows about, across every project, sorted by activity. Set `SHOW_HISTORY=1` to also browse older local transcripts.
- **Click, don't context-switch** — picking a session loads it into the right pane in place. Your window never changes.
- **Claude stays official** — the right pane runs the unmodified `claude` binary. Upgrades, plugins, settings: all untouched.

## Quickstart

Requirements: **tmux 3.2+**, **python 3.8+** (stdlib only, no pip packages), macOS or Linux.

```sh
# 1. Install tmux
brew install tmux          # macOS
# apt install tmux         # Debian/Ubuntu

# 2. Clone anywhere
git clone git@github.com:BoyceFong/claude-shannon.git ~/.claude-shannon

# 3. Add the launcher to ~/.zshrc (adjust the path if you cloned elsewhere)
claude() {
  if [[ "$1" == "shannon" ]]; then
    command "$HOME/.claude-shannon/bin/claude-shannon"
    return
  fi
  command claude "$@"
}

# 4. Open a new terminal, then from any directory:
claude shannon
```

You should see the two-pane workspace: sessions on the left, Claude Code's fleet view on the right. Click any session — it loads in the right pane.

Every other `claude` usage passes through untouched.

**Contents** — [Why](#why) · [Quickstart](#quickstart) · [Features](#features) · [Configuration](#configuration) · [How it works](#how-it-works) · [Limitations & FAQ](#limitations--faq) · [Contributing](#contributing) · [License](#license)

## Features

- **Fleet-aligned session list** — the same sessions, groups, order and statuses as Claude Code's fleet view (the `~/.claude/jobs` registry), grouped by project path. Set `SHOW_HISTORY=1` to also list older local transcripts from `~/.claude/projects` (they are appended inside their project group).
- **Collapsible projects, fleet-style** — the project path row is selectable and hoverable; click it (or press Enter on it) to fold its sessions away. A `▸` marks a collapsed project.
- **Live status icons, aligned with Claude Code's fleet view** — working: animated spinner (coral); awaiting input: `✱` (yellow); done: `∙` (green); stopped/idle: `∙` (gray); failed: `✱` (red).
- **Hover highlights** — rows, `◀` `▶` `✕` buttons light up under the pointer, like the fleet view.
- **Follows Claude's own navigation** — press `←` back to the fleet view, or open another session inside Claude's view, and the sidebar re-binds itself to whatever is actually on screen. It reads the running `claude` process and the pane's screen every ~1.5s, so a stale mapping can never make you open the wrong conversation.
- **In-place switching that never interrupts** — every conversation you open keeps its own tmux pane. Clicking a row swaps it into the right pane; a running turn keeps running (and keeps printing) while it is out of view. Switch back and it is exactly where you left it. Works across projects, not just the current one.
- **Fleet view as home screen** — the right pane starts on Claude Code's own fleet view; everything you can do there, you can do here.
- **Collapse to a rail** — `M-s` or the `◀ 收起` button shrinks the sidebar to a 2-column rail with a `▶` expand button at the bottom. Nothing is ever destroyed.
- **New session in any project — or a brand-new one** — `＋ 新建` in the footer lists your known projects, plus a `✎ 输入新路径` entry: type any path (the directory is created if it doesn't exist) and a fresh Claude Code session opens there, in the right pane. The new session appears in the sidebar immediately.
- **Quit from the sidebar** — `✕ 退出` (click twice to confirm), or `Ctrl+Q`, closes the whole workspace (Claude Code included).
- **Pinned widths, no flicker** — the sidebar is a tmux `main-vertical` main pane: its width is held by tmux itself while you resize the window; only the right pane flexes. Crossing `COLLAPSE_AT` switches between expanded and rail.

## Keybindings

| Key | Action |
| --- | --- |
| `M-s` (Alt+S) | Collapse / expand the sidebar |
| `C-q` (Ctrl+Q) | Quit the workspace |
| `↑` `↓` / `j` `k` | Move the selection (gray highlight band; it fades after 5 seconds) |
| `Enter` | Open the selected session; on a project path, expand / collapse that project |
| Mouse | Click rows and buttons (hover highlights first — no focus click needed); click a project path to expand / collapse it; drag the divider to resize |

> macOS `⌘` chords never reach the terminal (the GUI intercepts them), so `⌘X`/`⌘Q` cannot be bound inside tmux — hence Ctrl+Q for quit. If you prefer a Cmd-style quit, map `⌘Q` → "send Ctrl+Q" in your terminal's keyboard settings.

> `M-s` and `C-q` are tmux server-level bindings, so shannon guards them: they act only while the active session is the shannon workspace; in every other tmux session the key is forwarded to the pane unchanged.
- **Auto-fit** — below `COLLAPSE_AT` columns the sidebar auto-collapses so Claude keeps a usable width; it restores itself when the window grows.
- **Width memory** — drag the divider to whatever width you like; it sticks across collapse/expand cycles.
- **Stable ordering** — loading a conversation never shuffles the list; order is frozen when the workspace opens.

## Configuration

`~/.claude/shannon/config` (created with defaults on first run):

| Key | Default | Meaning |
| --- | --- | --- |
| `SIDEBAR_W` | `35` | Fallback expanded sidebar width (columns) |
| `COLLAPSE_AT` | `70` | Window width below which the sidebar auto-collapses to the rail |
| `RAIL_W` | `2` | Collapsed rail width |
| `SHOW_HISTORY` | `0` | `1` also lists local transcripts that are not in the fleet registry; `0` is 100% fleet parity |

The expanded width you set by dragging the divider is remembered in `~/.claude/shannon/sidebar-w` (clamped to 12–40 columns) and survives restarts.

With a custom `SHANNON_SESSION=name`, all state lives in `~/.claude/shannon/sessions/<name>/` instead, so parallel workspaces never share pane ids, width memory or logs.

## How it works

```
╭ sidebar ──╮ ╭ host pane ═══════════════════════════╮
│ python3   │ │ sh/while loop, never exits          │
│ curses    │ │   └─ official `claude`              │
│ reads     │ │       fleet view  →  --resume <id>  │
│ ~/.claude │ │                                     │
╰═══════════╯ ╰═══════════════════════════════════════╯
      tmux session "shannon" (M-s toggles the rail)
```

- **Left pane** — a pure-stdlib curses app. It reads conversation metadata from `~/.claude/projects` (exactly where Claude Code stores transcripts) and live status from `claude agents --json`. Everything stays on your machine.
- **Right pane** — a ~45-line shell host that keeps the pane alive and runs the stock `claude` binary inside it. Every conversation you open gets its own pane in a hidden window; switching is a `tmux swap-pane`, so nothing is restarted and a running turn keeps running. No Claude Code internals are touched.
- **State** — `~/.claude/shannon/` holds the pane registry (`panes.json`), pane ids, your remembered width, the collapse mode, and `host.log` (why Claude last exited). Named sessions use `~/.claude/shannon/sessions/<name>/`.

## Limitations & FAQ

- **Is this a fork?** No. The right pane runs whatever `claude` is on your `PATH`. If Claude Code changes tomorrow, shannon keeps working.
- **Why don't I see an old conversation?** By default the sidebar mirrors the fleet view, which only knows sessions in Claude Code's job registry. Purely local transcripts are hidden — open them with `/resume` in the right pane, or set `SHOW_HISTORY=1` in the config to list them in the sidebar.
- **Does switching kill a running turn?** No. Every conversation you open gets its own pane and keeps running in a hidden window; switching is a tmux pane swap, not a restart. The sidebar keeps up to six background conversations; beyond that, the least recently viewed *idle* ones are closed. A conversation with a turn in progress is never touched.
- **Linux?** Should work — nothing is macOS-specific. Report an issue if something breaks.
- **Nested tmux?** Works, but the rail keybinding (`M-s`) may be captured by the outer session; run the toggle script directly or rebind. Shannon no longer changes global tmux options (escape-time, focus-events) and its `client-resized` hook is session-scoped.
- **Why "shannon"?** The workspace was born in a project folder of that name. It also fits: Shannon gave us the measure of information — this is a pane that shows all of it.

## Contributing

Issues and PRs welcome. The whole app is six small scripts under `bin/` — `claude-shannon` (launcher), `shannon-workspace` (layout), `shannon-host` (right-pane host), `shannon-sidebar` (the curses UI and pane pool), `shannon-toggle` / `shannon-autofit` (collapse + resize hooks). Please keep the zero-dependency property: python stdlib and POSIX shell only.

Bug reports and feature requests go to [GitHub Issues](https://github.com/BoyceFong/claude-shannon/issues).

## Support

Questions and usage help: [GitHub Issues](https://github.com/BoyceFong/claude-shannon/issues) — mark non-bug questions with the `question` label.

## Security

All data read by this tool (conversation transcripts, live status) stays on your machine; nothing is sent anywhere. If you believe you found a vulnerability, open a GitHub issue marked **security** rather than emailing.

## License

[MIT](LICENSE)
