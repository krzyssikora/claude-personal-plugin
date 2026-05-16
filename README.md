# claude-personal-plugin

Personal Claude Code tooling — a marketplace, plugins, and optional
user-level slash commands that follow the user across every project on
a given machine. Not fijit-specific.

## What's in here

```
.claude-plugin/marketplace.json       Claude Code marketplace manifest
plugins/spec-review/                  the spec-review plugin (skill + command)
user-commands/spec-review.md          optional user-level slash command
docs/superpowers/specs/               design specs
docs/superpowers/plans/               implementation plans
```

## Skills

### `spec-review`

Iterative spec review via a fresh-context reviewer subagent. Dispatches
a reviewer, applies catches you (the parent agent) agree with, disputes
weak ones, commits each round, and stops on a clean verdict or a safety
cap. Designed by dogfooding the loop on its own spec — see
`docs/superpowers/specs/2026-05-16-spec-review-skill-design.md`.

## Install

Three steps. Step 3 is optional but recommended.

### 1. Register the marketplace

From any Claude Code session (the command is global):

```
/plugin marketplace add C:\Users\krzys\claude-personal-plugin
```

### 2. Install the plugin

```
/plugin install spec-review@claude-personal-plugin
```

Confirm by running `/reload-plugins` and then `/plugin` to view the
manager — `spec-review` should appear under the Installed tab.

After this step, the slash command is registered as
`/spec-review:spec-review` (Claude Code namespaces every plugin command
as `<plugin-name>:<command-name>` and the doubled form is unavoidable
when plugin name == command name). The skill is available via the Skill
tool as `spec-review:spec-review`.

### 3. (Optional) Install the user-level shortcut for clean `/spec-review`

Plugin commands are always namespaced. To get `/spec-review` directly,
copy the user-level shortcut into your `~/.claude/commands/`:

**Windows (PowerShell):**

```
New-Item -ItemType Directory -Force -Path $HOME\.claude\commands | Out-Null
Copy-Item user-commands\spec-review.md $HOME\.claude\commands\spec-review.md
```

**Bash / macOS / Linux:**

```
mkdir -p ~/.claude/commands
cp user-commands/spec-review.md ~/.claude/commands/spec-review.md
```

(The `~/.claude/commands/` directory is created on demand; if you've
never installed a user-level slash command before, it won't exist.)

After this, `/spec-review` (un-namespaced) is registered as a user-level
slash command and forwards `$ARGUMENTS` to the plugin's
`spec-review:spec-review` skill via the Skill tool. The skill itself
still lives in the plugin — the user-level file is just a thin shim.

`/reload-plugins` in any open session picks up the new command.

## Verifying the install

In a fresh Claude Code session:

- `/plugin` shows `spec-review` under the Installed tab with no errors.
- `/spec-review:spec-review --help`-style invocations (or just typing
  `/spec-review:spec-review` and observing the agent's response) reach
  the plugin's slash command body.
- If you did Step 3, `/spec-review` works the same way.

## Smoke test without installing

```
claude --plugin-dir "C:\Users\krzys\claude-personal-plugin"
```

Loads the plugin only for that Claude Code session — no marketplace,
no user-level shortcut. Useful when iterating on the plugin during
development.

## Repo layout reminder

This repo is *both* a Claude Code marketplace (`claude-personal-plugin`)
and the source-of-truth for one plugin (`spec-review`) inside it.
Additional plugins can be added later as sibling subdirectories under
`plugins/`. Additional user-level slash commands can be added as
sibling files under `user-commands/`.
