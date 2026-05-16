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

| Decision                | Choice                                                                                                |
| ----------------------- | ----------------------------------------------------------------------------------------------------- |
| Automation level        | Full autonomous loop until clean verdict or safety cap.                                               |
| Push-back rule          | Parent decides per catch: apply if correct or uncertain; push back with reasoning if weak.            |
| Distribution            | Personal Claude Code plugin in its own git repo, installed once per machine.                          |
| Entry point             | Slash command `/spec-review [path]` that invokes the skill; skill is also invocable via Skill tool.   |
| Inter-round persistence | Lives in the parent agent's conversation context, not on disk. Per-invocation scope.                  |
| Per-round commit        | Yes — each round that applies catches creates one git commit, so `git log` is the trail.              |
| Safety cap              | `MAX_ROUNDS = 5`. Not configurable in v1; runtime value lives in `SKILL.md`.                          |

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

## Subagent prerequisites

Round 1 dispatch and every subsequent round share these prerequisites,
which the parent enforces before calling `Agent`:

- **Subagent type:** `general-purpose`. Chosen because the reviewer's
  work is pure analysis with no specialized tool needs; `general-purpose`
  is the lowest-overhead type that still has filesystem read access.
- **Spec is read by the subagent, not embedded in the prompt.** The
  prompt names the absolute spec path; the subagent uses `Read` to
  fetch its contents. This keeps the dispatch payload small and avoids
  any drift between "what the parent sent" and "what the spec actually
  says on disk" between rounds.
- **`<SPEC_PATH>` is always an absolute path.** The parent resolves the
  spec to an absolute path before substitution. A relative path in the
  prompt is a bug. On Windows the path uses the native form (drive
  letter and backslashes); `Read` accepts either slash style, so the
  parent does not normalize.
- **Tools the subagent needs:** `Read` (to fetch the spec). The
  `general-purpose` subagent type already has `Read` among its default
  tools; the `Agent` tool does not accept a per-call tool allowlist, so
  the dispatch passes no tool-restriction argument.

## Invocation flow

1. User types `/spec-review` (or `/spec-review path/to/spec.md`).
2. The slash command body invokes the `spec-review` skill, forwarding
   `$ARGUMENTS` as the `spec` argument.
3. The skill instructs the parent agent to:
   1. **Resolve the target spec** (see "Spec discovery" below).
   2. Load `reviewer-prompt.md` from the skill directory.
   3. Run the loop (below) until a stop condition fires.
   4. Print the final report.

### Spec discovery

- **Argument given.** Treated literally as a single path (no glob
  expansion). Resolved relative to the parent's current working
  directory if not absolute; promoted to absolute before dispatch.
  Validation rules:
  - exactly one argument; more than one → error
    `"/spec-review accepts at most one argument; got N"`.
  - path must exist → otherwise error
    `"spec not found: <path>"`.
  - path must point to a regular file (not a directory or symlink to
    a directory) → otherwise error
    `"spec path must be a file: <path>"`.
- **No argument.** Recursively find `*.md` files under
  `<cwd>/docs/superpowers/specs/`, follow symlinks to files but not
  directories, and pick the one with the newest mtime. On mtime ties,
  pick the lexicographically largest filename; under the
  `YYYY-MM-DD-<slug>.md` convention this picks the most recently dated
  spec, and beyond that the longest-/last-alphabetical slug. Errors:
  - directory missing → `"no specs directory at <cwd>/docs/superpowers/specs/; pass a path explicitly"`.
  - directory exists but contains no `*.md` → `"no specs found under <cwd>/docs/superpowers/specs/; pass a path explicitly"`.

### Skill direct-invocation contract

When invoked via the `Skill` tool rather than the slash command (e.g.,
right after `brainstorming` writes a fresh spec), the skill accepts
exactly one argument:

- `spec` (optional, string): same semantics as the slash command's
  positional argument. If omitted, the skill applies the "no argument"
  discovery rule above.

The slash command is a thin shim: it forwards `$ARGUMENTS` verbatim as
the `spec` argument.

## The reviewer subagent prompt

Stored at `skills/spec-review/reviewer-prompt.md`. Verbatim contents:

