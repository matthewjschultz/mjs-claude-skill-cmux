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
| Run JavaScript in the page | `cmux browser --surface <id> eval '<js>'` |
| Press a key in the page | `cmux browser --surface <id> press Enter` |
| Show notification | `cmux notify --title "Title" --body "Body"` |
| Sidebar status pill | `cmux set-status <key> "value" --icon <name> --color "#hex"` |
| Sidebar progress bar | `cmux set-progress 0.5 --label "Working..."` |
| Sidebar log entry | `cmux log --level success "Done"` |

IDs accepted everywhere: UUIDs, short refs (`window:1`, `workspace:2`, `pane:3`, `surface:4`), or indexes.

## Common Mistakes — Never Do This

These commands **do not exist**. Using them is the most common cmux failure mode. They are plausible guesses that happen to be wrong — always prefer the "Use instead" column:

| Don't use | Use instead | Why |
|-----------|-------------|-----|
| `cmux send-surface <id> "text"` | `cmux send --surface <id> "text"` | `send-surface` was never a real command |
| `cmux send-key-surface <id> enter` | `cmux send-key --surface <id> enter` | Same — `--surface` is a flag, not a suffix |
| `cmux list-surfaces` / `--json` | `cmux tree` | No `list-surfaces` command exists |
| `cmux new-split right --browser` | `cmux new-pane --type browser --direction right` | `new-split` has no `--browser` flag and always makes terminals |
| `cmux browser eval 'await fetch(...)'` | `cmux browser eval '(async () => { ... })()'` | Top-level `await` is a `SyntaxError`; a returned promise is awaited for you |
| `http://localhost:...` in the browser | `http://127.0.0.1:...` | Older builds couldn't resolve `localhost`; `127.0.0.1` works everywhere |

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

cmux has an embedded browser with an automation API. Two rules that catch most mistakes:

1. **Always target a surface explicitly.** Pass `--surface <id>` (conventionally right after `browser`) or the surface as the first positional token: `cmux browser --surface surface:10 snapshot` and `cmux browser surface:10 snapshot` are equivalent. Get the id from `cmux tree` or from the output of `cmux browser open`.

2. **Use `127.0.0.1`, not `localhost`.** Older cmux builds couldn't resolve `localhost` in the embedded browser ("This page couldn't load"). Current builds can, but `127.0.0.1` works on every version and avoids IPv4/IPv6 ambiguity with dev servers, so substitute it in every URL you pass to `open`, `goto`, etc.:
   ```bash
   cmux browser open http://127.0.0.1:3000
   ```

### Running JavaScript in the page

`eval`, `press`, and `addscript` work on current cmux (verified on 0.65.2; they failed with `js_error` on every call in 0.63 — manaflow-ai/cmux#2610). Use them freely; they're usually faster and more precise than stitching together `get` calls.

```bash
cmux browser --surface <id> eval 'document.title'
cmux browser --surface <id> eval "document.querySelectorAll('button').length"
cmux browser --surface <id> eval '[...document.querySelectorAll("a")].map(a => a.href)'
cmux browser --surface <id> eval '(async () => (await fetch("/api/health")).status)()'
```

How `eval` behaves:

- **It returns the value of the last expression.** Objects and arrays come back as pretty-printed JSON; `null` and `undefined` print as such. Multiple statements are fine (`const n = 5; n * 2` → `10`).
- **Promises are awaited** — `eval 'fetch("/").then(r => r.status)'` prints `200`. **Top-level `await` is a `SyntaxError`**; wrap it in an async IIFE: `eval '(async () => { await ...; return x })()'`.
- **Exceptions are reported with their real message** and exit 1: `Error: js_error: ReferenceError: Can't find variable: foo`. Read the message — it's the page's actual error.
- **Quote for the shell.** Single-quote the script so `$`, `!`, and backticks aren't expanded; use double quotes inside the JS. `--script '<js>'` is an equivalent explicit form.
- **Return data rather than logging it.** `console.log` from `eval`/`addscript` usually shows up in `console list`, but on some strict-CSP sites (e.g. github.com) console capture stays empty even though the script ran.

Keyboard input and injection:

```bash
cmux browser --surface <id> press Enter          # also: key / keydown / keyup
cmux browser --surface <id> press Escape         # W3C key names: Tab, ArrowDown, Backspace, b, ...
cmux browser --surface <id> addscript 'window.__debug = true'
cmux browser --surface <id> addstyle '.cookie-banner { display: none !important }'
cmux browser --surface <id> wait --function 'document.querySelectorAll(".row").length > 10' --timeout-ms 5000
```

- `press` dispatches real key events to the focused element — a `keydown` listener on the page sees them. Focus the target first (`click` or `focus <selector>`) if it matters which element receives the key.
- `addscript` (alias `addinitscript`) runs the script in the current page immediately. **It does not survive navigation** — re-run it after every `goto`/`reload`.
- `wait --function` polls a JS predicate until it's truthy — use it instead of `sleep` when waiting on app state.

### Inspection primitives

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

`snapshot` returns a structured accessibility tree — the fastest way to understand page state. `--interactive` assigns refs to clickable elements, but see the next section: those refs are fragile. For anything the `get` variants can't express (computed values, filtered lists, app state on `window`), use `eval`.

### Interacting with pages — two caveats

1. **CSS selectors are more reliable than ref selectors.** Refs from `snapshot --interactive` expire between snapshots and frequently fail with "Element not found". Prefer CSS:
   ```bash
   # Reliable
   cmux browser --surface surface:10 click "button:nth-child(5)" --snapshot-after
   # Often fails
   cmux browser --surface surface:10 click '[ref=e12]'
   ```

2. **Synthetic clicks may not trigger React state changes.** `cmux browser click` (and `element.click()` / dispatched MouseEvents) often do not propagate through React's synthetic event system, so tab switches and other React-controlled interactions can silently do nothing. Always confirm an interaction landed (`snapshot`, or `eval` on the state you expected to change). If it didn't, try keyboard input (`focus <selector>` then `press Enter` / `press Space`), or hit the underlying APIs with `curl` or `eval '(async () => ...)()'` with `fetch`.

### Debugging workflow

When something is broken in a web app running in a cmux browser surface:

1. **Find the browser surface:** `cmux tree`
2. **Get page structure:** `cmux browser --surface <id> snapshot`
3. **Check visible text:** `cmux browser --surface <id> get text "body"`
4. **Check captured console output:** `cmux browser --surface <id> console list`
5. **Check captured page errors:** `cmux browser --surface <id> errors list`
6. **Probe live state with `eval`:** app globals, `localStorage`, computed styles, or an in-page `fetch` that carries the page's cookies — e.g. `eval 'JSON.stringify(localStorage)'`.
7. **If the page won't load at all**, try `127.0.0.1` instead of `localhost`.
8. **For API-level bugs, use `curl` directly** — or `eval` with `fetch` when the request needs the page's session.

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
