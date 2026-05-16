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
  work is pure analysis with no specialized tool needs; among the
  currently-available subagent types it is the lowest-overhead one with
  filesystem read access. If a future Claude Code release ships a more
  appropriate type, swap it in.
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

The parent uses Claude Code's `Glob` tool as the discovery primitive
(its symlink and case-sensitivity semantics are consistent across Linux,
macOS, and Windows).

- **Argument given.** Treated literally as a single path (no glob
  expansion). Resolved relative to the parent's current working
  directory if not absolute; promoted to absolute before dispatch.
  Validation rules:
  - exactly one argument; more than one → error
    `"/spec-review accepts at most one argument; got N"`.
  - path must exist → otherwise error
    `"spec not found: <path>"`.
  - path must point to a regular file (not a directory) → otherwise
    error `"spec path must be a file: <path>"`.
- **No argument.** Discover via
  `Glob(pattern="docs/superpowers/specs/**/*.md")` from the parent's
  cwd, then pick the file with the newest mtime. On mtime ties, pick
  the lexicographically largest path; under the
  `YYYY-MM-DD-<slug>.md` convention this picks the most recently dated
  spec, and beyond that the largest-alphabetical slug. Errors:
  - the `Glob` call returns zero results → error
    `"no specs found under <cwd>/docs/superpowers/specs/; pass a path explicitly"`.
    (`Glob` does not distinguish "directory missing" from "directory
    empty"; the single error message covers both, which is fine for
    this UX.)

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
Do not emit `### ` headings other than catch headings; the parent's
parser will ignore non-matching H3s, but emitting them is undefined
behavior.

<PRIOR_ROUNDS_BLOCK>
In prior rounds:
- Applied:
  <bullet list of brief descriptions, or "(none)" if empty>
- Disagreed-with by the spec author, with their reasoning:
  <bullet list of items, or "(none)" if empty>

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
  included) with the rendered prior-rounds content. Empty sub-lists are
  rendered as `(none)` so the structure is recognizable even when one
  side has nothing.
- `<SPEC_PATH>` is always replaced with the absolute spec path.

**Pre-dispatch sanity assertion:** after substitution, the parent
asserts the rendered prompt contains neither the literal token
`<PRIOR_ROUNDS_BLOCK>` nor `</PRIOR_ROUNDS_BLOCK>`. A failed assertion
is a parent-side bug; abort with `"template substitution leaked block
tags into the rendered prompt"`.

## The parent loop algorithm

Pseudocode for the parent agent's behavior, as instructed by `SKILL.md`:

