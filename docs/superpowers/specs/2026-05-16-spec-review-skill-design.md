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
| Safety cap              | `MAX_ROUNDS = 5` (soft) and `HARD_CAP = 20` (hard). See "Stop conditions" for the severity-aware rule. |

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
  prompt is a bug. The parent passes whatever absolute-path form the
  resolution step returned (typically backslashes from `Glob` on
  Windows, forward-slashes elsewhere); `Read` accepts either style, so
  the parent does not normalize.
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
   2. **Emit the resolved absolute spec path** in the parent's
      assistant-turn response so the path appears in the transcript
      before the first dispatch. The loop does NOT pause for user
      confirmation — emission is for visibility only; the user can
      ctrl-c if the resolution is wrong. The path is wrapped in inline
      backticks (e.g., `` `C:\path\to\spec.md` ``) so the markdown
      renderer doesn't interpret backslashes, underscores, or other
      special characters in the path.
   3. Load `reviewer-prompt.md` from the skill directory.
   4. Run the loop (below) until a stop condition fires.
   5. Emit the final report.

(There is no `print` primitive in a Claude Code session; the parent
"emits" by including text in its assistant turn. The pseudocode below
uses `emit_to_user("...")` as a stand-in.)

### Spec discovery

The parent uses Claude Code's `Glob` tool as the discovery primitive
(its symlink and case-sensitivity semantics are consistent across Linux,
macOS, and Windows).

- **Argument given.** `$ARGUMENTS` is a single string forwarded
  verbatim. The parent **does not tokenize**: the entire trailing
  string (after trimming surrounding whitespace) is treated as one
  path. This makes paths with spaces work without quoting, but it
  also means there is no "multi-argument" error to raise. Resolved
  relative to the parent's cwd if not absolute; promoted to absolute
  before dispatch. The parent's cwd in a Claude Code session is
  whatever Claude Code set, which is usually but not always what the
  user expects — passing an absolute path avoids ambiguity, and the
  resolved path is emitted before dispatch (see Invocation flow).
  Validation rules:
  - trimmed string must be non-empty (treat empty as "no argument").
  - path must exist → otherwise error
    `"spec not found: <path>"`.
  - path must point to a regular file (not a directory) → otherwise
    error `"spec path must be a file: <path>"`.
- **No argument.** Discover via
  `Glob(pattern="docs/superpowers/specs/**/*.md")` from the parent's
  cwd. Claude Code's `Glob` returns paths "sorted by modification
  time"; the parent assumes newest-first (verifying on first use is
  prudent — if the actual order is oldest-first, the parent should
  reverse the result before taking the first). Tie behavior between
  files with identical mtimes follows whatever `Glob` returns and is
  not separately specified; in practice ties are rare, and users with
  ambiguity-sensitive workflows pass an explicit path. Discovery
  does NOT walk upward to a git root; if the user is in a subdirectory
  of a repo whose specs live at the repo root, they must pass an
  explicit path. Errors:
  - the `Glob` call returns zero results → error
    `"no specs found under <cwd>/docs/superpowers/specs/; pass a path explicitly"`.

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

Stored at `skills/spec-review/reviewer-prompt.md`. This prompt is the
sole canonical definition of what each severity tier means; the design
doc's "Severity tiers" subsection only describes the tiers' role in the
loop, not their definitions. Verbatim prompt contents:

```
You are reviewing a spec at <SPEC_PATH>. You will NOT make any changes —
your only job is to find issues clearly and in detail.

Read the spec fully via the Read tool. Then return a structured review.

Use the following severity tiers as a triage aid for the human reader
and to gate the loop's safety cap. The parent loop only continues past
its soft cap as long as CRITICAL or IMPORTANT catches are still being
flagged; mark catches accordingly — don't inflate MINORs to IMPORTANT
to extend the loop, and don't downgrade real CRITICALs to avoid extra
rounds.

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

At least one of `**Where:**`, `**What's wrong:**`, or `**Address:**`
must contain non-empty content. A heading with no body is not a catch.

Use sequential short-ids per severity (C1, C2, ...; I1, I2, ...; M1, M2,
...). Do not nest catches; do not combine multiple issues into one item.

H3 (`### `) headings are reserved for catch headings. If you need to
structure your response with other section headers (a summary header,
a closing paragraph), use H2 (`## `) headings or plain prose. The
parent's parser ignores non-matching H3s without error, but emitting
them risks confusing readers — and emitting ONLY non-matching H3s
alongside `VERDICT: ISSUES_REMAIN` is treated as a malformed response
because there are zero parseable catches.

<PRIOR_ROUNDS_BLOCK>
In prior rounds:
- Applied:
  <bullet list of brief descriptions, or "(none)" if empty>
- Disagreed-with by the spec author, with their reasoning:
  <bullet list of items, or "(none)" if empty>
