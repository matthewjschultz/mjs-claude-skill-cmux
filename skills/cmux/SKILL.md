---
name: cmux
description: |
  Control and integrate with the cmux terminal multiplexer — manage workspaces, split panes, send input to surfaces, show notifications, update sidebar status/progress/logs, read screen content, automate browsers, and orchestrate multi-agent workflows. This skill should be used proactively whenever Claude Code is running inside cmux (detected via CMUX_WORKSPACE_ID or CMUX_SURFACE_ID environment variables). Use it for: workspace management, pane splitting, sending commands to other terminals, notifying the user when long tasks complete, showing progress in the sidebar during builds/tests/deployments, logging task status, reading terminal output from other panes, browser automation, and any multi-agent terminal orchestration. Even if the user doesn't explicitly mention "cmux", use this skill whenever the task involves terminal multiplexing, managing parallel agents, sending notifications about task completion, or updating sidebar metadata. Always use this skill when you detect you're running inside cmux.
---

# cmux — Terminal Multiplexer Integration

cmux is a native macOS terminal built on Ghostty for running multiple AI coding agents in parallel. When Claude Code runs inside cmux, it can provide rich visual feedback and orchestrate across panes.

This file is the **pattern guide** — how to think about cmux and the commands you'll use most. For exhaustive flags and lesser-used subcommands, read `api-reference.md` in this skill directory.

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

## Quick Reference

The happy path for the most common operations. Use these verbatim before reaching for anything fancier.

| Task | Command |
|------|---------|
| See workspace layout | `cmux tree` |
| Open terminal split (right) | `cmux new-split right` |
| Open browser split (right) | `cmux new-pane --type browser --direction right [--url <url>]` |
| Open/reuse a browser surface | `cmux browser open [url]` |
| Send command to a surface | `cmux send --surface <id> "cmd"` then `cmux send-key --surface <id> enter` |
| Read surface output | `cmux read-screen --surface <id> --lines 50` |
| Navigate browser (specific surface) | `cmux browser --surface <id> goto <url>` |
| Inspect browser page | `cmux browser --surface <id> snapshot` |
| Show notification | `cmux notify --title "Title" --body "Body"` |
| Sidebar status pill | `cmux set-status <key> "value" --icon <name> --color "#hex"` |
| Sidebar progress bar | `cmux set-progress 0.5 --label "Working..."` |
| Sidebar log entry | `cmux log --level success "Done"` |

IDs accepted everywhere: UUIDs, short refs (`window:1`, `workspace:2`, `pane:3`, `surface:4`), or indexes.

## Common Mistakes — Never Do This

These commands **do not exist** or are **broken**. Using them is the most common cmux failure mode. The wrong-looking ones are plausible guesses that happen to be wrong — always prefer the "Use instead" column:

| Don't use | Use instead | Why |
|-----------|-------------|-----|
| `cmux send-surface <id> "text"` | `cmux send --surface <id> "text"` | `send-surface` was never a real command |
| `cmux send-key-surface <id> enter` | `cmux send-key --surface <id> enter` | Same — `--surface` is a flag, not a suffix |
| `cmux list-surfaces` / `--json` | `cmux tree` | No `list-surfaces` command exists |
| `cmux new-split right --browser` | `cmux new-pane --type browser --direction right` | `new-split` has no `--browser` flag and always makes terminals |
| `cmux browser navigate <url>` | `cmux browser goto <url>` | Subcommand is `goto`, not `navigate` |
| `cmux browser snapshot --surface <id>` | `cmux browser --surface <id> snapshot` | `--surface` must come **before** the browser subcommand |
| `cmux browser screenshot --out <path>` | `cmux browser --surface <id> screenshot` | No `--out`; screenshots return base64 (use `--json` for structured output) |
| `cmux browser eval "..."` | `get text` / `get html` / `snapshot` / `console list` | **Broken upstream** (manaflow-ai/cmux#2610) |
| `cmux browser press <key>` | `click` / `type` / `fill`, or drive the API with `curl` | **Broken upstream** (manaflow-ai/cmux#2610) |
| `cmux browser addscript <script>` | No workaround | **Broken upstream** (manaflow-ai/cmux#2610) |
| `http://localhost:...` in the browser | `http://127.0.0.1:...` | Embedded browser can't resolve `localhost` |

If something in this table seems too cautious, sanity-check against the live CLI (`cmux <subcommand> --help`) before assuming the docs are wrong. The patterns here came from real failures.

## Opening a Browser

There are two ways to get a browser surface, and they solve different problems. Pick by intent:

1. **`cmux new-pane --type browser --direction <dir> [--url <url>]`** — use when you care about *placement*. Creates a new pane in the direction you specify and puts a browser surface inside it. This is the right choice for "open a browser to the right" or "put a browser below this pane".

2. **`cmux browser open [url]`** — use when you just want *a browser surface somewhere*. Reuses an existing browser surface in the workspace when one exists (prints `placement=reuse`) and creates one if needed. It prints the surface id on success, which you can capture and pass to subsequent `cmux browser --surface <id> ...` commands:
   ```bash
   cmux browser open http://127.0.0.1:3000
   # OK surface=surface:10 pane=pane:5 placement=reuse
   ```

**Never use `cmux new-split` for browsers.** `new-split` always creates terminals — there is no `--browser` flag. This is the single most common cmux mistake.

## Proactive Behaviors

The sidebar is cmux's superpower. Use it — but only when it genuinely helps the user. Noise in the sidebar is worse than silence, because users tune it out.

**When to proactively use the sidebar:**

- **Explicitly long-running work**: full builds, full test suites, deploys, large migrations, long linting runs, multi-minute refactors. If the task is likely to take ~30 seconds or more, sidebar feedback is valuable.
- **Work the user said they'd walk away from**: "I'm switching to another workspace, let me know when it's done" → notifications and progress are obviously wanted.
- **Multi-agent orchestration**: when you're coordinating work across multiple panes, per-agent status pills tell the user at a glance which agents are running, done, or failed.

**When NOT to use the sidebar:**

- Short tool calls (reads, small edits, single commands). Don't wrap `npm install` of a handful of packages in progress bars.
- Anything the user can see happening in front of them in the same pane.
- Cosmetic touches they didn't ask for. In particular, **don't set a workspace color on session start unless the user asked for one** — if they wanted one they'd have set it. You may *suggest* it, but don't do it silently.

### The core feedback primitives

**Progress + status for long work:**
```bash
cmux set-status task "building" --icon hammer --color "#ff9500"
cmux set-progress 0.3 --label "Running tests (12/40)..."
# ... when done
cmux clear-progress
cmux clear-status task
```

**Completion notifications** when the user is likely elsewhere:
```bash
cmux notify --title "Tests passed" --body "All 40 tests green in 23s"
cmux log --level success "Tests passed (40/40)"
```

**Error alerts** with clear labels:
```bash
cmux notify --title "Build failed" --body "TypeScript error in src/auth.ts:42"
cmux log --level error --source build "TypeScript error in src/auth.ts:42"
cmux set-status task "failed" --icon xmark --color "#ff3b30"
```

**Milestone logging** when orchestrating a multi-step task — the sidebar log becomes a timeline of what happened:
```bash
cmux log "Starting deployment to staging"
cmux log --level progress "Migrating database..."
cmux log --level success "Deployment complete"
```

Always clean up (`clear-progress`, `clear-status <key>`) when the task finishes, so the sidebar reflects current state rather than accumulated debris.

## Core Commands

The patterns below cover ~90% of real cmux use. For exhaustive subcommand listings, see `api-reference.md`.

### Discovering the layout

`cmux tree` is your primary discovery tool. It prints the full hierarchy with surface IDs, types, and titles — use it before sending to a surface you haven't addressed yet, and any time the layout may have changed.

```bash
cmux tree
# pane pane:5
#   ├── surface surface:1 [browser] "MDLZ" http://127.0.0.1:3000/
#   └── surface surface:10 [browser] "MDLZ" [selected] http://127.0.0.1:3000/
```

There is no `list-surfaces` command — `cmux tree` is how you enumerate them.

### Splitting panes

```bash
cmux new-split <left|right|up|down> [--surface <id|ref>]    # terminal
cmux new-pane --type browser --direction <dir> [--url <url>] # browser
```

Don't blindly split in the same direction every time — that creates unusably narrow or short panes. Before splitting, check the current layout with `cmux tree` and choose a direction that makes sense. See "Intelligent Splitting" below.

### Sending input to other surfaces

This is the key to multi-agent orchestration. Use `--surface` to target a specific pane:

```bash
cmux send --surface surface:2 "npm run dev"
cmux send-key --surface surface:2 enter
```

Text is sent as typed input. Follow with `send-key enter` if you need to press Enter, or include `\n` in the text. Available keys: `enter`, `tab`, `escape`, `backspace`, `delete`, `up`, `down`, `left`, `right`.

**Not `send-surface` or `send-key-surface`** — those aren't real commands. `--surface` is a flag.

### Reading what's on another screen

```bash
cmux read-screen --surface surface:2 --lines 50       # last 50 lines
cmux read-screen --surface surface:2 --scrollback     # include scrollback
```

Useful for monitoring another agent's output, diagnosing a failure in another pane, or confirming a command completed.

### Notifications and sidebar

```bash
cmux notify --title "Title" --body "Body" [--subtitle "..."] [--workspace <id>] [--surface <id>]
cmux set-status <key> "<value>" --icon <name> --color "#hex"
cmux clear-status <key>
cmux set-progress <0.0-1.0> --label "..."
cmux clear-progress
cmux log [--level info|progress|success|warning|error] [--source <name>] "message"
```

Use a **unique key per concern** for status pills (`build`, `tests`, `deploy`) so they don't collide. Progress is a single global bar — don't interleave progress bars from different tasks.

## Browser Automation

cmux has an embedded browser with an automation API. Three rules that catch most mistakes:

1. **`--surface` comes BEFORE the subcommand.** `cmux browser --surface surface:10 snapshot` works. `cmux browser snapshot --surface surface:10` fails with `browser requires a subcommand`.

2. **Use `127.0.0.1`, not `localhost`.** The embedded browser cannot resolve `localhost` and shows "This page couldn't load." Substitute `127.0.0.1` in every URL you pass to `open`, `goto`, etc.:
   ```bash
   # BROKEN
   cmux browser open http://localhost:3000
   # WORKS
   cmux browser open http://127.0.0.1:3000
   ```

3. **`eval`, `press`, and `addscript` are currently broken upstream (manaflow-ai/cmux#2610).** Every invocation — even `cmux browser eval "1"` — returns `Error: js_error: A JavaScript exception occurred`. Read-only commands (`get`, `click`, `snapshot`, `console list`, `errors list`) work fine on the same surface. Until the bug is fixed, use these substitutes:
   - Instead of `eval` → `get text`, `get html`, `get title`, `snapshot`, `console list`, `errors list`
   - Instead of `press` → `click`, `type`, `fill`, or drive the underlying API with `curl`
   - Instead of `addscript` → no workaround; wait for the fix

### Inspection primitives (the ones that work)

```bash
cmux browser --surface <id> snapshot [--interactive]   # accessibility tree
cmux browser --surface <id> get title
cmux browser --surface <id> get url
cmux browser --surface <id> get text "body"            # visible text of an element
cmux browser --surface <id> get html "#root"           # inner HTML of an element
cmux browser --surface <id> get value "input[name=email]"
cmux browser --surface <id> get attr "a.primary" "href"
cmux browser --surface <id> get count "button"
cmux browser --surface <id> console list               # captured console messages
cmux browser --surface <id> errors list                # captured page errors
```

`snapshot` returns a structured accessibility tree — the fastest way to understand page state. `--interactive` assigns refs to clickable elements, but see the next section: those refs are fragile.

### Interacting with pages — two caveats

1. **CSS selectors are more reliable than ref selectors.** Refs from `snapshot --interactive` expire between snapshots and frequently fail with "Element not found". Prefer CSS:
   ```bash
   # Reliable
   cmux browser --surface surface:10 click "button:nth-child(5)" --snapshot-after
   # Often fails
   cmux browser --surface surface:10 click '[ref=e12]'
   ```

2. **Synthetic clicks may not trigger React state changes.** `cmux browser click` (and `element.click()` / dispatched MouseEvents) often do not propagate through React's synthetic event system, so tab switches and other React-controlled interactions can silently do nothing. For inspection this is fine; for *interaction* with React apps, fall back to hitting the underlying APIs with `curl`.

### Debugging workflow

When something is broken in a web app running in a cmux browser surface:

1. **Find the browser surface:** `cmux tree`
2. **Get page structure:** `cmux browser --surface <id> snapshot`
3. **Check visible text:** `cmux browser --surface <id> get text "body"`
4. **Check captured console output:** `cmux browser --surface <id> console list`
5. **Check captured page errors:** `cmux browser --surface <id> errors list`
6. **If the page won't load at all**, try `127.0.0.1` instead of `localhost`.
7. **For API-level bugs, use `curl` directly** — and remember `eval`/`press` are currently broken (manaflow-ai/cmux#2610), so API-level testing is the robust path.

## Intelligent Splitting

Don't blindly split in the same direction every time — that creates unusably narrow or short panes. Before splitting, `cmux tree` and read the shape:

- **1 pane** → single full-screen. Split right for a natural side-by-side.
- **2 panes, side-by-side** → already left/right. Split one of them down for an L-shape.
- **2 panes, stacked** → already top/bottom. Split one right to avoid a third vertical slice.
- **3+ panes** → the workspace is getting crowded. Prefer a **new workspace** over another split:
  ```bash
  cmux new-workspace --name "agent-4" --cwd /path/to/project
  ```

**Match surface type to position:**

- **Terminals with scrolling output** (servers, logs, test watchers) → vertical splits are fine (tall and narrow works for log lines)
- **Browsers and editors** → need horizontal width; split right, not down
- **Monitoring / quick-glance** → bottom splits (wide and short)

**Respect user intent.** If the user rearranged panes since your last check, treat their layout as intentional: if they widened a pane, don't split it; if they closed one of yours, don't recreate it; if they moved a browser left, put new terminals on the right.

When in doubt on a complex layout (4+ panes), ask the user rather than guessing wrong.

**Example — setting up a dev environment:**
```bash
# Start with one pane (your coding session). Add dev server to the right:
cmux new-split right
cmux send --surface surface:2 "npm run dev"
cmux send-key --surface surface:2 enter

# Add test watcher below the dev server:
cmux new-split down --surface surface:2
cmux send --surface surface:3 "npm test -- --watch"
cmux send-key --surface surface:3 enter

# Result: code on left, dev server top-right, tests bottom-right
```

## Multi-Agent Orchestration

One of cmux's most powerful uses is coordinating multiple AI agents across panes. The pattern:

1. **Create panes** for each agent:
   ```bash
   cmux new-split right
   cmux tree                            # see the full hierarchy with surface IDs
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

4. **Per-agent status pills** so the sidebar tells the whole story at a glance:
   ```bash
   cmux set-status auth "running" --icon hammer --color "#007aff"
   cmux set-status db "done" --icon checkmark --color "#34c759"
   ```

5. **Notify when done:**
   ```bash
   cmux notify --title "All agents complete" --body "Auth, DB, and API modules ready"
   ```

### Polling vs. waiting

For long-running orchestration where you need to react to changes, poll `cmux tree` at reasonable intervals (every 5-15 seconds, not more). When a layout change is detected, re-resolve surface IDs and don't blindly proceed with your original plan — the user may have rearranged or closed panes, which is a signal about intent.

For strict synchronization between agents, prefer `cmux wait-for` over polling:
```bash
# Agent A signals completion
cmux wait-for --signal "auth-done"

# Agent B waits for that signal
cmux wait-for "auth-done" --timeout 60
```

## Style Guide

- Keep status pill values short (1-3 words) — they appear in a small badge
- Use clear, specific notification titles — the user may have many workspaces
- Choose log levels meaningfully: `info` for milestones, `progress` for ongoing work, `success`/`error` for outcomes, `warning` for things that need attention but aren't failures
- Clean up after yourself: clear progress bars and status pills when tasks finish
- Use `--source` on log entries when multiple subsystems are active so the user can tell them apart

## Where to go for more

- **`api-reference.md`** (in this skill directory) — exhaustive CLI listing: every subcommand, every flag, tmux compat, hooks, clipboard/buffers, global flags, environment variables.
- **`cmux --help`** / **`cmux <subcommand> --help`** — authoritative and always current. If something in these docs doesn't match the live CLI, trust the CLI.
- **`cmux capabilities --json`** — machine-readable list of every method cmux exposes; useful when you suspect a command exists but can't remember its exact name.