```
You are reviewing a spec at <SPEC_PATH>. You will NOT make any changes —
your only job is to find issues clearly and in detail.

Read the spec fully via the Read tool. Then return a structured review.

Use the following severity tiers as a triage aid for the human reader.
They DO NOT change the loop's behavior — they help the author calibrate
which catches matter most.

CRITICAL — blocks correct implementation:
  missing requirements, contradictions, undefined behavior at boundaries,
  unstated dependencies, "magic" steps with no explanation.

IMPORTANT — would make implementation noticeably worse or error-prone:
  vague requirements, missed edge cases, ambiguous interfaces,
  invariants that aren't pinned down, weak test guidance.

MINOR — refinements that would make the spec clearer or implementation
  easier.

FORMAT — each catch MUST be a discrete item using a level-3 heading,
in this exact shape:

### <SEVERITY>. <short-id>: <one-line title>
**Where:** <section / quote from the spec>
**What's wrong:** <one or two sentences>
**Address:** <one or two sentences on what specifically should change>

Use sequential short-ids per severity (C1, C2, ...; I1, I2, ...; M1, M2,
...). Do not nest catches; do not combine multiple issues into one item.

<PRIOR_ROUNDS_BLOCK>
In prior rounds:
- Applied: <bullet list of brief descriptions>
- Disagreed-with by the spec author, with their reasoning:
  - <item> — author said: <reasoning>
  - ...
Do not re-raise disagreed items unless the author's reasoning has a
real flaw you can articulate. If you re-raise one, the entry's heading
MUST have the form

  ### <SEVERITY>. <short-id>: REBUTTAL: <one-line title>

(i.e. the literal "REBUTTAL: " token appears between the short-id colon
and the title), and the body must explain the flaw in the author's
reasoning, not just re-assert the original concern.
</PRIOR_ROUNDS_BLOCK>

End your response with exactly one of the following lines, as the LAST
non-empty line of the response (no trailing prose, no code fences):

  VERDICT: NO_REMAINING_ISSUES
  VERDICT: ISSUES_REMAIN
```

**Template-substitution rules** (the parent obeys these before dispatch):

- On round 1: the parent removes the entire block delimited by
  `<PRIOR_ROUNDS_BLOCK>` and `</PRIOR_ROUNDS_BLOCK>`, **including the
  delimiter lines themselves and the blank line that follows the
  closing delimiter**. The subagent must not see those XML-like tags.
- On round 2+: the parent replaces the delimited block (delimiters
  included) with the rendered prior-rounds content. No tags remain.
- `<SPEC_PATH>` is always replaced with the absolute spec path.

## The parent loop algorithm

Pseudocode for the parent agent's behavior, as instructed by `SKILL.md`:

```
applied       = []   # brief descriptions, persist across rounds
disagreements = []   # {catch, parent's reasoning} pairs, persist across rounds
stale         = []   # catches whose Edit failed because the anchor was
                     # already changed by an earlier catch this round
round         = 1
MAX_ROUNDS    = 5

assert_git_preconditions()           # see "Git preconditions" below
spec_path = resolve_spec(argument)   # absolute, see "Spec discovery"
template  = read("skills/spec-review/reviewer-prompt.md")

loop:
  prompt   = render_template(template, spec_path, applied, disagreements, round)
  response = Agent.dispatch(general-purpose, prompt)
  parsed   = parse(response)

  # Single retry on malformed output, with a correction nudge.
  if parsed == MALFORMED:
    nudged   = prompt + "\n\nYour previous response did not match the " \
                        "required format. Re-emit your review in the "  \
                        "exact format described above."
    response = Agent.dispatch(general-purpose, nudged)
    parsed   = parse(response)
    if parsed == MALFORMED:
      report("malformed", round, applied, disagreements)
      exit

  verdict, catches = parsed

  if verdict == NO_REMAINING_ISSUES:
    report("clean", round, applied, disagreements)
    exit

  # Normalize: a catch is treated as a rebuttal if its title carries the
  # explicit "REBUTTAL: " prefix OR its title/body substantially restates
  # any prior disagreement. Round-1 catches always overlap an empty list
  # and so are never rebuttals — a stray "REBUTTAL: " prefix on round 1
  # is therefore stripped silently and the catch is processed as new.
  for catch in catches:
    catch.is_rebuttal = (catch.title.startswith("REBUTTAL: ")
                         and disagreements != []) \
                        or overlaps_any(catch, disagreements)

  # Decide per catch.
  applied_this_round   = []
  disagreed_this_round = []
  stale_this_round     = []

  for catch in catches:
    if catch.is_rebuttal:
      if rebuttal_is_compelling(catch, disagreements):
        result = apply_edit(catch)
        if result == OK:
          applied_this_round += brief_description(catch)
          remove_matching_disagreement(catch, disagreements)
        else:  # STALE
          stale_this_round += catch
      else:
        # weak rebuttal — keep the existing disagreement, do not re-add
        pass
    else:
      if parent_judges_catch_correct_or_uncertain(catch):
        result = apply_edit(catch)
        if result == OK:
          applied_this_round += brief_description(catch)
        else:  # STALE
          stale_this_round += catch
      else:
        disagreed_this_round += {catch, parent's reasoning}

  applied       += applied_this_round
  disagreements += disagreed_this_round
  stale         += stale_this_round

  # Commit BEFORE the stuck check so a stuck-exit never leaves applied
  # edits uncommitted.
  if applied_this_round:
    try:
      git add <spec_path>
      git commit -m "spec(<topic>): round <round> revisions " \
                  "(applied K, disputed M, stale S)"
    except CommitFailed as e:
      report("commit-failed", round, applied, disagreements, stale, error=e)
      exit

  # Stuck detector: every catch this round was a rebuttal and none were
  # compelling — applied_this_round is empty AND every catch was a
  # rebuttal. Stale catches do not block stuck (they were rebuttals that
  # the parent agreed with but couldn't apply mechanically).
  if catches and applied_this_round == [] \
     and all(catch.is_rebuttal for catch in catches):
    report("stuck", round, applied, disagreements, stale)
    exit

  round += 1
  if round > MAX_ROUNDS:
    report("cap-hit", round - 1, applied, disagreements, stale, remaining=catches)
    exit
```

