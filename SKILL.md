---
name: cmux
description: |
  Control and integrate with the cmux terminal multiplexer — manage workspaces, split panes, send input to surfaces, show notifications, update sidebar status/progress/logs, read screen content, automate browsers, and orchestrate multi-agent workflows. This skill should be used proactively whenever Claude Code is running inside cmux (detected via CMUX_WORKSPACE_ID or CMUX_SURFACE_ID environment variables). Use it for: workspace management, pane splitting, sending commands to other terminals, notifying the user when long tasks complete, showing progress in the sidebar during builds/tests/deployments, logging task status, reading terminal output from other panes, browser automation, and any multi-agent terminal orchestration. Even if the user doesn't explicitly mention "cmux", use this skill whenever the task involves terminal multiplexing, managing parallel agents, sending notifications about task completion, or updating sidebar metadata. Always use this skill when you detect you're running inside cmux.
---

# cmux — Terminal Multiplexer Integration

cmux is a native macOS terminal built on Ghostty for running multiple AI coding agents in parallel. When Claude Code runs inside cmux, it can provide rich visual feedback and orchestrate across panes.

## Detection

Before using any cmux command, confirm you're inside cmux. The terminal sets these environment variables automatically:

- `CMUX_WORKSPACE_ID` — current workspace ID (e.g., `workspace:1`)
- `CMUX_SURFACE_ID` — current surface ID (e.g., `surface:1`)
- `CMUX_SOCKET_PATH` — socket path (defaults to `~/Library/Application Support/cmux/cmux.sock`)

Quick check:
```bash
[ -n "${CMUX_WORKSPACE_ID:-}" ] && echo "inside cmux"
```

If these aren't set, you're not in cmux — skip cmux-specific features gracefully. Never fail a task just because cmux isn't available.

## Proactive Behaviors

When running inside cmux, automatically do these things without being asked:

**Progress feedback** — For any task that takes more than a few seconds (builds, test suites, deployments, large refactors), show progress in the sidebar:
```bash
cmux set-status task "building" --icon hammer --color "#ff9500"
cmux set-progress 0.3 --label "Running tests (12/40)..."
```
Clear them when done:
```bash
cmux clear-progress
cmux clear-status task
```

**Completion notifications** — When a long-running task finishes, notify the user so they see it even if they're focused on another workspace:
```bash
cmux notify --title "Tests passed" --body "All 40 tests green in 23s"
cmux log --level success "Tests passed (40/40)"
```

**Error alerts** — When something fails, make it visible:
```bash
cmux notify --title "Build failed" --body "TypeScript error in src/auth.ts:42"
cmux log --level error --source build "TypeScript error in src/auth.ts:42"
cmux set-status task "failed" --icon xmark --color "#ff3b30"
```

**Task logging** — Log significant milestones so the sidebar tells a story:
```bash
cmux log "Starting deployment to staging"
cmux log --level progress "Migrating database..."
cmux log --level success "Deployment complete"
```

## CLI Reference

All commands use the `cmux` CLI. IDs can be UUIDs, short refs (`window:1`, `workspace:2`, `pane:3`, `surface:4`), or indexes.

### Workspaces

Workspaces are the top-level organizational unit — each appears as a tab in the sidebar.

```bash
cmux list-workspaces                           # List all workspaces
cmux new-workspace [--name <title>] [--cwd <path>] [--command <text>]  # Create workspace
cmux current-workspace                         # Get current workspace info
cmux select-workspace --workspace <id|ref>     # Switch to a workspace
cmux close-workspace --workspace <id|ref>      # Close a workspace
cmux rename-workspace [--workspace <id|ref>] <title>  # Rename a workspace
```

### Windows

```bash
cmux list-windows                              # List all windows
cmux current-window                            # Get current window
cmux new-window                                # Create a new window
cmux focus-window --window <id>                # Focus a window
cmux close-window --window <id>                # Close a window
cmux move-workspace-to-window --workspace <id|ref> --window <id|ref>
```

### Panes and Surfaces

Panes contain surfaces. A pane can hold multiple surfaces (tabs). Split the current pane in any direction.

```bash
cmux new-split <left|right|up|down> [--workspace <id|ref>] [--surface <id|ref>]
cmux new-pane [--type <terminal|browser>] [--direction <left|right|up|down>] [--workspace <id|ref>] [--url <url>]
cmux new-surface [--type <terminal|browser>] [--pane <id|ref>] [--workspace <id|ref>] [--url <url>]
cmux list-panes [--workspace <id|ref>]         # List panes
cmux list-pane-surfaces [--workspace <id|ref>] [--pane <id|ref>]  # List surfaces in a pane
cmux tree [--all] [--workspace <id|ref>]       # Show full hierarchy
cmux focus-pane --pane <id|ref> [--workspace <id|ref>]
cmux close-surface [--surface <id|ref>] [--workspace <id|ref>]
cmux resize-pane --pane <id|ref> [--workspace <id|ref>] (-L|-R|-U|-D) [--amount <n>]
```

