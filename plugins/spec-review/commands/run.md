---
description: Iteratively review a spec via a fresh-context reviewer subagent until it reports no remaining issues (or a safety cap fires). Pass an optional path to the spec; with no path, the most recent file under docs/superpowers/specs/ is used.
argument-hint: "[path-to-spec.md]"
---

Invoke the `spec-review:spec-review` skill via the Skill tool. Forward `$ARGUMENTS` verbatim to the skill as its `spec` argument — no tokenization, no quoting. The skill handles empty-argument discovery and validation.