- Could not be located in the current spec (the parent could not
  mechanically apply these earlier; you may re-raise with a fresh
  anchor quote if they still apply):
  <bullet list of catch titles, or "(none)" if empty>

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
  included) with the rendered prior-rounds content. The blank line
  immediately AFTER `</PRIOR_ROUNDS_BLOCK>` is preserved so the
  rendered content is visually separated from the "End your response
  with..." instruction. (Round 1 strips that trailing blank because
  nothing is rendered in its place; round 2+ preserves it as the
  visual separator — same line, opposite treatment, different cases.)
  Empty sub-lists are rendered as `(none)` so the structure is
  recognizable even when one side has nothing.
- **Rendered-block scaffolding** (round 2+): the parent regenerates
  the full block content — not just the bullet lists. The literal
  scaffolding the parent emits, in order, is:

  ```
  In prior rounds:
  - Applied:
    <bullets, or "(none)">
  - Disagreed-with by the spec author, with their reasoning:
    <bullets, or "(none)">
  - Could not be located in the current spec (the parent could not
    mechanically apply these earlier; you may re-raise with a fresh
    anchor quote if they still apply):
    <bullets, or "(none)">
  ```

  No delimiter tokens appear in the rendered output — only the
  scaffolding above plus rendered bullets.
- **Sub-list bullet formats** (round 2+, two-space-indented under each
  sub-list label):
  - Applied: `- [<severity>] <brief_description>`.
  - Disagreed-with: `- [<catch.severity>] <catch.title> — reasoning: <reasoning>`.
  - Could not be located: `- [<catch.severity>] <prefix><catch.title>; was anchored to: "<catch.where>"; intent: <one-line summary of catch.address or catch.whats_wrong>`. Embedded double quotes in `<catch.where>` are passed through verbatim with no escaping; readers tolerate the slight ambiguity, and the bullet's surrounding double quotes are visually distinct enough.
    where `<prefix>` is the literal string `REBUTTAL: ` when
    `catch.is_rebuttal_prefixed` is true and the empty string
    otherwise. Concrete examples:
    - non-rebuttal stale: `- [IMPORTANT] commit_spec missing -C flag; was anchored to: "git add \\"<spec_path>\\""; intent: scope git ops to the spec's repo`
    - rebuttal stale: `- [IMPORTANT] REBUTTAL: memory bloat is unbounded in practice; was anchored to: "..."; intent: ...`
  These mirror (and intentionally overlap with) the final-report
  rendering rules so the reviewer sees the same information the user
  will see.
- `<SPEC_PATH>` is always replaced with the absolute spec path.

**Pre-dispatch sanity assertion:** after substitution, the parent
asserts the rendered prompt contains none of the literal tokens
`<PRIOR_ROUNDS_BLOCK>`, `</PRIOR_ROUNDS_BLOCK>`, or `<SPEC_PATH>`. A
failed assertion is a parent-side bug; abort with `"template
substitution leaked block tags or the <SPEC_PATH> placeholder into the
rendered prompt"`. Without the `<SPEC_PATH>` check, a substitution
miss would silently send a prompt that tells the subagent to `Read` a
file literally named `<SPEC_PATH>`, wasting a dispatch and producing
a MALFORMED retry instead of a clean `template-bug` exit.

## The parent loop algorithm

Pseudocode for the parent agent's behavior, as instructed by `SKILL.md`.
List operations use `.append(x)` for a single element and `+= [...]` for
list extension. String concatenation with non-string operands (e.g.,
`"round " + round`) is shorthand for the implementing language's
natural form (e.g., `f"round {round}"` in Python). Shell-style
operations are wrapped in helper-function references (`commit_spec`,
`git_diff_quiet`) to remind implementers that they need to be invoked
as actual tool/shell calls with proper quoting and `-C <repo>` so they
target the spec's repository rather than the parent's cwd.