```
applied       = []   # brief descriptions, persist across rounds
disagreements = []   # {catch, parent's reasoning} pairs, persist across rounds
stale         = []   # catches whose Edit could not be applied within a round
                     # because a prior catch's edit clobbered the anchor
notes         = []   # one-line notes surfaced in the final report
                     # (e.g., zero-match rebuttal removals)
round         = 1
MAX_ROUNDS    = 5    # illustrative; runtime canonical source is SKILL.md

assert_git_preconditions()           # see "Git preconditions" below
spec_path = resolve_spec(argument)   # absolute, see "Spec discovery"
template  = read("skills/spec-review/reviewer-prompt.md")

loop:
  prompt = render_template(template, spec_path, applied, disagreements, round)
  assert_no_leaked_tags(prompt)

  try:
    response = Agent.dispatch(general-purpose, prompt)
  except DispatchError as e:
    report("dispatch-error", round, applied, disagreements, stale, notes, error=e)
    exit

  parsed = parse(response)

  # Single retry on malformed output, with a correction nudge.
  if parsed == MALFORMED:
    nudged = prompt + "\n\nYour previous response did not match the "  \
                      "required format. Re-emit your review in the "    \
                      "exact format described above."
    try:
      response = Agent.dispatch(general-purpose, nudged)
    except DispatchError as e:
      report("dispatch-error", round, applied, disagreements, stale, notes, error=e)
      exit
    parsed = parse(response)
    if parsed == MALFORMED:
      report("malformed", round, applied, disagreements, stale, notes)
      exit

  verdict, catches = parsed     # catches is a list of catch records (see schema)

  if verdict == NO_REMAINING_ISSUES:
    report("clean", round, applied, disagreements, stale, notes)
    exit

  # The parser already stripped any "REBUTTAL: " prefix from catch.title
  # and set catch.is_rebuttal_prefixed accordingly. A catch is treated
  # as a rebuttal if (a) it carried the explicit prefix AND there are
  # priors to rebut, OR (b) the parent judges it substantially restates
  # any prior disagreement.
  for catch in catches:
    catch.is_rebuttal = (catch.is_rebuttal_prefixed and disagreements != []) \
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
          applied_this_round.append(brief_description(catch))
          removed = remove_matching_disagreement(catch, disagreements)
          if removed == 0:
            notes.append("rebuttal '" + catch.short_id +
                         "' matched no disagreement; treated as new applied")
        elif result == STALE:
          stale_this_round.append(catch)
        elif result == INVALID_ANCHOR:
          disagreed_this_round.append(
            {catch, reasoning: "could not locate the anchor cited by "
                               "the reviewer; treating as could-not-address"})
      else:
        # weak rebuttal — keep the existing disagreement, do not re-add
        pass
    else:
      if parent_judges_catch_correct_or_uncertain(catch):
        result = apply_edit(catch)
        if result == OK:
          applied_this_round.append(brief_description(catch))
        elif result == STALE:
          stale_this_round.append(catch)
        elif result == INVALID_ANCHOR:
          disagreed_this_round.append(
            {catch, reasoning: "could not locate the anchor cited by "
                               "the reviewer; treating as could-not-address"})
      else:
        disagreed_this_round.append({catch, parent's reasoning})

  applied       += applied_this_round
  disagreements += disagreed_this_round
  stale         += stale_this_round

  # Commit BEFORE the stuck check so a stuck-exit never leaves applied
  # edits uncommitted. Single -m flag, single string.
  if applied_this_round:
    try:
      git add <spec_path>
      git commit -m "spec(<topic>): round <round> revisions (applied <k>, disputed <m>, stale <s>)"
      # k / m / s are PER-ROUND counts (lowercase). Final-report
      # K / M (uppercase) are CUMULATIVE.
    except CommitFailed as e:
      report("commit-failed", round, applied, disagreements, stale, notes, error=e)
      exit

  # Stuck detector: any round with zero forward progress is stuck. This
  # catches both "all catches were weak rebuttals" AND "all new catches
  # were disputed" — both are stable points where the parent has nothing
  # to apply. Trade-off: if the reviewer would have accepted all
  # disputes on the next round (returning NO_REMAINING_ISSUES), we exit
  # one round early. Acceptable; the user can re-run if curious.
  if catches and applied_this_round == [] and stale_this_round == []:
    report("stuck", round, applied, disagreements, stale, notes)
    exit

  round += 1
  if round > MAX_ROUNDS:
    report("cap-hit", round - 1, applied, disagreements, stale, notes,
           remaining=catches)
    exit
```

### Parser output schema

`parse(response)` returns either the sentinel `MALFORMED` or the tuple
`(verdict, catches)` where:

- `verdict` is one of the literal strings `NO_REMAINING_ISSUES` or
  `ISSUES_REMAIN`.
- `catches` is a list of catch records. Each record is:

  ```
  {
    severity:              "CRITICAL" | "IMPORTANT" | "MINOR",
    short_id:              "C1" | "I3" | "M2" | ...,    # matches [CIM]\d+
    title:                 string,        # REBUTTAL: prefix removed by parser
    where:                 string,        # body of "**Where:** ..." line
    whats_wrong:           string,        # body of "**What's wrong:** ..."
    address:               string,        # body of "**Address:** ..."
    body_raw:              string,        # everything between heading
                                          # and next heading, verbatim
    is_rebuttal_prefixed:  bool,          # true iff heading carried
                                          # "REBUTTAL: " between short-id
                                          # colon and title
  }
  ```

Parser permissiveness rules:

- `### ` headings whose content does not match
  `<SEVERITY>. <short-id>: ...` are ignored (not catches). The parser
  does not error on them.
