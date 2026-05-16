# claude-personal-plugin

Personal Claude Code plugin — skills and slash commands that follow the
user across every project on a given machine. Not fijit-specific.

## Skills

- `/spec-review` — iterative spec review via a fresh-context subagent
  loop. Designed and dogfooded into a tight spec; implemented in v0.1.0.
  See `docs/superpowers/specs/2026-05-16-spec-review-skill-design.md`.

## Install

This repo is both the marketplace and the single plugin it contains.

From a Claude Code session:

```
/plugin marketplace add C:\Users\krzys\claude-personal-plugin
/plugin install claude-personal-plugin@claude-personal-plugin
```

(First name = plugin; second name = marketplace; they happen to match.)

To verify: `/plugin` opens the manager and shows the plugin under the
Installed tab. `/spec-review` should appear in the slash-command palette.

**One-shot smoke test without persistent install:**

```
claude --plugin-dir "C:\Users\krzys\claude-personal-plugin"
```

Loads the plugin only for that Claude Code session.