```
applied       = []   # list of {severity, brief_description}
disagreements = []   # list of {catch, reasoning}
stale         = []   # list of catch records (full)
notes         = []   # one-line informational strings; no dedup, no cap
round         = 1
MAX_ROUNDS    = 5    # soft cap; illustrative — runtime source: SKILL.md
HARD_CAP      = 20   # hard cap; illustrative — runtime source: SKILL.md

try:
  spec_path = resolve_spec(argument)
except SpecResolutionError as e:
  # Route through report() with empty buckets for UX consistency with
  # other failure exits (precondition-failed, template-bug, etc.).
  report("resolve-failed", round=0, applied=[], disagreements=[],
         stale=[], notes=[], error=e)
  exit

emit_to_user("Resolved spec: `" + spec_path + "`")   # backtick-wrapped
                                                      # per Invocation flow
try:
  assert_git_preconditions(spec_path)
except PreconditionAborted as e:
  report("precondition-failed", round=0, applied=[], disagreements=[],
         stale=[], notes=[], error=e)
  exit

topic = derive_topic(spec_path)                       # see <topic> derivation
repo      = git_toplevel(spec_path)            # cached for later -C calls
start_sha = git_rev_parse(repo, "HEAD")        # baseline for the
                                               # final-report Commits range
template  = read("skills/spec-review/reviewer-prompt.md")

loop:
  prompt = render_template(template, spec_path, applied, disagreements, stale)
  try:
    assert_no_leaked_tags(prompt)              # raises TemplateBugError
  except TemplateBugError as e:
    report("template-bug", round, applied, disagreements, stale, notes, error=e)
    exit

  try:
    response = Agent.dispatch(general-purpose, prompt)
  except DispatchError as e:
    report("dispatch-error", round, applied, disagreements, stale, notes, error=e)
    exit

  parsed = parse(response)

  # Single retry on malformed output, with a correction nudge. NOTE: the
  # retry is a fresh subagent — no memory of the first attempt. The
  # nudge directs first-attempt compliance on format alone; we do NOT
  # echo the prior malformed response back (would blow up payload on
  # garbage output).
  if parsed == MALFORMED:
    nudged = prompt + "\n\nA previous attempt at this review did not "  \
                      "match the required format. Emit your review in " \
                      "the exact format described above; in particular, "\
                      "the LAST non-empty line must be exactly "         \
                      "'VERDICT: NO_REMAINING_ISSUES' or "               \
                      "'VERDICT: ISSUES_REMAIN'."
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
    # Verdict trumps catches. If the reviewer somehow emitted catches
    # alongside a clean verdict, log them by short_id so the discard is
    # visible — don't act on them.
    if catches:
      notes.append("clean verdict accompanied by " + len(catches) +
                   " catch(es), discarded: " +
                   ", ".join(c.short_id for c in catches))
    report("clean", round, applied, disagreements, stale, notes)
    exit

  # Note stray REBUTTAL: prefixes that target nothing (no current
  # disagreements; common but not exclusive to round 1 — also fires
  # when prior compelling rebuttals removed every disagreement). The
  # parser already stripped the REBUTTAL: prefix from catch.title and
  # set is_rebuttal_prefixed.
  #
  # Implicit-rebuttal classification (overlap-based, no explicit prefix)
  # is logged only AFTER compellingness is decided, to avoid emitting
  # two notes for one weak-rebuttal catch.
  for catch in catches:
    if catch.is_rebuttal_prefixed and disagreements == []:
      notes.append("catch " + catch.short_id + ": REBUTTAL: prefix "
                   "appeared with no current disagreements; treated as "
                   "a new catch")

    catch.is_rebuttal = (catch.is_rebuttal_prefixed and disagreements != []) \
                        or overlaps_any(catch, disagreements)

  # Snapshot the spec AND the disagreements list at the start of the
  # round. S0 anchors STALE vs INVALID_ANCHOR. D0 anchors rebuttal
  # compellingness — without it, an earlier within-round compelling
  # rebuttal would remove the matching disagreement and cause later
  # rebuttals of the same disagreement to be mis-classified as weak.
  S0 = read(spec_path)
  D0 = list(disagreements)   # shallow copy; sufficient — entries are
                             # treated as immutable within a round

  applied_this_round   = []
  disagreed_this_round = []
  stale_this_round     = []

  for catch in catches:
    if catch.is_rebuttal:
      if rebuttal_is_compelling(catch, D0):
        # The parent ACCEPTED the rebuttal logically — remove the prior
        # disagreement up-front, before attempting the edit. If zero
        # entries match, the rebuttal is still treated as accepted, and
        # the caller (here) logs a note.
        removed = remove_matching_disagreement(catch, disagreements)
        if removed == 0:
          notes.append("catch " + catch.short_id + ": compelling "
                       "rebuttal of no current disagreement; treated as "
                       "a fresh accepted catch")

        result = apply_edit(catch, S0)
        if result == OK:
          applied_this_round.append({severity: catch.severity,
                                     brief_description: brief_description(catch)})
        elif result == STALE:
          stale_this_round.append(catch)
        elif result == INVALID_ANCHOR:
          # Accepted-but-couldn't-mechanically-apply. Route to stale[]
          # (semantically: "could not apply") rather than disagreements
          # (semantically: "parent disagrees"). See I5.
          stale_this_round.append(catch)
          notes.append("catch " + catch.short_id + ": compelling "
                       "rebuttal but anchor not located; routed to stale")
      else:
        # Weak rebuttal — the disagreement (in its D0 form) already
        # surfaces this issue. Log a single combined note that also
        # records implicit classification (if applicable) so the user
        # sees one entry per catch instead of two.
        if catch.is_rebuttal_prefixed:
          notes.append("round " + round + ": rebuttal " +
                       catch.short_id + " (explicit prefix) was not "
                       "compelling against the D0 disagreement(s)")
        else:
          notes.append("round " + round + ": catch " + catch.short_id +
                       " classified as implicit rebuttal by overlap; "
                       "judged non-compelling against D0 and dropped")
    else:
      if parent_judges_catch_correct_or_uncertain(catch):
        result = apply_edit(catch, S0)
        if result == OK:
          applied_this_round.append({severity: catch.severity,
                                     brief_description: brief_description(catch)})
        elif result == STALE:
          stale_this_round.append(catch)
        elif result == INVALID_ANCHOR:
          disagreed_this_round.append(
            {catch, reasoning: "could not locate the anchor cited by "
                               "the reviewer; treating as could-not-address"})
      else:
        disagreed_this_round.append({catch, reasoning: <parent's reasoning>})

  applied       += applied_this_round
  disagreements += disagreed_this_round
  stale         += stale_this_round

  # Commit BEFORE the stuck/cap checks so a non-clean exit never leaves
  # applied edits uncommitted.
  if applied_this_round:
    k = len(applied_this_round)
    m = len(disagreed_this_round)
    s = len(stale_this_round)
    try:
      commit_spec(repo, spec_path, topic, round, k, m, s)
      # commit_spec(repo, spec_path, topic, round, k, m, s) runs (with
      # proper path quoting) against the spec's repo, not the parent's
      # cwd. All bracketed names below are interpolations:
      #   git -C "<repo>" add "<spec_path>"
      #   git -C "<repo>" commit -m "spec(<topic>): round <round> revisions (applied <k>, disputed <m>, stale <s>)"
      #
      # After commit, detect a hook that rewrote the spec by comparing
      # the working tree to the just-made commit. A hook that staged
      # its rewrite produces a clean diff (no signal needed); a hook
      # that rewrote without re-staging leaves an unstaged diff.
      if not git_diff_quiet(repo, "HEAD", spec_path):
        notes.append("round " + round + ": a commit hook modified the "
                     "spec but did not re-stage; subsequent rounds "
                     "read the modified content from disk")
    except CommitFailed as e:
      report("commit-failed", round, applied, disagreements, stale, notes, error=e)
      exit

  # Stuck detector: any round with zero forward progress is stuck.
  # Captures (a) all catches were weak rebuttals (dropped to notes) and
  # (b) all new catches were disputed. Severity-blind — if the parent
  # rejected everything, more rounds don't help.
  if catches and applied_this_round == [] and stale_this_round == []:
    # Classifying note so the report makes the round's character clear.
    weak_count     = sum(1 for c in catches if c.is_rebuttal)
    disputed_count = len(disagreed_this_round)
    if weak_count == len(catches):
      kind = "all " + len(catches) + " catches were weak rebuttals"
    elif disputed_count == len(catches):
      kind = "all " + len(catches) + " catches were disputed as fresh"
    else:
      kind = ("mixed: " + weak_count + " weak rebuttals, " +
              disputed_count + " disputed-fresh")
    notes.append("stuck round " + round + ": " + kind)
    report("stuck", round, applied, disagreements, stale, notes)
    exit

  round += 1

  # Hard cap: unconditional safety net for pathological runs.
  if round > HARD_CAP:
    report("hard-cap", round - 1, applied, disagreements, stale, notes)
    exit

  # Soft cap: fires only when the most recent round produced NO
  # CRITICAL or IMPORTANT catches. If the reviewer is still flagging
  # critical/important material, the loop continues past MAX_ROUNDS,
  # adding a one-time note on the crossing.
  last_round_had_severity = any(c.severity in ("CRITICAL", "IMPORTANT")
                                for c in catches)
  if round > MAX_ROUNDS and not last_round_had_severity:
    report("cap-hit", round - 1, applied, disagreements, stale, notes)
    exit
  if round == MAX_ROUNDS + 1 and last_round_had_severity:
    notes.append("crossed soft cap (MAX_ROUNDS=" + MAX_ROUNDS + ") "
                 "because the previous round still surfaced "
                 "CRITICAL/IMPORTANT catches; loop continues until "
                 "either a clean/stuck/cap exit or HARD_CAP=" +
                 HARD_CAP + ")")
```

