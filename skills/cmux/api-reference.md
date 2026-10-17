# cmux API Reference

Exhaustive CLI reference for cmux. SKILL.md covers the patterns and gotchas; this file is where to look up a specific flag, subcommand, or lesser-used feature.

IDs accepted everywhere: UUIDs, short refs (`window:1`, `workspace:2`, `pane:3`, `surface:4`), or indexes.

## Sanity checks

If a command in this file doesn't work as described, verify against the live CLI before assuming the docs are right:

```bash
cmux --help
cmux <subcommand> --help
cmux capabilities --json     # machine-readable list of all methods
cmux version
cmux guide                   # cmux's own agent guide
cmux docs <topic>            # official docs and skill links: api, browser, agents, capture, settings, ...
```

cmux evolves faster than this file. When in doubt, the CLI is authoritative.

## Workspaces

Workspaces are the top-level organizational unit — each appears as a tab in the sidebar.

```bash
cmux list-workspaces                           # List all workspaces
cmux new-workspace [--name <title>] [--description <text>] [--cwd <path>] [--command <text>] [--focus <true|false>]
cmux current-workspace                         # Get current workspace info
cmux select-workspace --workspace <id|ref>
cmux close-workspace --workspace <id|ref>
cmux rename-workspace [--workspace <id|ref>] <title>
cmux reorder-workspace --workspace <id|ref> (--index <n> | --before <id|ref> | --after <id|ref>)
```

### Workspace actions

Context-menu operations including color and pinning:

```bash
cmux workspace-action --action set-color --color <name|#hex> [--workspace <id|ref>]
cmux workspace-action --action clear-color [--workspace <id|ref>]
cmux workspace-action --action pin [--workspace <id|ref>]
cmux workspace-action --action unpin [--workspace <id|ref>]
cmux workspace-action --action rename --title <text> [--workspace <id|ref>]
```

Named colors: `Red`, `Crimson`, `Orange`, `Amber`, `Olive`, `Green`, `Teal`, `Aqua`, `Blue`, `Navy`, `Indigo`, `Purple`, `Magenta`, `Rose`, `Brown`, `Charcoal`. Colors also accept `#RRGGBB` hex values.

## Windows

```bash
cmux list-windows
cmux current-window
cmux new-window
cmux focus-window --window <id>
cmux close-window --window <id>
cmux move-workspace-to-window --workspace <id|ref> --window <id|ref>
```

## Panes and surfaces

Panes contain surfaces. A pane can hold multiple surfaces (tabs).

```bash
cmux new-split <left|right|up|down> [--workspace <id|ref>] [--surface <id|ref>] [--command <text>] [--focus <true|false>]
cmux new-pane [--type <terminal|browser>] [--direction <left|right|up|down>] [--workspace <id|ref>] [--url <url>] [--command <text>] [--focus <true|false>]
cmux new-surface [--type <terminal|browser>] [--pane <id|ref>] [--workspace <id|ref>] [--url <url>] [--command <text>] [--focus <true|false>]
cmux list-panes [--workspace <id|ref>]
cmux list-pane-surfaces [--workspace <id|ref>] [--pane <id|ref>]
cmux tree [--all] [--workspace <id|ref>]       # Full hierarchy — preferred discovery command
cmux --json tree --all                         # Same, as JSON (windows → workspaces → panes → surfaces)
cmux identify --json                           # Caller and focused refs, version, socket path
cmux focus-pane --pane <id|ref> [--workspace <id|ref>]   # Moves the user's focus; only when asked
cmux close-surface [--surface <id|ref>] [--workspace <id|ref>]
cmux split-off --surface <id|ref> <left|right|up|down> [--focus <true|false>]   # Move a tab into its own new split
cmux resize-pane --pane <id|ref> [--workspace <id|ref>] (-L|-R|-U|-D) [--amount <n>]
```

Remember: **`new-split` always creates terminals.** For a browser pane, use `new-pane --type browser` or `cmux browser open`.

`--command <text>` (terminals only) starts the normal interactive shell and types the text plus Enter, so the command runs at once and the shell survives it. `--focus false` leaves the user's focus where it is. `cmux --json <command>` returns the new `surface_ref` / `pane_ref` as JSON.

## Surfaces / tabs

Surfaces appear as tabs within panes. Rename them, reorder them, move them between panes, and perform context-menu actions.