### Rubrics

`apply_edit(catch)` → Uses `Edit` (or `Write` for whole-section
rewrites) to modify the spec to address `catch`. Edits within a round
are applied sequentially in the order the reviewer listed the catches.
Returns `OK` on success, `STALE` if the `Edit` fails because the anchor
text has already been modified by a prior catch in this round. `STALE`
catches are not added to `applied[]` or `disagreements[]`; they go to
`stale[]` and are surfaced in the final report so the author can
re-address them on the next round (where the reviewer is likely to
re-raise them) or by hand.

`brief_description(catch)` → A one-line summary in the spec author's
own words, ≤ 100 characters, focused on the change applied rather than
the catch's original phrasing. Example: catch C1 "subagent file access
undefined" → `"specified subagent reads spec via Read; <SPEC_PATH> is absolute"`.

`parent_judges_catch_correct_or_uncertain(catch)` → True when the parent
either agrees the catch identifies a real gap, or is uncertain whether
the gap is real but believes addressing it will not harm the spec.
False only when the parent has positive reason to believe the catch is
wrong (factually, scope-wise, or rests on a misreading).

`rebuttal_is_compelling(catch, disagreements)` → True when the rebuttal
points to a new fact, new consequence, or a logical flaw in the
disagreement's reasoning. Re-asserting the original concern in stronger
language is NOT compelling. Returns `False` if `disagreements` is empty
(degenerate case; a round-1 catch with a stray `REBUTTAL: ` prefix has
already been stripped to a normal catch upstream, so this branch is
defensive).

`overlaps_any(catch, disagreements)` → True when the catch's title or
body substantially restates any disagreement. The parent makes this
judgment by reading both texts; no string-similarity threshold is
imposed. Returns `False` on an empty `disagreements` list.

`remove_matching_disagreement(catch, disagreements)` → Removes every
entry `d` from `disagreements` for which `overlaps_any(catch, [d])`
returns True. If zero entries are removed (because the rebuttal flagged
a catch without an underlying disagreement), the catch is treated as a
normal new catch on this round and a one-line note is logged for the
final report.

### `<topic>` derivation for commit messages

The slug used in `spec(<topic>): ...` is the spec filename stem with
any leading `YYYY-MM-DD-` prefix stripped. Example:
`2026-05-16-spec-review-skill-design.md` → topic `spec-review-skill-design`.
If the filename has no date prefix, the full stem is used.

### Severity tiers

Severity (CRITICAL / IMPORTANT / MINOR) is a human-readability aid: it
helps the author calibrate which catches matter most when watching the
report, and it informs (but does not determine) the parent's apply /
dispute decisions. The loop's stop conditions and counting are
severity-agnostic. The reviewer is asked to triage anyway because the
discipline of triaging is itself a useful prompt for finding issues.

### State across rounds

Three lists, all held in the parent agent's conversation context for
the duration of one `/spec-review` invocation only. Re-initialized to
empty at the start of every invocation, so two consecutive invocations
on different specs in the same session do not bleed state.

- `applied[]`: short bullets describing what was changed and why.
- `disagreements[]`: each entry is `{catch, parent's reasoning}`.
  Re-sent verbatim each subsequent round so the reviewer cannot quietly
  re-raise the same item without acknowledging the prior reasoning.
