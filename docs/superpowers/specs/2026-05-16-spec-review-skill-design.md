# `/spec-review` — design

**Date:** 2026-05-16
**Status:** Design approved, ready for implementation plan.
**Scope:** A personal-plugin skill that automates iterative spec review
inside a single Claude Code session.

## Goal

Replace a manual two-session ping-pong workflow with a single slash
command, `/spec-review`, that iterates a spec to a clean verdict using a
fresh-context reviewer subagent — without losing the discipline of
fresh-eyes review or the parent's ability to push back on weak catches.

## Background: the manual workflow being replaced

Today the author runs two Claude Code sessions side-by-side:

1. **Session 1** writes a spec to `docs/superpowers/specs/`.
2. **Session 2** is opened with a fresh prompt: *"Review the spec at
   `<path>`. Do not make changes. List clearly and in detail all aspects
   that should be addressed."*
3. The author copies Session 2's response into Session 1, prefixed with
   "I asked another agent to review the spec:".
4. Session 1 usually accepts the catches and updates the spec.
5. The author returns to Session 2 with "the spec has been updated,
   could you give it another look?"
6. Loop until Session 2 reports no remaining issues.

This is effective — it surfaces real bugs and tightens the spec — but
mechanical and slow.

## Key insight enabling automation

Claude Code's `Agent` tool dispatches a **subagent with no access to the
parent's conversation**. That gives the same fresh-context property the
author currently obtains by opening a second session. The whole loop can
therefore live in a single session, with the parent agent driving and a
new subagent dispatched per round.

## Design decisions

| Decision                | Choice                                                                                     |
| ----------------------- | ------------------------------------------------------------------------------------------ |
| Automation level        | Full autonomous loop until clean verdict or safety cap.                                    |
| Push-back rule          | Parent decides per catch: apply, or push back with reasoning carried to the next round.    |
| Distribution            | Personal Claude Code plugin in its own git repo, installed once per machine.               |
| Entry point             | Slash command `/spec-review [path]` that invokes the skill.                                |
| Inter-round persistence | Lives in the parent agent's conversation context, not on disk.                             |
| Per-round commit        | Yes — each round that applies catches creates one git commit, so `git log` is the trail.   |
| Safety cap              | `MAX_ROUNDS = 5` by default.                                                               |

## Plugin layout

```
claude-personal-plugin/                       (git repo, ~/claude-personal-plugin/)
├── .claude-plugin/
│   └── plugin.json                           # plugin manifest
├── commands/
│   └── spec-review.md                        # slash command entry
├── skills/
│   └── spec-review/
│       ├── SKILL.md                          # parent loop instructions
│       └── reviewer-prompt.md                # strict reviewer template
└── docs/
    └── superpowers/
        └── specs/
            └── 2026-05-16-spec-review-skill-design.md   (this file)
```

The slash command file delegates to the skill so the skill remains
invokable directly via the Skill tool (e.g., right after brainstorming
writes a fresh spec, the agent can self-invoke it without the user
typing the command).

## Invocation flow

1. User types `/spec-review` (or `/spec-review path/to/spec.md`).
2. The slash command's body instructs the parent agent to invoke the
   `spec-review` skill, passing through `$ARGUMENTS`.
3. The skill instructs the parent to:
   1. Resolve the target spec:
      - argument given → use it as a path (relative to cwd or absolute);
      - no argument → most recently modified `*.md` under
        `docs/superpowers/specs/` by mtime;
      - neither resolves → print an error and stop.
   2. Load `reviewer-prompt.md`.
   3. Run the loop (below) until a stop condition fires.
   4. Print the final report.

## The reviewer subagent prompt

Stored at `skills/spec-review/reviewer-prompt.md`. Verbatim contents:

```
You are reviewing a spec at <SPEC_PATH>. You will NOT make any changes —
your only job is to find issues clearly and in detail.

Read the spec fully. Then return a structured review.

CRITICAL — blocks correct implementation:
  missing requirements, contradictions, undefined behavior at boundaries,
  unstated dependencies, "magic" steps with no explanation.

IMPORTANT — would make implementation noticeably worse or error-prone:
  vague requirements, missed edge cases, ambiguous interfaces,
  invariants that aren't pinned down, weak test guidance.

MINOR — refinements that would make the spec clearer or implementation
  easier.

For each issue, give:
  (1) what's wrong,
  (2) where in the spec (section / quote),
  (3) what specifically should be addressed.

<PRIOR_ROUNDS_BLOCK — included only on round 2+>
In prior rounds:
- Applied: <bullet list of brief descriptions>
- Disagreed-with by the spec author, with their reasoning:
  - <item> — author said: <reasoning>
  - ...
Do not re-raise disagreed items unless the author's reasoning has a real
flaw you can articulate. If you re-raise one, your entry MUST start with
"REBUTTAL:" and explain the flaw in their reasoning, not just re-assert
the original concern.
</PRIOR_ROUNDS_BLOCK>

End your response with exactly one of:
  VERDICT: NO_REMAINING_ISSUES
  VERDICT: ISSUES_REMAIN
```