- Missing `**Where:**` / `**What's wrong:**` / `**Address:**` lines
  inside an otherwise valid catch heading are tolerated; the missing
  field is left empty. The catch still counts.
- The verdict line check is strict: the last non-empty line of the
  response, after trimming whitespace, must equal exactly
  `VERDICT: NO_REMAINING_ISSUES` or `VERDICT: ISSUES_REMAIN`. Anything
  else makes the whole response `MALFORMED`.
- If the response has zero parseable catches AND verdict is
  `ISSUES_REMAIN`, the response is `MALFORMED` (contradictory).

### Rubrics

`apply_edit(catch)` → Uses `Edit` (or `Write` for whole-section
rewrites) to modify the spec to address `catch`. Edits within a round
are applied sequentially in the order the reviewer listed the catches.
Returns:

- `OK` — the edit succeeded.
- `STALE` — `Edit` failed because the anchor text was present on disk
  at the start of the round but has been changed by a prior catch's
  edit in this same round. Recorded in `stale[]`; surfaced in the final
  report; not re-sent to the reviewer (they'll likely re-raise
  organically).
- `INVALID_ANCHOR` — `Edit` failed and the anchor text was never
  present on disk at the start of the round (the reviewer hallucinated
  it or quoted from a stale memory of the spec). Recorded in
  `disagreements[]` with reasoning "could not locate the anchor cited
  by the reviewer; treating as could-not-address" so the reviewer sees
  the parent disagreed on grounds of mechanical infeasibility.

To distinguish `STALE` from `INVALID_ANCHOR`, the parent caches the
spec's content at the start of each round and checks whether the
`Edit`'s `old_string` was present in that snapshot.

`brief_description(catch)` → A one-line summary in the spec author's
own words, ≤ 100 characters, focused on the change applied rather than
the catch's original phrasing. Example: catch C1 "subagent file access
undefined" → `"specified subagent reads spec via Read; <SPEC_PATH> is absolute"`.

`parent_judges_catch_correct_or_uncertain(catch)` → True when the parent
either agrees the catch identifies a real gap, or is uncertain whether
the gap is real but believes addressing it will not harm the spec.
False only when the parent has positive reason to believe the catch is
wrong (factually, scope-wise, or rests on a misreading).

  **Asymmetric rule for removals.** When the catch's remediation
  primarily *removes* existing spec content (rather than adding or
  refining it), the uncertainty branch flips: the parent applies only
  on positive agreement, never on uncertainty. Removing on doubt is
  too easy to regret.

`rebuttal_is_compelling(catch, disagreements)` → True when the rebuttal
points to a new fact, new consequence, or a logical flaw in the
disagreement's reasoning. Re-asserting the original concern in stronger
language is NOT compelling. Returns `False` if `disagreements` is empty
(degenerate; should be unreachable since the rebuttal-path branch is
guarded by `is_rebuttal == True` which requires non-empty disagreements
or an overlap).

`overlaps_any(catch, disagreements)` → True when the catch's title or
body substantially restates any disagreement. The parent makes this
judgment by reading both texts; no string-similarity threshold is
imposed. Returns `False` on an empty `disagreements` list.

`remove_matching_disagreement(catch, disagreements)` → Removes every
entry `d` from `disagreements` for which `overlaps_any(catch, [d])`
returns True. Returns the number of entries removed. If zero, the
catch is treated as a normal applied catch and a one-line note is added
to `notes[]` for the final report.

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

Four lists, all held in the parent agent's conversation context for
the duration of one `/spec-review` invocation only. Re-initialized to
empty at the start of every invocation, so two consecutive invocations
on different specs in the same session do not bleed state.

- `applied[]`: short bullets describing what was changed and why.
- `disagreements[]`: each entry is `{catch, parent's reasoning}`.
  Re-sent verbatim each subsequent round so the reviewer cannot quietly
  re-raise the same item without acknowledging the prior reasoning.
- `stale[]`: catches whose `Edit` returned `STALE` (anchor present at
  start of round but clobbered by an earlier within-round edit).
  Surfaced in the final report; not re-sent to the reviewer.
- `notes[]`: one-line notes the loop wants to surface in the final
  report (currently: zero-match rebuttal removals).

