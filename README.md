# claude-personal-plugin

Personal Claude Code plugin — skills and slash commands that follow the
user across every project on a given machine. Not fijit-specific.

## Skills

- `/spec-review` — iterative spec review via a fresh-context subagent
  loop. Designed and dogfooded into a tight spec; implemented in v0.1.0.
  See `docs/superpowers/specs/2026-05-16-spec-review-skill-design.md`.

## Install

This repo is a Claude Code marketplace (`claude-personal-plugin`) containing one plugin (`spec-review`) under `plugins/spec-review/`. More plugins may be added later as siblings.

From a Claude Code session:

```
/plugin marketplace add C:\Users\krzys\claude-personal-plugin
/plugin install spec-review@claude-personal-plugin
```

(First name = plugin; second name = marketplace.)

To verify: `/plugin` opens the manager and shows the plugin under the
Installed tab. `/spec-review` should appear in the slash-command palette.

**One-shot smoke test without persistent install:**

```
claude --plugin-dir "C:\Users\krzys\claude-personal-plugin"
```

Loads the plugin only for that Claude Code session.
