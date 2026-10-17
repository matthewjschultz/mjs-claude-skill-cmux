# mjs-claude-skill-cmux

A Claude Code skill for working inside [cmux](https://github.com/manaflow-ai/cmux), the native macOS terminal built on Ghostty for running AI coding agents in parallel.

## Why this exists

cmux exposes almost everything through its CLI: workspaces, split panes, sending input to other terminals, sidebar status pills, progress bars, notifications, and an embedded browser with an automation API. Without guidance, Claude reaches for plausible commands that don't exist (`send-surface`, `list-surfaces`, `new-split --browser`), opens browsers in terminal panes, and floods the sidebar with noise. This skill carries the real command surface, the patterns that work, and the mistakes that came up again and again in real sessions.

It activates on its own whenever Claude Code is running inside cmux (detected via `CMUX_WORKSPACE_ID` / `CMUX_SURFACE_ID`), and stays out of the way otherwise.

## What it covers

- Discovering the layout with `cmux tree` and `cmux identify`, and splitting panes sensibly based on what's already open, without stealing the user's focus.
- Starting commands in new panes with `--command`, sending input to shells, and reading their output.
- Orchestrating several agents at once, and talking to them with `cmux agent message` instead of typing into their terminals.
- Sidebar feedback for long-running work: status pills, progress, log entries, and completion notifications, used only when they help.
- Showing results in panes: the Markdown viewer, the diff viewer, and window screenshots and recordings.
- Browser automation: opening browser panes, inspecting pages, `eval`, `press`, dialogs, viewport emulation, and `cmux browser repl` for Playwright-style scripting with trusted input.
- `wait-for` signals for synchronizing agents across panes.

Everything in the skill is verified against cmux 0.65.2. `eval`, `press`, and `addscript` failed on every call in cmux 0.63 ([manaflow-ai/cmux#2610](https://github.com/manaflow-ai/cmux/issues/2610)); they work on 0.65.2, and the skill now uses them.

cmux also ships its own agent guide and skills (`cmux guide`, `cmux docs <topic>`). This skill is a single, opinionated pattern guide for Claude Code that points to them for depth.

## Contents

- `skills/cmux/SKILL.md`: the pattern guide: detection, quick reference, commands that don't exist, browser automation, intelligent splitting, and multi-agent orchestration.
- `skills/cmux/api-reference.md`: exhaustive CLI reference for flags and lesser-used subcommands.
- `evals/`: task evals (`evals.json`) and trigger evals (`trigger_eval.json`).
- `cmux.skill`: the packaged skill file.

## Install

```
/plugin marketplace add matthewjschultz/mjs-claude-skills
/plugin install cmux@mjs-claude-skills
```

Or download `cmux.skill` and install it directly.

## Credits

This skill documents [cmux](https://github.com/manaflow-ai/cmux) by Manaflow. It is an independent companion and is not affiliated with or endorsed by the cmux authors.

## License

MIT. See [LICENSE](LICENSE).
