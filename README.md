# claude-personal-plugin

Personal Claude Code tooling — a marketplace and plugins that follow you
across every project on a given machine.

## What's in here

```
.claude-plugin/marketplace.json       Claude Code marketplace manifest
plugins/spec-review/                  the spec-review plugin (skill)
plugins/plan-review/                  the plan-review plugin (skill)
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

### `plan-review`

The same loop applied to implementation plans instead of specs. Reads
the most recent plan under `docs/superpowers/plans/` (or a path you
pass), dispatches a reviewer tuned for the plan genre — sequencing,
dependencies between steps, actionable tasks, verification guidance —
applies the catches you agree with, disputes weak ones, commits each
round, and stops on a clean verdict or a safety cap. Use it after the
`writing-plans` skill writes a plan.

## Install

Two steps.

### 1. Register the marketplace

From any Claude Code session (the command is global):

```
/plugin marketplace add krzyssikora/claude-personal-plugin
```

The `owner/repo` shorthand resolves to this GitHub repo. The full-URL
form (`/plugin marketplace add https://github.com/krzyssikora/claude-personal-plugin.git`)
also works. To register from a local clone instead, pass the clone's
absolute path in place of the shorthand.

### 2. Install the plugin(s)

```
/plugin install spec-review@claude-personal-plugin
/plugin install plan-review@claude-personal-plugin
```

Install either or both. Confirm by running `/reload-plugins` and then
`/plugin` to view the manager — the installed plugins should appear
under the Installed tab.

Each plugin ships a single skill and no command file. After install you
run it by typing `/spec-review` or `/plan-review` (optionally with a
path: `/spec-review path/to/spec.md`) — Claude Code exposes a plugin
skill directly under its own name. The model can also invoke it via the
Skill tool as `spec-review:spec-review` / `plan-review:plan-review`.

Why no command file: the skill is already invocable as `/spec-review`, so
a command file would be redundant — and a plugin command that shares its
skill's leaf name makes the Skill tool route
`Skill("spec-review:spec-review")` to the command stub instead of the
skill body. Shipping the skill alone gives the clean slash command and
sidesteps that collision. (Don't re-add a same-named command.)

## Verifying the install

In a fresh Claude Code session:

- `/plugin` shows the installed plugins (`spec-review`, `plan-review`)
  under the Installed tab with no errors.
- Typing `/spec-review` or `/plan-review` runs the skill.

## Smoke test without installing

Clone the repo, then launch a Claude Code session pointed at the clone:

```
git clone https://github.com/krzyssikora/claude-personal-plugin.git
claude --plugin-dir ./claude-personal-plugin
```

Loads the plugins only for that Claude Code session — no marketplace
registration needed. Useful when iterating on a plugin during
development. `--plugin-dir` accepts either a directory or a `.zip`
archive (Claude Code v2.1.128+) and may be repeated to load multiple
plugins.

## Repo layout reminder

This repo is *both* a Claude Code marketplace (`claude-personal-plugin`)
and the source-of-truth for the plugins (`spec-review`, `plan-review`)
inside it. Additional plugins can be added later as sibling
subdirectories under `plugins/`.
