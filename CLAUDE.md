# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A **Claude Code plugin marketplace template**: a `.claude-plugin/marketplace.json` registry plus a set of plugins under `plugins/`. There is no application code, package manager, or build system — everything is JSON manifests and Markdown (commands, skills, agents, docs). Validation is done with `jq` and shell scripts, not a test runner.

## Commands

### Validate the marketplace and all plugins

This mirrors exactly what `.github/workflows/validate-plugins.yml` (CI) runs — use it before committing:

```bash
# 1. marketplace.json must be valid JSON with name, owner.name, plugins (array)
jq empty .claude-plugin/marketplace.json

# 2. Every plugins[] entry needs name + source; author (if present) needs author.name
jq -r '.plugins[].name' .claude-plugin/marketplace.json | sort | uniq -d   # must be empty (no dup names)

# 3. For each plugin source directory: .claude-plugin/plugin.json must exist, be valid JSON,
#    have a "name" field, and any commands/*.md should start with `---` frontmatter.
```

### Validate a single plugin (interactive, inside Claude Code)

```
/plugin-development:validate      # run from the plugin's root directory
```

### Scaffold/extend a plugin (interactive, inside Claude Code)

```
/plugin-development:init <plugin-name>
/plugin-development:add-command <name> <description>
/plugin-development:add-skill <name> <when-to-use>
/plugin-development:add-agent <name> <description>
/plugin-development:add-hook <event> <matcher>
/plugin-development:test-local
```

### Test a plugin locally

```bash
claude
/plugin marketplace add .
/plugin install <plugin-name>@cc-magi-engineer
/<plugin-name>:<command>
# after edits:
/plugin uninstall <plugin-name>@cc-magi-engineer
/plugin install <plugin-name>@cc-magi-engineer
```

There is no lint/test/build command beyond the `jq`-based structural checks above and the in-session `/plugin-development:validate` command.

## Architecture

### Two levels of manifest

- **`.claude-plugin/marketplace.json`** (repo root) — the marketplace registry. Each entry in `plugins[]` has `name`, `source` (path to the plugin dir), and metadata (`version`, `author`, `category`, `tags`, `keywords`). This is what `/plugin marketplace add` reads.
- **`plugins/<name>/.claude-plugin/plugin.json`** — each plugin's own manifest (`name`, `version`, `description`, `author`, `license`, `keywords`). Component directories (`commands/`, `agents/`, `skills/`, `hooks/`) live at the **plugin root**, never inside `.claude-plugin/`, and are auto-discovered — do not add `"commands"`, `"agents"`, etc. fields to `plugin.json` pointing at standard paths (this breaks discovery).

### Plugin component types

| Type | Location | Purpose |
|---|---|---|
| Commands | `commands/*.md` | User-triggered slash commands; frontmatter needs `description` (and optionally `argument-hint`); filenames kebab-case |
| Skills | `skills/<name>/SKILL.md` | Model-invoked ambient guidance; frontmatter `name` must exactly match the directory name; `description` must state both what and when to use it (≤1024 chars, no XML tags, no reserved words `claude`/`anthropic`) |
| Agents | `agents/*.md` | Sub-agents invoked via `/agents <name>` for deep, separate-context-window analysis |
| Hooks | `hooks/hooks.json` + `scripts/*.sh` | Lifecycle automation (PreToolUse, PostToolUse, SessionStart, ...); scripts must reference `${CLAUDE_PLUGIN_ROOT}` (never relative paths) and be `chmod +x` |

### The `plugin-development` plugin is the toolkit for building the other plugins

It implements a hybrid design, all under `plugins/plugin-development/`:

- **`skills/plugin-authoring/`** — read-only ambient skill (allowed-tools: Read, Grep, Glob) that activates when `.claude-plugin/`, `plugin.json`, `marketplace.json`, or component dirs are touched. It diagnoses, then proposes running one of the slash commands below rather than editing files itself. Reference material (schemas, templates, examples, best practices) lives in progressive-disclosure subfiles under that skill directory — read those before hand-authoring a manifest.
- **`commands/`** — `init`, `add-command`, `add-skill`, `add-agent`, `add-hook`, `validate`, `test-local`. These are deterministic, template-based scaffolding/validation actions.
- **`agents/plugin-reviewer.md`** — deep multi-file release-readiness audit (structure, manifest, hook safety, marketplace readiness); invoked explicitly, not automatically.
- **`hooks/hooks.json`** + **`scripts/`** — `validate-plugin.sh` runs as a `PreToolUse` guard on `Write|Edit`: it walks up from the edited file to find the nearest `.claude-plugin/plugin.json`, and blocks (exit 2) if it's missing or if no component directory exists. `format-or-lint.sh` is a `PostToolUse` stub (currently a no-op) meant to be extended per-project.

### `docs/` vs `plugins/plugin-development/skills/plugin-authoring/`

`docs/` holds long-form reference copies of the official Claude Code plugin docs (plugins, hooks, marketplaces, settings, skills, slash-commands, sub-agents) for offline/local reading. The `plugin-authoring` skill's own `schemas/`, `templates/`, `examples/`, and `best-practices/` files are the *actionable* condensed versions used during authoring — prefer those when scaffolding, and fall back to `docs/` for full detail.

### Key invariants enforced by CI (`.github/workflows/validate-plugins.yml`) and the hooks

- `marketplace.json` must be valid JSON with `name`, `owner.name`, and a `plugins` array; no duplicate plugin `name`s.
- Every `plugins[].source` directory must contain `.claude-plugin/plugin.json` with at least a `name` field; `author`, if present, must be an object with `name`; `repository`, if present, must be a string URL (not an object).
- Command Markdown files should open with `---` frontmatter (warning, not hard failure, in CI; hard failure via the plugin's own PreToolUse hook only checks for the presence of `.claude-plugin/plugin.json` and at least one component directory).