### Parser output schema

`parse(response)` returns either the sentinel `MALFORMED` or the tuple
`(verdict, catches)` where:

- `verdict` is one of the literal strings `NO_REMAINING_ISSUES` or
  `ISSUES_REMAIN`.
- `catches` is a list of catch records, **ordered in document order**
  (the order the catches appear in the reviewer's response). Each
  record is:

  ```
  {
    severity:              "CRITICAL" | "IMPORTANT" | "MINOR",
    short_id:              "C1" | "I3" | "M2" | ...,    # matches [CIM]\d+
    title:                 string,        # REBUTTAL: prefix removed by parser
    where:                 string,        # body of "**Where:** ..." line;
                                          # "" if the line was absent
    whats_wrong:           string,        # body of "**What's wrong:** ...";
                                          # "" if absent
    address:               string,        # body of "**Address:** ...";
                                          # "" if absent
    is_rebuttal_prefixed:  bool,          # true iff heading carried
                                          # "REBUTTAL: " between short-id
                                          # colon and title
  }
  # (Earlier drafts also carried a body_raw field that captured the
  # full text between a catch heading and the next heading. No rubric
  # or report-rendering rule consumes it, so v1 drops it; if future
  # work needs verbatim catch bodies for debugging or training, add
  # it back.)
  ```

The loop augments each catch record at decision time with one runtime
field — `is_rebuttal: bool` — the parent's classification (explicit
prefix OR overlap-with-a-prior-disagreement). The parser does not
populate it.