- `stale[]`: catches whose `Edit` failed because the anchor was already
  changed by an earlier catch in the same round. Surfaced in the final
  report; not re-sent to the reviewer (they'll likely re-raise organically).

Nothing is persisted to disk between rounds besides the spec edits
themselves and the per-round commits.

## Git preconditions and failure handling

Before the first dispatch the parent calls `assert_git_preconditions()`:

- If the resolved spec is not inside a git working tree → abort with
  `"spec is not inside a git repository; per-round commits cannot be created"`.
- If the working tree has staged changes touching files OTHER than the
  spec → abort with
  `"unrelated staged changes present; commit or stash them before running /spec-review"`.
- Unstaged changes to files other than the spec are allowed (the parent
  only `git add`s the spec file itself).

On commit failure during the loop (hook rejection, signing failure,
detached HEAD, hook that modifies the spec file, etc.) the loop stops
with the `commit-failed` status. No retries. Already-applied edits in
the prior rounds remain committed; the most recent round's edits remain
on disk for the user to inspect.

## Stop conditions

| Condition                                                                                                  | Final status    |
| ---------------------------------------------------------------------------------------------------------- | --------------- |
| Reviewer returns `VERDICT: NO_REMAINING_ISSUES`                                                            | `clean`         |
| Every catch this round was a rebuttal and none were compelling (so `applied_this_round` is empty)          | `stuck`         |
| `round > MAX_ROUNDS`                                                                                       | `cap-hit`       |
| Reviewer response is unparseable (missing verdict, missing catches, dispatch error) and one retry also failed | `malformed`   |
| `git commit` failed mid-loop                                                                               | `commit-failed` |

`stuck` is treated as a success — the reviewer has no new material. The
parent surfaces the unresolved items so the user can intervene only if
they want to.

`cap-hit`, `malformed`, and `commit-failed` are the cases that surface
context for human review.

## Verdict parsing

The parent treats the response as parseable iff:

- The last non-empty line, after trimming whitespace, equals exactly
  `VERDICT: NO_REMAINING_ISSUES` or `VERDICT: ISSUES_REMAIN`.
- Every catch is delimited by a `### ` heading of the form
  `### <SEVERITY>. <short-id>: [REBUTTAL: ]<title>`, where `<SEVERITY>`
  is `CRITICAL`, `IMPORTANT`, or `MINOR` (the literal severity tier
  followed by a period), `<short-id>` matches `[CIM]\d+`, and the
  `REBUTTAL: ` prefix on `<title>` is optional. Example with rebuttal:
  `### IMPORTANT. I9: REBUTTAL: memory bloat is unbounded in practice`.

If either condition fails, the response is `MALFORMED`, triggering one
retry with the correction nudge appended to the prompt:
`"Your previous response did not match the required format. Re-emit
your review in the exact format described above."`

## Final report (printed to terminal)

```
Spec:    `docs/superpowers/specs/2026-05-16-spec-review-skill-design.md`
Status:  `clean` | `stuck` | `cap-hit` | `malformed` | `commit-failed`
Rounds:  N
Applied (K):
  - [CRITICAL] `<brief description>`
  - [IMPORTANT] `<brief description>`
  - ...
Disputed (M):
  - [MINOR] `<catch title>` — reasoning: `<parent's reasoning>`
  - ...
Stale (S, only if any edits failed due to a prior catch's edit):
  - [IMPORTANT] `<catch title>`
  - ...
Remaining (only if cap-hit):
  - [CRITICAL] `<catch title>`
  - ...
Commits:
  - `<sha>` spec(`<topic>`): round 1 revisions ...
  - `<sha>` spec(`<topic>`): round 2 revisions ...
```

## Verification (manual sanity checks)

These are sanity checks the author runs once after installing the
plugin. They are not regression tests — there is no fixture corpus and
reviewer behavior is stochastic. The expected outcomes are qualitative.

1. **Convergence sanity check.** Run `/spec-review` against a
   deliberately weak spec (missing edge cases, contradictory
   requirements). Expect: convergence in 2–3 rounds with `clean` or
   `stuck`, and a `git log` showing one commit per applied round.
2. **Push-back sanity check.** Run `/spec-review` against a tight spec
   where the reviewer is likely to over-reach (e.g., suggest sections
   the spec genuinely doesn't need). Expect: at least one entry in
   `Disputed` in the final report, and on subsequent rounds the
   reviewer either accepts the silence or emits a `REBUTTAL: …` entry
   — not a silent re-raise.
3. **Spec discovery sanity check.** Run `/spec-review` with no
   argument in a repo containing multiple specs; confirm the newest by
   mtime is picked. Run with an explicit path and confirm it overrides
   discovery.

## Out of scope for this design

- Concurrent reviewers (more than one subagent per round).
- Persisting `applied` / `disagreements` to disk for resumability
  across Claude Code restarts.
- Cross-project shared review history.
- A non-Claude-Code variant (e.g., for raw API or other clients).
- Runtime configurability of `MAX_ROUNDS`. The runtime value lives in
  `SKILL.md`; references in this design doc (decision table and
  pseudocode) are informational and do not auto-update if you change
  SKILL.md.
- Disagreement-memory bloat mitigation. At a 5-round cap with terse
  disagreement entries, total context overhead is bounded at hundreds
  of tokens. Revisit only if real use surfaces friction.

These are deferred until the basic loop is in use and the friction
points are real, not hypothetical.

## Open questions

Any surprises uncovered during implementation should be flagged back
into this section in the plan that follows, rather than silently
resolved.