**Bucket disjointness invariant:** every catch surfaces in exactly one
of `applied`, `disagreements`, `stale`, or — on `cap-hit` — `remaining`.
INVALID_ANCHOR catches are routed to `disagreements`. STALE catches go
to `stale`. The parent enforces this invariant; double-counting is a
bug.

Nothing is persisted to disk between rounds besides the spec edits
themselves and the per-round commits.

## Git preconditions and failure handling

Before the first dispatch the parent calls `assert_git_preconditions()`:

- If the resolved spec is not inside a git working tree → abort with
  `"spec is not inside a git repository; per-round commits cannot be created"`.
- If the **spec file itself** has staged or unstaged uncommitted
  changes → abort with
  `"spec has uncommitted changes; commit or stash them before running /spec-review so the per-round commit trail is accurate"`.
  Without this check, the parent's first `git add <spec_path>` plus
  `git commit` would sweep the user's pre-existing in-progress edits
  into "round 1 revisions" and silently corrupt the audit trail.
- If the working tree has staged changes touching files OTHER than the
  spec → abort with
  `"unrelated staged changes present; commit or stash them before running /spec-review"`.
- Unstaged changes to files other than the spec are allowed (the parent
  only `git add`s the spec file itself).

On commit failure during the loop (hook rejection, signing failure,
detached HEAD, hook that modifies the spec file, etc.) the loop stops
with the `commit-failed` status. No retries. Already-applied edits in
prior rounds remain committed; the most recent round's edits remain on
disk for the user to inspect.

## Stop conditions

| Condition                                                                                                          | Final status     |
| ------------------------------------------------------------------------------------------------------------------ | ---------------- |
| Reviewer returns `VERDICT: NO_REMAINING_ISSUES`                                                                    | `clean`          |
| Round produced zero forward progress (`applied_this_round == []` and `stale_this_round == []`) with at least one catch | `stuck`      |
| `round > MAX_ROUNDS`                                                                                               | `cap-hit`        |
| `Agent.dispatch` itself raised (transport or platform failure)                                                     | `dispatch-error` |
| Reviewer response parsed as `MALFORMED` twice in a row (initial + single retry with correction nudge)              | `malformed`      |
| `git commit` failed mid-loop                                                                                       | `commit-failed`  |

`stuck` is treated as a success — the reviewer has no new material the
parent will accept. The parent surfaces the unresolved items so the user
can intervene only if they want to.

`cap-hit`, `malformed`, `dispatch-error`, and `commit-failed` are the
cases that surface context for human review.

## Final report (printed to terminal)

Cumulative totals (`K`, `M`, `S`) are uppercase. Per-round counts in
commit messages (`k`, `m`, `s`) are lowercase. The `Notes:` and
`Remaining:` sections render only when non-empty; the `Stale:` and
`Commits:` sections render `(none)` when empty so their absence is
distinguishable from a structural omission.

```
Spec:    `docs/superpowers/specs/2026-05-16-spec-review-skill-design.md`
Status:  `clean` | `stuck` | `cap-hit` | `malformed` | `dispatch-error` | `commit-failed`
Rounds:  N

Applied (K):
  - [CRITICAL] `<brief description>`
  - [IMPORTANT] `<brief description>`
  - ...

Disputed (M):
  - [MINOR] `<catch title>` — reasoning: `<parent's reasoning>`
  - ...

Stale (S):
  - [IMPORTANT] `<catch title>`
  - ...
  (or `(none)`)

Remaining (only on `cap-hit`):
  - [CRITICAL] `<catch title>`
  - ...

Notes:
  - <one-line note>
  - ...
  (omitted entirely when empty)

Commits:
  - `<sha>` spec(`<topic>`): round 1 revisions (applied k, disputed m, stale s)
  - ...
  (or `(none)`)
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
- Two-round stuck grace period (waiting for the reviewer to potentially
  accept all disputes before declaring stuck). The current single-round
  rule is the simpler default; if `stuck` exits feel premature in
  practice, revisit.

These are deferred until the basic loop is in use and the friction
points are real, not hypothetical.

## Open questions

Any surprises uncovered during implementation should be flagged back
into this section in the plan that follows, rather than silently
resolved.