```bash
cmux rename-tab <title>                        # Rename current surface's tab
cmux rename-tab --surface <id|ref> <title>
cmux tab-action --action <name> [--tab <id|ref>] [--surface <id|ref>] [--workspace <id|ref>]

cmux move-surface --surface <id|ref> [--pane <id|ref>] [--before <id|ref>] [--after <id|ref>] [--index <n>]
cmux reorder-surface --surface <id|ref> (--index <n> | --before <id|ref> | --after <id|ref>)
cmux surface-health [--workspace <id|ref>]
cmux refresh-surfaces                          # Refresh snapshots for focused workspace
```

Tab actions: `rename`, `clear-name`, `close-left`, `close-right`, `close-others`, `new-terminal-right`, `new-browser-right`, `reload`, `duplicate`, `pin`, `unpin`, `mark-unread`.

## Sending input

```bash
cmux send "echo hello"                         # To current surface
cmux send --surface <id|ref> "echo hello"      # To specific surface
cmux send-key enter                            # Keypress to current surface
cmux send-key --surface <id|ref> enter         # Keypress to specific surface
cmux paste --surface <id|ref> [--submit] [text | -]   # Bracketed paste; --submit presses Enter after
cmux send-panel --panel <id|ref> "text"
cmux send-key-panel --panel <id|ref> <key>
```

Available keys: `enter`, `tab`, `escape`, `backspace`, `delete`, `up`, `down`, `left`, `right`.

Text is sent as typed input. Follow with `cmux send-key enter` if you need Enter, or include `\n` in the text. Don't use any of these to talk to another agent; use `cmux agent message`.

## Agent messages

Deliver a message to the Claude Code or Codex session running in another workspace or surface, through its hooks rather than its terminal:

```bash
cmux agent message <target> [--from <name>] <text|->   # target: workspace/surface ref or workspace title
cmux agent message --reply-to <id> [--from <name>] <text|->
cmux agent inbox [--surface <target>] [--state queued|delivered|read|failed] [--mark-read]
cmux agent messages [on|off|status] [<target>] [--workspace]
```

States go `queued` → `delivered` → `read`. Bodies are text only, up to 32 KiB. Headless `claude -p` / `codex exec` runs don't receive messages. Messages can be turned off globally (`agentMessages.enabled` in `~/.config/cmux/cmux.json`), per agent, or per workspace.

## Reading screen content

```bash
cmux read-screen                               # Current surface
cmux read-screen --surface <id|ref>
cmux read-screen --scrollback                  # Include scrollback buffer
cmux read-screen --lines 50                    # Last N lines
```

## Notifications

```bash
cmux notify --title "Title" --body "Body"
cmux notify --title "Title" --subtitle "Sub" --body "Body"
cmux notify --title "Title" --body "Body" --workspace <id|ref> --surface <id|ref>
cmux list-notifications
cmux clear-notifications
```

## Sidebar metadata

### Status pills

Small labeled badges. Use a unique key per tool/concern so they don't collide.

```bash
cmux set-status build "compiling" --icon hammer --color "#ff9500"
cmux set-status tests "running" --icon checkmark --color "#007aff" [--priority <n>]
cmux clear-status build
cmux list-status
```

`list-status` also shows the pills cmux's own agent hooks maintain (`claude_code`, `program_status`); leave those alone.

### Progress bar

A single progress bar (0.0 to 1.0) with an optional label:

```bash
cmux set-progress 0.5 --label "Building... (50%)"
cmux clear-progress
```

### Log entries

Scrollable log in the sidebar. Levels: `info`, `progress`, `success`, `warning`, `error`.

```bash
cmux log "Starting task"
cmux log --level progress --source build "Compiling modules..."
cmux log --level success "All done"
cmux log --level error --source tests "3 failures"
cmux clear-log
cmux list-log --limit 10
```

### Full sidebar dump

```bash
cmux sidebar-state [--workspace <id|ref>]
```

### Todo list

`cmux todo <add|list|check|uncheck|start|rm|clear>` manages a per-workspace checklist in the sidebar. It belongs to the user: change it only when they ask, and use your own task tracking for your plans.

## Showing results

```bash
cmux markdown open <path> [--focus false]      # Formatted Markdown viewer, live-reloads on change
cmux diff [--staged|--branch|--last-turn] [--base <ref>] [--cwd <path>] [--layout split|unified] [--focus false]
cmux diff <patch-file|->                       # View a patch
cmux shot [--out <path>] [--format png|jpg] [--max-width <n>] [--region <x,y,w,h>]   # cmux window only
cmux record start [--out <path>] [--gif] [--max-seconds <n>] | record stop | record status
```

`shot` and `record` capture only the cmux window and need no Screen Recording permission.