The `VERDICT:` line is the machine-checkable signal that gates the loop.

## The parent loop algorithm

Pseudocode for the parent agent's behavior, as instructed by `SKILL.md`:

```
applied        = []     # brief descriptions, persist across rounds
disagreements  = []     # {item, reasoning} pairs, persist across rounds
round          = 1
MAX_ROUNDS     = 5      # safety cap

spec_path = resolve_spec(argument)
template  = read("skills/spec-review/reviewer-prompt.md")

loop:
  prompt = template
             .replace("<SPEC_PATH>", spec_path)
             .with_prior_rounds_block(applied, disagreements)   # only if non-empty

  response       = Agent.dispatch(general-purpose, prompt)
  verdict, catches = parse(response)

  if verdict == NO_REMAINING_ISSUES:
    report("clean", round, applied, disagreements)
    exit

  # Decide per catch.
  applied_this_round    = []
  disagreed_this_round  = []
  for catch in catches:
    if I judge it correct, useful, or genuinely uncertain:
      edit spec to address it
      applied_this_round += brief_description(catch)
    else:
      disagreed_this_round += {catch, my_reasoning}

  applied       += applied_this_round
  disagreements += disagreed_this_round

  # Stuck detector: every catch was a REBUTTAL of an existing
  # disagreement AND none of the rebuttals exposed a real flaw
  # in the author's prior reasoning.
  if catches and all(catch is REBUTTAL and rebuttal_not_compelling for catch in catches):
    report("stuck", round, applied, disagreements)
    exit

  if applied_this_round:
    git add <spec_path>
    git commit -m "spec(<topic>): round <round> revisions (applied K, disputed M)"

  round += 1
  if round > MAX_ROUNDS:
    report("cap-hit", round - 1, applied, disagreements, remaining=catches)
    exit
```

### State across rounds

Two lists, both held in the parent agent's conversation context:

- `applied[]`: short bullets describing what was changed and why.
- `disagreements[]`: each entry is `{catch, parent's reasoning}`.
  Re-sent verbatim each subsequent round so the reviewer cannot quietly
  re-raise the same item without acknowledging the prior reasoning.

Nothing is persisted to disk between rounds besides the spec edits
themselves and the per-round commits.

### Stop conditions

| Condition                                          | Final status |
| -------------------------------------------------- | ------------ |
| Reviewer returns `VERDICT: NO_REMAINING_ISSUES`    | `clean`      |
| All new catches are weak REBUTTALs of disagreements | `stuck`      |
| `round > MAX_ROUNDS`                               | `cap-hit`    |

`stuck` is treated as a success — the reviewer has no new material. The
parent surfaces the unresolved items so the user can intervene only if
they want to.

`cap-hit` is the only case that surfaces a remaining-catches block for
human review.

## Final report (printed to terminal)

```
Spec:    docs/superpowers/specs/2026-05-16-spec-review-skill-design.md
Status:  clean | stuck | cap-hit
Rounds:  N
Applied (K):
  - ...
Disputed (M):
  - <catch> — reasoning: <parent's reasoning>
  - ...
Remaining (only if cap-hit):
  - ...
Commits:
  - <sha> spec(<topic>): round 1 revisions ...
  - <sha> spec(<topic>): round 2 revisions ...
```

## Verification of the skill itself

After installing the plugin, sanity-check with:

1. **Convergence test.** Run `/spec-review` against a deliberately weak
   spec (missing edge cases, contradictory requirements). Expect:
   convergence in 2–3 rounds with `clean` or `stuck`, and a `git log`
   showing one commit per applied round.
2. **Push-back test.** Run `/spec-review` against a tight spec where
   the reviewer is likely to over-reach (e.g., suggest sections the
   spec genuinely doesn't need). Expect: `disagreements` > 0 in the
   final report, and the reviewer either accepts the silence or
   produces a `REBUTTAL:` entry — not a silent re-raise.
3. **Spec discovery.** Run `/spec-review` with no argument in a repo
   containing multiple specs; confirm it picks the newest by mtime.
   Run with an explicit path and confirm it overrides discovery.

## Out of scope for this design

- Concurrent reviewers (more than one subagent per round).
- Persisting `applied` / `disagreements` to disk for resumability across
  Claude Code restarts.
- Cross-project shared review history.
- A non-Claude-Code variant (e.g., for raw API or other clients).

These are deferred until the basic loop is in use and the friction
points are real, not hypothetical.

## Open questions

None at design time. Any surprises uncovered during implementation
should be flagged back here in the plan, not silently resolved.