### Sending Input

Send text or keypresses to any surface. This is the key to multi-agent orchestration — you can type commands into other panes programmatically. Use `--surface` to target a specific pane.

```bash
cmux send "echo hello"                         # Send text to current surface
cmux send --surface <id|ref> "echo hello"      # Send text to specific surface
cmux send-key enter                            # Send keypress to current surface
cmux send-key --surface <id|ref> enter         # Send keypress to specific surface
cmux send-panel --panel <id|ref> "text"        # Send to a panel
cmux send-key-panel --panel <id|ref> <key>     # Send key to a panel
```

Available keys: `enter`, `tab`, `escape`, `backspace`, `delete`, `up`, `down`, `left`, `right`

When sending commands to execute, the text is sent as typed input — follow with `cmux send-key enter` if you need to press Enter, or include `\n` in the text.

### Reading Screen Content

Read what's currently displayed in any terminal pane — useful for monitoring agent output.

```bash
cmux read-screen                               # Read current surface
cmux read-screen --surface <id|ref>            # Read specific surface
cmux read-screen --scrollback                  # Include scrollback buffer
cmux read-screen --lines 50                    # Last N lines
```

### Notifications

Notifications appear as rings around panes and badges in the sidebar — they're how you get the user's attention across workspaces.

```bash
cmux notify --title "Title" --body "Body"
cmux notify --title "Title" --subtitle "Sub" --body "Body"
cmux notify --title "Title" --body "Body" --workspace <id|ref> --surface <id|ref>
cmux list-notifications
cmux clear-notifications
```

### Sidebar Metadata

The sidebar shows rich status information per workspace. Use these to give the user at-a-glance visibility into what you're doing.

**Status pills** — Small labeled badges. Use a unique key per tool/concern so they don't collide:
```bash
cmux set-status build "compiling" --icon hammer --color "#ff9500"
cmux set-status tests "running" --icon checkmark --color "#007aff"
cmux clear-status build
cmux list-status
```

**Progress bar** — A single progress bar (0.0 to 1.0) with an optional label:
```bash
cmux set-progress 0.5 --label "Building... (50%)"
cmux clear-progress
```

**Log entries** — A scrollable log in the sidebar. Levels: info, progress, success, warning, error:
```bash
cmux log "Starting task"
cmux log --level progress --source build "Compiling modules..."
cmux log --level success "All done"
cmux log --level error --source tests "3 failures"
cmux clear-log
cmux list-log --limit 10
```

**Full sidebar dump:**
```bash
cmux sidebar-state [--workspace <id|ref>]
```

### Browser Automation

cmux has an embedded browser with a full automation API. Create browser panes and control them programmatically.

```bash
cmux browser open [url]                        # Open browser in new split
cmux browser open-split [url]                  # Explicit split
cmux browser goto <url> [--snapshot-after]     # Navigate
cmux browser back|forward|reload [--snapshot-after]
cmux browser url                               # Get current URL
cmux browser snapshot [--interactive] [--compact]  # Get DOM snapshot
cmux browser eval <script>                     # Execute JavaScript
cmux browser click <selector> [--snapshot-after]
cmux browser type <selector> <text> [--snapshot-after]
cmux browser fill <selector> [text] [--snapshot-after]
cmux browser press <key> [--snapshot-after]
cmux browser screenshot [--out <path>] [--json]
cmux browser wait [--selector <css>] [--text <text>] [--url-contains <text>] [--timeout-ms <ms>]
cmux browser get <url|title|text|html|value|attr|count|box|styles> [...]
cmux browser is <visible|enabled|checked> <selector>
cmux browser console <list|clear>
cmux browser errors <list|clear>
cmux browser tab <new|list|switch|close|<index>> [...]
```

### Utility

```bash
cmux ping                                      # Check if cmux is responsive
cmux capabilities                              # List available methods
cmux identify [--workspace <id|ref>] [--surface <id|ref>]  # Show context
cmux version                                   # Show version
cmux tree --all                                # Full hierarchy of all windows/workspaces/panes/surfaces
cmux find-window [--content] [--select] <query>  # Search across windows
```

### Hooks

Set up event-driven automation:
```bash
cmux set-hook <event> <command>                # Register a hook
cmux set-hook --list                           # List hooks
cmux set-hook --unset <event>                  # Remove a hook
```