## Browser automation

The embedded browser has its own automation API. See SKILL.md for the high-level rules (explicit surface targeting, `127.0.0.1` rule, `eval` semantics, when to use `repl`). This section lists the commonly used subcommands; `cmux browser --help` has the full set.

Target a surface with `--surface <id>` after `browser`, or pass it as the first positional token (`cmux browser surface:10 snapshot`). Selectors are CSS or refs from `snapshot --interactive`.

```bash
cmux browser open [url] [--focus false]        # Open/reuse a browser surface; prints surface id (--json: surface_ref)
cmux browser open-split [url]                  # Explicit split
cmux browser goto|navigate <url> [--snapshot-after]
cmux browser back|forward|reload [--snapshot-after]
cmux browser snapshot [--interactive] [--compact] [--selector <css>] [--max-depth <n>]
cmux browser find <role|text|label|placeholder|alt|title|testid|first|last|nth> [...]   # role: [--name <text>] [--exact] <role>
cmux browser click|dblclick|hover|focus <selector> [--snapshot-after]   # Synthetic: isTrusted is false
cmux browser check|uncheck|scroll-into-view <selector> [--snapshot-after]
cmux browser type <selector> <text> [--snapshot-after]
cmux browser fill <selector> [text] [--snapshot-after]   # Empty text clears the input
cmux browser select <selector> <value> [--snapshot-after]
cmux browser scroll [--selector <css>] [--dx <n>] [--dy <n>] [--snapshot-after]
cmux browser set-input-files --selector <css> (--file <path> ... | --clear)
cmux browser screenshot [--out <path>] [--json] # Without --out, saves to a temp file and prints its path
cmux browser wait [--selector <css>] [--text <text>] [--url-contains <text>] [--load-state interactive|complete] [--function <js>] [--timeout-ms <ms>]
cmux browser get <url|title|text|html|value|attr|count|box|styles> [...]   # styles: [--property <name>]
cmux browser is <visible|enabled|checked> <selector>
cmux browser eval '<js>'                       # Prints the last expression's value (JSON for objects)
cmux browser press|key|keydown|keyup <key>     # Trusted key events; W3C names: Enter, Escape, Tab, ArrowDown, b, ...
cmux browser dialog <accept|dismiss> [text]    # Answer a pending alert/confirm/prompt
cmux browser addscript '<js>'                  # Alias: addinitscript. Runs now; does not survive navigation
cmux browser addstyle '<css>'                  # Inject CSS into the current page
cmux browser highlight <selector>              # Outline an element for the user
cmux browser viewport <width> <height> | reset # Emulate a CSS-pixel viewport inside the pane
cmux browser frame <selector|main>             # Switch into an iframe and back
cmux browser cookies <get|set|clear> [...]
cmux browser storage <local|session> <get|set|clear> [...]
cmux browser state <save|load> <path>          # Cookies + storage to/from a file
cmux browser download list [--json] | download wait [--path <path>]
cmux browser focus-webview | is-webview-focused
cmux browser console <list|clear>              # Read/clear captured console messages
cmux browser errors <list|clear>               # Read/clear captured page errors
cmux browser tab <new|list|switch|close|<index>> [...]
cmux browser repl [--session <name>] [--eval <code>|-] [--timeout <ms>] [<code>]   # Playwright JS; see below
```

On 0.65.2 the WKWebView browser returns `not_supported` for `network route`, `trace`, `screencast`, `input`, `geolocation`, and `offline`.

### `repl` quick reference

```bash
cmux browser repl 'await page.goto("http://127.0.0.1:3000"); await snapshot({ interactive: true })'
cmux browser repl --session s 'const p = await tabs.use((await tabs.list())[0].id); 1'   # attach to a visible tab
cmux browser repl --session s 'p.on("dialog", d => d.accept()); await p.getByRole("button", { name: "Delete" }).click()'
cmux browser repl --session s 'await p.locator("e5").fill("me@x.com"); await snapshot()'
cmux browser repl --eval - < script.js
cmux browser repl list | reset <session> | guide
cmux browser repl mcp [--session <name>]       # Serve the REPL as an MCP server on stdio
```