Parser permissiveness rules:

- `### ` headings whose content does not match
  `<SEVERITY>. <short-id>: ...` are ignored (not catches). The parser
  does not error on them.
- The heading's severity word is authoritative for classification; the
  short_id letter (`C`/`I`/`M`) is recorded but not cross-checked. A
  mismatched heading like `### IMPORTANT. M3: ...` parses as IMPORTANT
  with short_id `M3`. Loop-gating decisions (severity-aware soft cap,
  Disputed/Stale rendering) use `catch.severity`.
- Inside an otherwise valid catch heading, the `**Where:**`,
  `**What's wrong:**`, and `**Address:**` lines are tolerated as
  missing — but **at least one of the three must contain non-empty
  content** for the catch to count as parseable. A heading with no
  body content is silently skipped (not counted as a catch and not
  causing `MALFORMED`).
- The `REBUTTAL: ` prefix is matched **case-sensitively**, immediately
  after the `<short-id>: ` colon-space, requiring **exactly one space**
  after the literal `REBUTTAL:`. Variants (`Rebuttal:`, `REBUTTAL:`
  with no trailing space, leading whitespace, double space) do NOT
  set `is_rebuttal_prefixed`; they are treated as part of the title.
- The verdict line check is strict: the last non-empty line of the
  response, after trimming whitespace, must equal exactly
  `VERDICT: NO_REMAINING_ISSUES` or `VERDICT: ISSUES_REMAIN`. Anything
  else makes the whole response `MALFORMED`.
- If the response has zero parseable catches AND verdict is
  `ISSUES_REMAIN`, the response is `MALFORMED` (contradictory).

### Rubrics

`apply_edit(catch, S0)` → The parent re-reads the current spec from
disk, locates the change site implied by the catch (using `catch.where`
as the primary anchor, plus `whats_wrong` / `address` as guides), and
**authors its own `old_string`** for the `Edit` tool from the current
spec content. The catch fields are descriptive, not literal anchors —
the reviewer was asked to *quote* in `where` but may have paraphrased
or quoted with formatting drift. For whole-section rewrites the parent
may use `Write` instead. Edits within a round are applied in the order
the parser returned catches (which is document order). Returns:

- `OK` — the `Edit` succeeded.
- `STALE` — `Edit` failed, AND the parent-authored `old_string` is a
  substring of `S0` (the snapshot taken at the start of this round)
  but is no longer present on disk. This means an earlier within-round
  edit clobbered the anchor. Recorded in `stale[]`; surfaced in the
  final report; the prior-rounds block lists it so the reviewer can
  re-quote with a fresh anchor.
- `INVALID_ANCHOR` — `Edit` failed, AND the parent-authored
  `old_string` is NOT a substring of `S0`. The parent could not locate
  the change site from the catch's descriptive fields; the reviewer
  may have hallucinated or quoted with too much drift.

`brief_description(catch)` → A one-line summary in the **parent's**
voice (not the reviewer's wording), focused on the change applied
rather than the catch's original phrasing. Aim for around 100
characters; the parent agent's judgment is the only check (no
mechanical truncation). Example: catch C1 "subagent file access
undefined" → `"specified subagent reads spec via Read; spec path
always absolute"`.

`parent_judges_catch_correct_or_uncertain(catch)` → True when the parent
either agrees the catch identifies a real gap, or is uncertain whether
the gap is real but believes addressing it will not harm the spec.
False only when the parent has positive reason to believe the catch is
wrong (factually, scope-wise, or rests on a misreading).

  **Asymmetric rule for removal-based remediation.** Evaluated against
  the *parent's planned remediation*, not the reviewer's `address`
  suggestion. If the parent's plan would primarily *remove* existing
  spec content rather than add or refine it, the uncertainty branch
  flips: apply only on positive agreement, never on uncertainty. If
  uncertain but the catch can be addressed by adding clarification
  instead of removing content, prefer that path.

  This guard applies only to the non-rebuttal branch. A
  compelling-rebuttal acceptance is treated as positive agreement by
  definition (the parent has been convinced by new facts or a logical
  flaw), so the asymmetric rule is deliberately not consulted there —
  if the rebuttal's remediation removes content, that's the right
  outcome.

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
returns True. Called on the **compelling-rebuttal** branch, before
`apply_edit`, so the prior disagreement is gone whether or not the
edit succeeds mechanically. Returns the number of entries removed —
the caller (not this function) is responsible for noting the
zero-match case.