### Clipboard / Buffers

```bash
cmux set-buffer [--name <name>] <text>
cmux list-buffers
cmux paste-buffer [--name <name>] [--workspace <id|ref>] [--surface <id|ref>]
```

### tmux Compatibility

cmux provides tmux-compatible commands for scripts that expect tmux:
```bash
cmux capture-pane [--scrollback] [--lines <n>]  # Alias for read-screen
cmux swap-pane --pane <id|ref> --target-pane <id|ref>
cmux break-pane [--workspace <id|ref>] [--pane <id|ref>]
cmux join-pane --target-pane <id|ref>
cmux next-window | previous-window | last-window
cmux last-pane
cmux respawn-pane [--command <cmd>]
```

### Global Flags

These work on most commands:

| Flag | Purpose |
|------|---------|
| `--workspace <id\|ref>` | Target a specific workspace |
| `--surface <id\|ref>` | Target a specific surface |
| `--pane <id\|ref>` | Target a specific pane |
| `--window <id\|ref>` | Target a specific window |
| `--id-format refs\|uuids\|both` | Control identifier format in output |

### Environment Variables

| Variable | Description |
|----------|-------------|
| `CMUX_WORKSPACE_ID` | Auto-set: current workspace ID, used as default `--workspace` |
| `CMUX_SURFACE_ID` | Auto-set: current surface ID, used as default `--surface` |
| `CMUX_TAB_ID` | Optional: used by tab-action/rename-tab as default `--tab` |
| `CMUX_SOCKET_PATH` | Override socket path (default: `~/Library/Application Support/cmux/cmux.sock`) |
| `CMUX_SOCKET_PASSWORD` | Socket auth password (if required) |

## Intelligent Splitting

Don't blindly split in the same direction every time — that creates unusably narrow or short panes. Before splitting, check the current layout with `cmux tree` and choose the direction that makes sense.

**Rules of thumb:**

1. **Check the layout first.** Run `cmux tree` to see what panes already exist. If there's already a right split, split down (or vice versa). Avoid splitting a pane that's already small.

2. **Alternate directions.** If the workspace has one pane, split right (side-by-side is the most natural starting point). For a third pane, split one of the existing panes down to create an L-shape. For four panes, aim for a 2x2 grid.

3. **Match the purpose to the position:**
   - **Dev servers / long-running processes** → right split (tall, narrow is fine for scrolling output)
   - **Logs / test output** → bottom split (wide, short works well for log lines)
   - **Secondary agent / editor** → right split (needs width for code)
   - **Quick commands / monitoring** → bottom split

4. **Don't over-split.** Three panes in one workspace is usually the practical maximum before things get cramped. If you need a fourth agent, consider a new workspace instead:
   ```bash
   cmux new-workspace --name "agent-4" --cwd /path/to/project
   ```

5. **Example: setting up a dev environment:**
   ```bash
   # Start with one pane (your coding session)
   # Add dev server to the right
   cmux new-split right
   cmux send --surface surface:2 "npm run dev"
   cmux send-key --surface surface:2 enter

   # Add test watcher below the dev server
   cmux new-split down --surface surface:2
   cmux send --surface surface:3 "npm test -- --watch"
   cmux send-key --surface surface:3 enter

   # Result: code on left, dev server top-right, tests bottom-right
   ```

## Multi-Agent Orchestration

One of cmux's most powerful uses is coordinating multiple AI agents. Here's the pattern:

1. **Create panes** for each agent:
   ```bash
   cmux new-split right
   cmux tree  # see the full hierarchy with surface IDs
   ```

2. **Launch agents** in each pane:
   ```bash
   cmux send --surface surface:2 "claude --task 'implement auth module'"
   cmux send-key --surface surface:2 enter
   ```

3. **Monitor progress** by reading their screens:
   ```bash
   cmux read-screen --surface surface:2 --lines 20
   ```

4. **Update sidebar** with per-agent status:
   ```bash
   cmux set-status auth "running" --icon hammer --color "#007aff"
   cmux set-status db "done" --icon checkmark --color "#34c759"
   ```

5. **Notify when done**:
   ```bash
   cmux notify --title "All agents complete" --body "Auth, DB, and API modules ready"
   ```

## Style Guide

- Keep status pill values short (1-3 words) — they appear in a small badge
- Use clear, specific notification titles — the user may have many workspaces
- Choose log levels meaningfully: `info` for milestones, `progress` for ongoing work, `success`/`error` for outcomes, `warning` for things that need attention but aren't failures
- Clean up after yourself: clear progress bars and status pills when tasks finish
- Use `--source` on log entries when multiple subsystems are active so the user can tell them apart