Globals: `page`, `tabs` (`list()`, `open(url, { background })`, `use(id)`, `get(id)`, `content({ urls, format })`, `history()`), `snapshot()`, `screenshot()`, `fetch()` (with the tab's cookies), `fs`/`path`/`os`/`Buffer` (limited to the working and temp directories), `sleep()`, `session` (`keep()`, `storageState()`, `setStorageState()`, `allowedDomains()`, `configure()`, `record()`), `secrets`, `search()`, `tools`. Page additions beyond Playwright: `page.markdown()`, `page.extract()`, `page.searchText()`, `page.scrollToText()`, `page.consoleMessages()`, `page.errors()`, `page.dialog()`, `page.fileChooser()`, `page.clipboard`, `page.elementAt(x, y)`, `page.keep()`. `cmux browser repl guide` documents each one.

### `eval` quick reference

```bash
cmux browser --surface surface:10 eval 'document.title'
cmux browser --surface surface:10 eval '({ w: innerWidth, h: innerHeight })'     # → JSON object
cmux browser --surface surface:10 eval 'fetch("/api").then(r => r.status)'      # promise is awaited
cmux browser --surface surface:10 eval '(async () => { await x(); return 1 })()' # top-level await is a SyntaxError
cmux browser --surface surface:10 eval --script 'document.title'                # explicit form
```

Exceptions exit 1 with the page's real message, e.g. `Error: js_error: Error: boom`.

### `get` variants

`get` extracts specific fields from the page without writing JS:

```bash
cmux browser --surface surface:10 get title
cmux browser --surface surface:10 get url
cmux browser --surface surface:10 get text "body"
cmux browser --surface surface:10 get text "h1"
cmux browser --surface surface:10 get html "#root"
cmux browser --surface surface:10 get value "input[name=email]"
cmux browser --surface surface:10 get attr "a.primary" "href"
cmux browser --surface surface:10 get count "button"
```

## Utility

```bash
cmux ping                                      # Check if cmux is responsive
cmux capabilities                              # List available methods
cmux capabilities --json                       # JSON output for scripting
cmux identify [--workspace <id|ref>] [--surface <id|ref>]
cmux version
cmux tree --all                                # Full hierarchy of all windows/workspaces/panes/surfaces
cmux find-window [--content] [--select] <query>
```

## Hooks

Event-driven automation:

```bash
cmux set-hook <event> <command>
cmux set-hook --list
cmux set-hook --unset <event>
```

## Clipboard / buffers

```bash
cmux set-buffer [--name <name>] <text>
cmux list-buffers
cmux paste-buffer [--name <name>] [--workspace <id|ref>] [--surface <id|ref>]
```

## Wait-for synchronization

For coordinated multi-agent workflows where you want to block on an event rather than poll:

```bash
cmux wait-for --signal "auth-done"             # Raise a named signal
cmux wait-for "auth-done" --timeout 60         # Wait on it (seconds)
```

## tmux compatibility

cmux provides tmux-compatible commands for scripts that expect tmux:

```bash
cmux capture-pane [--scrollback] [--lines <n>] # Alias for read-screen
cmux swap-pane --pane <id|ref> --target-pane <id|ref>
cmux break-pane [--workspace <id|ref>] [--pane <id|ref>]
cmux join-pane --target-pane <id|ref>
cmux next-window | previous-window | last-window
cmux last-pane
cmux respawn-pane [--command <cmd>]
```

## Global flags

These work on most commands:

| Flag | Purpose |
|------|---------|
| `--workspace <id\|ref>` | Target a specific workspace |
| `--surface <id\|ref>` | Target a specific surface |
| `--pane <id\|ref>` | Target a specific pane |
| `--window <id\|ref>` | Target a specific window |
| `--id-format refs\|uuids\|both` | Control identifier format in output |

## Environment variables

| Variable | Description |
|----------|-------------|
| `CMUX_WORKSPACE_ID` | Auto-set: current workspace ID, used as default `--workspace` |
| `CMUX_SURFACE_ID` | Auto-set: current surface ID, used as default `--surface` |
| `CMUX_TAB_ID` | Optional: used by tab-action/rename-tab as default `--tab` |
| `CMUX_SOCKET_PATH` | Override socket path (default: `~/.local/state/cmux/cmux.sock`; before 0.65, `~/Library/Application Support/cmux/cmux.sock`) |
| `CMUX_SOCKET_PASSWORD` | Socket auth password (if required) |

## Nonexistent commands

These were never real commands. Never use them:

| Don't use | Use instead |
|-----------|-------------|
| `cmux send-surface` | `cmux send --surface <id>` |
| `cmux send-key-surface` | `cmux send-key --surface <id>` |
| `cmux list-surfaces` / `cmux list-surfaces --json` | `cmux tree` |
| `cmux new-split --browser` | `cmux new-pane --type browser --direction <dir>` |