### Report rendering

`report(status, round, applied, disagreements, stale, notes, error=None)`
renders the final report to the user via `emit_to_user`. The mapping:

- `Spec:` — the absolute spec path (cached at resolve time).
- `Status:` — the literal status string.
- `Rounds:` — the integer `round` argument. Cap-related exits pass
  `round - 1` so the "rounds completed" count matches what the user
  saw happen (the loop body advanced `round` before the cap check).
- `Applied (K):` — bullets rendered from `applied[]`, each as
  `- [<severity>] <brief_description>`.
- `Disputed (M):` — bullets rendered from `disagreements[]`, each as
  `- [<catch.severity>] <catch.title> — reasoning: <reasoning>`. (An
  earlier draft attached a `(rebuttal)` tag here, but in v1 no rebuttal
  path ever lands in `disagreements[]` — compelling rebuttals route to
  `applied[]` or `stale[]`, weak rebuttals route to `notes[]` — so the
  tag was unreachable and has been dropped.)
- `Stale (S):` — bullets from `stale[]`, each as
  `- [<catch.severity>] <catch.title>`, or `(none)` if empty.
- `Notes:` — bullets from `notes[]`, omitted entirely if empty.
- `Commits:` — bullets from `git -C <repo> log --reverse --format='%h %s' <start_sha>..HEAD -- <spec_path>`,
  each as `- <sha> <commit-subject>`, or `(none)` if no commits were
  made. `start_sha` was captured at loop entry; the range is bounded
  to the spec file so unrelated commits in the repo are excluded.
- `Error:` — rendered on `dispatch-error`, `commit-failed`,
  `template-bug`, `precondition-failed`, and `resolve-failed`, as
  `Error: <repr(error)>`. The two pre-loop statuses (`precondition-failed`,
  `resolve-failed`) carry the only actionable info the user has for
  those failures, so the Error line MUST render — without it the user
  sees a status but no recovery hint.

Additional rule: on `commit-failed`, also render a clarifying line
above `Commits:`:

```
NOTE: round <round> edits are on disk but were not committed; the
Applied (K) count includes them, but git log does not.
```

### `<topic>` derivation for commit messages

`derive_topic(spec_path)`: the spec filename stem with any leading
`YYYY-MM-DD-` prefix stripped, then **normalized** to be safe for
conventional-commits scopes: lowercased, runs of non-alphanumeric
characters collapsed to a single hyphen, leading and trailing hyphens
trimmed. If the result is empty (e.g., the spec was named
`2026-05-16.md` with no slug), `derive_topic` falls back to the
un-stripped, normalized stem (so `2026-05-16.md` → topic `2026-05-16`).
Examples:
- `2026-05-16-spec-review-skill-design.md` → `spec-review-skill-design`.
- `Feature Plan v2.md` → `feature-plan-v2`.
- `2026-05-16.md` → `2026-05-16` (fallback path).

### Severity tiers

Severity (CRITICAL / IMPORTANT / MINOR) plays two roles:

1. **Human-readability aid.** It helps the author calibrate which
   catches matter most when watching the report, and it informs (but
   does not determine) the parent's apply / dispute decisions.
2. **Soft-cap gate.** The loop's `cap-hit` exit is severity-aware: the
   soft cap (`MAX_ROUNDS`) fires only when the latest round produced
   no CRITICAL or IMPORTANT catches. As long as the reviewer is still
   flagging critical or important material, the loop continues past
   the soft cap up to `HARD_CAP`.

Stuck and clean exits are severity-agnostic — those reflect parent
behavior (forward progress) and reviewer verdict respectively.

**Canonical definitions** of each tier live in the reviewer prompt
above; do not duplicate them here.

### State across rounds

Four lists, all held in the parent agent's conversation context for
the duration of one `/spec-review` invocation only. Re-initialized to
empty at the start of every invocation, so two consecutive invocations
on different specs in the same session do not bleed state.

- `applied[]`: records of `{severity, brief_description}`.
- `disagreements[]`: records of `{catch, reasoning}`. Re-sent each
  subsequent round in its **current** state — entries removed by
  compelling rebuttals are gone from the next round's prior-rounds
  block; everything else is unchanged. (An earlier draft also
  carried an `accepted_but_stale` flag for a future bucket split; v1
  routes that case to `stale[]` instead, so the field is omitted.)
  **Naming note:** this internal bucket is named `disagreements` in
  state and pseudocode but surfaces externally as `Disputed` in the
  final report header and `disputed` in commit messages. The
  divergence is intentional — "disagreements" describes the data,
  "disputed" reads better in user-facing output — but trace it with
  this note when grepping.
- `stale[]`: full catch records whose `Edit` returned `STALE` (or
  whose compelling-rebuttal-INVALID_ANCHOR case was routed here).
  Surfaced in the final report AND re-sent to the reviewer in the
  prior-rounds block (the third sub-list, "Could not be located in the
  current spec"), so the reviewer can re-quote with a fresh anchor.
- `notes[]`: free-form one-line informational notes. No deduplication;
  no length cap. The `Notes:` section may grow large on pathological
  runs — accepted in v1 (see Out of Scope).
- `start_sha`: the HEAD commit of the spec's repo captured immediately
  after `assert_git_preconditions`. Used to bound the final report's
  `Commits:` listing to commits made within this invocation.

**Bucket disjointness invariant.** Every catch record surfaces in
exactly one of `applied`, `disagreements`, or `stale`. The invariant
applies to **catch records, not logical issues**: a stale catch
re-emitted by the reviewer in a later round counts as a fresh catch
record and can land in any bucket on that round. *Exception:* a
**weak rebuttal** appears in none of these buckets — the prior
disagreement it restates already surfaces the same logical issue; the
weak rebuttal is instead recorded in `notes[]`.

Nothing is persisted to disk between rounds besides the spec edits
themselves and the per-round commits.

## Git preconditions and failure handling

The parent's git operations target the repository that **contains the
spec file**, not the parent's cwd's repository (the two can differ).
The check uses `git -C <dirname(spec_path)> rev-parse --show-toplevel`.

Before the first dispatch the parent calls
`assert_git_preconditions(spec_path)`:

- If `git -C <dirname(spec_path)> rev-parse --show-toplevel` fails →
  abort with
  `"spec is not inside a git repository; per-round commits cannot be created"`.
- If `git -C <repo> rev-parse --abbrev-ref HEAD` returns `HEAD` (detached
  HEAD) → abort with
  `"spec's repository is in detached-HEAD state; check out a branch before running /spec-review"`.
- If any of `rebase-merge` / `rebase-apply` / `MERGE_HEAD` /
  `CHERRY_PICK_HEAD` paths exist under `git -C <repo> rev-parse --git-path <...>`
  → abort with
  `"spec's repository has an in-progress rebase/merge/cherry-pick; finish or abort it first"`.
- If the **spec file itself** has staged or unstaged uncommitted
  changes → abort with
  `"spec has uncommitted changes; commit, stash, or — if the diff is only line-endings on a fresh clone — run 'git add --renormalize <spec>'. The per-round commit trail needs the spec at a known baseline."`.
- If the working tree has staged changes touching files OTHER than the
  spec → abort with
  `"unrelated staged changes present; commit or stash them before running /spec-review"`.
- Unstaged changes to files other than the spec are allowed (the parent
  only `git add`s the spec file itself).

These pre-checks let the loop abort *before* spending a subagent
dispatch when the repo state guarantees commit-failure later.

When `assert_git_preconditions` aborts, the parent emits the bespoke
message via `report("precondition-failed", round=0, applied=[],
disagreements=[], stale=[], notes=[], error=<message>)`. This routes
through the same final-report machinery as other failure exits so the
user gets a consistent UX, with `Rounds: 0` and `(none)` for the
empty buckets and commit list. The "Resolved spec:" preamble line is
also emitted before this precondition check fires, so the user
briefly sees the spec path twice on a precondition failure (once in
the preamble, once in the report's `Spec:` field) — accepted noise.

On commit failure during the loop (hook rejection, signing failure,
etc.) the loop stops with the `commit-failed` status. No retries.
Already-applied edits in prior rounds remain committed; the most
recent round's edits remain on disk for the user to inspect.

**Hooks that rewrite the spec during commit** are a separate case:
the commit may succeed but with content the parent did not author.
The pseudocode handles this by comparing the on-disk spec to the
parent's written content after each commit and logging a note. The
subsequent round's reviewer reads from disk, so the modified content
becomes the new ground truth — not catastrophic, but the note ensures
the divergence is visible in the final report.

**Working-tree cleanliness on exit:** `clean`, `stuck`, `cap-hit`, and
`hard-cap` exits commit every round's applied edits before exiting.
The `malformed` and `dispatch-error` exits return before any edits are
attempted in the failing round, so the working tree contains only
committed changes at exit. The `commit-failed` exit is the only path
that leaves uncommitted edits on disk — by design, so the user can
inspect what would have been committed.

## Stop conditions

| Condition                                                                                                                | Final status     |
| ------------------------------------------------------------------------------------------------------------------------ | ---------------- |
| Reviewer returns `VERDICT: NO_REMAINING_ISSUES`                                                                          | `clean`          |
| `catches != [] and applied_this_round == [] and stale_this_round == []` (zero forward progress in a round)               | `stuck`          |
| `round > MAX_ROUNDS` AND the most recent round produced no CRITICAL or IMPORTANT catches                                 | `cap-hit`        |
| `round > HARD_CAP` (unconditional)                                                                                       | `hard-cap`       |
| `Agent.dispatch` itself raised (transport or platform failure)                                                           | `dispatch-error` |
| Reviewer response parsed as `MALFORMED` twice in a row (initial + single retry with correction nudge)                    | `malformed`      |
| `git commit` failed mid-loop                                                                                             | `commit-failed`  |
| Pre-dispatch leaked-tag assertion failed (parent-side template-substitution bug)                                         | `template-bug`   |
| `assert_git_preconditions` aborted before the loop started                                                               | `precondition-failed` |
| `resolve_spec` aborted because the spec couldn't be located or validated                                                 | `resolve-failed`      |

`stuck` and `cap-hit` are treated as successes — the loop has done as
much as it can productively do. The parent surfaces unresolved items
so the user can intervene only if they want to.

`hard-cap`, `dispatch-error`, `malformed`, `commit-failed`,
`template-bug`, `precondition-failed`, and `resolve-failed` are the
cases that strongly surface for human review. `template-bug`
indicates a parent-side bug (substitution leaked block tags or the
`<SPEC_PATH>` placeholder into the dispatched prompt); the loop
reports it so the user gets a usable error rather than a crash.

## Final report (printed to terminal)

Cumulative counts: `K = len(applied)`, `M = len(disagreements)`,
`S = len(stale)`. These uppercase names appear in the report headers.
Per-round counts (`k`, `m`, `s`) appear in commit messages.

```
Spec:    `<absolute spec path>`
Status:  clean | stuck | cap-hit | hard-cap | malformed | dispatch-error | commit-failed | template-bug | precondition-failed | resolve-failed
Rounds:  N

Applied (K = <count>):
  - [CRITICAL] <brief description>
  - [IMPORTANT] <brief description>
  - ...
  (or `(none)` when K = 0)

Disputed (M = <count>):
  - [MINOR] <catch title> — reasoning: <parent's reasoning>
  - ...
  (or `(none)` when M = 0)

Stale (S = <count>):
  - [IMPORTANT] <catch title>
  - ...
  (or `(none)` when S = 0)

Notes:
  - <one-line note>
  - ...
  (omitted entirely when empty)

Commits:
  - <sha> spec(<topic>): round 1 revisions (applied <k>, disputed <m>, stale <s>)
  - ...
  (or `(none)`)

Error (rendered on dispatch-error / commit-failed / template-bug /
       precondition-failed / resolve-failed):
  <error repr>
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
   where the reviewer is likely to over-reach. Expect: at least one
   entry in `Disputed`, and on subsequent rounds the reviewer either
   accepts the silence or emits a `REBUTTAL: …` entry — not a silent
   re-raise.
3. **Spec discovery sanity check.** Run `/spec-review` with no
   argument; confirm the newest spec by mtime is picked. Run with an
   explicit path and confirm it overrides discovery.
4. **Severity-aware cap sanity check.** Run `/spec-review` against a
   spec rich enough to produce >5 rounds of CRITICAL/IMPORTANT catches.
   Expect: the loop continues past round 5 with a one-line crossing
   note, and only exits with `cap-hit` once the latest round contains
   only MINORs.

## Process notes

Any surprises uncovered during implementation should be flagged into
the implementation plan that follows, not silently resolved.

## Out of scope for this design

- Concurrent reviewers (more than one subagent per round).
- Persisting `applied` / `disagreements` to disk for resumability
  across Claude Code restarts.
- Cross-project shared review history.
- A non-Claude-Code variant (e.g., for raw API or other clients).
- Runtime configurability of `MAX_ROUNDS` and `HARD_CAP`. The runtime
  values live in `SKILL.md` as one-line prose statements (e.g., "Soft
  cap: 5 rounds. Hard cap: 20 rounds."); the design doc's numeric
  literals are illustrative and do not auto-update.
- Disagreement-memory bloat mitigation.
- Notes-section truncation on pathological runs.
- Two-round stuck grace period.
- Echoing the prior malformed response back into the retry prompt.
- Constraining catch-emission order beyond "document order" (parser
  guarantees document order; reviewer is not constrained to top-down
  spec order within a response).
- A separate `accepted_but_stale` bucket in v1. The
  compelling-rebuttal-INVALID_ANCHOR case is routed into `stale[]`
  with a clarifying note; if this proves confusing in practice,
  promote it to its own bucket.

These are deferred until the basic loop is in use and the friction
points are real, not hypothetical.
