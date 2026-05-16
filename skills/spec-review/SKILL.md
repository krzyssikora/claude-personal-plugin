---
name: spec-review
description: Iteratively review a spec at `docs/superpowers/specs/<...>.md` via a fresh-context subagent loop. Apply catches you agree with, push back on weak ones, commit each round, stop on a clean verdict or a safety cap. Invoke after `brainstorming` writes a fresh spec, or on demand via `/spec-review [path]`.
---

# spec-review

## When to use this skill

- Right after the `brainstorming` skill writes a new spec under `docs/superpowers/specs/`.
- When the user runs `/spec-review` (with or without a path argument).
- When you've materially revised an existing spec and want an independent review pass.

Do NOT use this skill to review code, plans, or non-spec markdown. The reviewer prompt is tuned for the spec genre.

## Resolve the target spec

1. Read the `spec` argument the skill was invoked with. The slash command forwards `$ARGUMENTS` verbatim; it is a single string with no tokenization.

2. Trim surrounding whitespace.

3. If the trimmed string is non-empty, treat it as a single path:
   - If not absolute, resolve relative to the current working directory.
   - Promote to an absolute path before any subagent dispatch.
   - If the path does not exist → emit `Status: resolve-failed` with `Error: spec not found: <path>` (see the "Render the final report" section) and stop.
   - If the path is a directory → emit `Status: resolve-failed` with `Error: spec path must be a file: <path>` and stop.

4. If the trimmed string is empty (no argument):
   - Use the `Glob` tool with pattern `docs/superpowers/specs/**/*.md` from the current working directory.
   - `Glob` returns paths sorted by modification time. Assume newest-first; take the first result. If the first result looks older than other results (sanity check via the timestamps shown alongside the Glob output), reverse and take the last.
   - If `Glob` returns zero results → emit `Status: resolve-failed` with `Error: no specs found under <cwd>/docs/superpowers/specs/; pass a path explicitly` and stop.
   - Do NOT walk upward to a git root.

5. Emit the resolved path to the user (your assistant-turn output) as a markdown inline-code line so backslashes and underscores don't get mangled:

   `Resolved spec: \`<absolute-path>\``

6. Continue to the git precondition checks.

## Check git preconditions

All git operations target the **repository containing the spec**, not necessarily the parent's cwd. Compute `repo = git -C <dirname(spec_path)> rev-parse --show-toplevel` and cache it. Use `git -C "<repo>" ...` everywhere below.

Run these checks in order; on any failure, emit `Status: precondition-failed` with the corresponding `Error: ...` message and stop:

1. **In a git repo:** if `git -C <dirname(spec_path)> rev-parse --show-toplevel` fails → `Error: spec is not inside a git repository; per-round commits cannot be created`.

2. **Branch checked out:** if `git -C <repo> rev-parse --abbrev-ref HEAD` returns `HEAD` (detached state) → `Error: spec's repository is in detached-HEAD state; check out a branch before running /spec-review`.

3. **No in-progress operation:** for each of `rebase-merge`, `rebase-apply`, `MERGE_HEAD`, `CHERRY_PICK_HEAD`, check whether `git -C <repo> rev-parse --git-path <name>` resolves to an existing path. If any does → `Error: spec's repository has an in-progress rebase/merge/cherry-pick; finish or abort it first`.

4. **Spec is clean:** if `git -C <repo> diff --quiet -- <spec_path>` fails (working-tree diff) OR `git -C <repo> diff --quiet --cached -- <spec_path>` fails (index diff) → `Error: spec has uncommitted changes; commit, stash, or — if the diff is only line-endings on a fresh clone — run "git add --renormalize <spec>". The per-round commit trail needs the spec at a known baseline.`

5. **No unrelated staged changes:** if `git -C <repo> diff --cached --name-only` lists any file other than `<spec_path>` (relative to repo root) → `Error: unrelated staged changes present; commit or stash them before running /spec-review`.

Unstaged changes to files other than the spec are allowed. The parent only stages the spec file.

After preconditions pass, capture `start_sha = git -C <repo> rev-parse HEAD` (a baseline for the final report's Commits range), and compute `topic` from the spec filename per the "Topic derivation" subsection below.

## Topic derivation

`topic` is used in commit messages as `spec(<topic>): ...`.

1. Take the spec filename stem (filename minus the `.md` extension).
2. If it starts with `YYYY-MM-DD-` (four digits, dash, two digits, dash, two digits, dash), strip that prefix.
3. Normalize: lowercase, collapse runs of non-alphanumeric characters to a single hyphen, trim leading and trailing hyphens.
4. If the result is empty, fall back to the un-stripped, normalized stem.

Examples:
- `2026-05-16-spec-review-skill-design.md` → `spec-review-skill-design`
- `Feature Plan v2.md` → `feature-plan-v2`
- `2026-05-16.md` → `2026-05-16` (fallback)

## State maintained across rounds

These four lists plus the scalars persist for one invocation of the skill. Re-initialize to empty/zero at the start of every invocation; do not carry state across invocations.

- `applied` — records of `{severity, brief_description}`. Each successful per-catch edit appends one entry.
- `disagreements` — records of `{catch, reasoning}`. Catches the parent disputed (or accepted-but-could-not-locate on the non-rebuttal path). The bucket is internally named `disagreements` but surfaces externally as `Disputed` in the report.
- `stale` — full catch records whose Edit failed because a within-round prior edit clobbered the anchor, OR compelling-rebuttal-with-INVALID-ANCHOR cases.
- `notes` — free-form one-line strings (zero-match rebuttal removals, weak-rebuttal dismissals, stray-REBUTTAL prefixes, hook-rewrote-spec detection, etc.).
- `round = 1`
- `MAX_ROUNDS = 5` (soft cap — keep this value here as the single source of truth)
- `HARD_CAP = 20` (hard cap — never exceed)
- `start_sha` (captured above)
- `topic` (computed above)
- `spec_path` (the absolute path)
- `repo` (the git toplevel containing the spec)

## Main loop

Repeat until a stop condition fires.

### Step A — Render the reviewer prompt

1. Read the template at `./reviewer-prompt.md` — Claude Code resolves paths beginning with `./` relative to this skill's installed directory, where `SKILL.md` and `reviewer-prompt.md` live side-by-side.

2. Substitute `<SPEC_PATH>` with the absolute `spec_path`.

3. Handle the `<PRIOR_ROUNDS_BLOCK>` / `</PRIOR_ROUNDS_BLOCK>` block:
   - **Round 1:** remove the entire block, including both delimiter lines AND the single blank line immediately following the closing delimiter.
   - **Round 2+:** replace the delimited block (delimiters included) with the rendered scaffolding below. Preserve the blank line immediately after where `</PRIOR_ROUNDS_BLOCK>` used to be — it is the visual separator between the rendered content and the verdict instruction.

   Rendered scaffolding (round 2+), with sub-list bullets two-space-indented under each label:

   ```
   In prior rounds:
   - Applied:
     - [<severity>] <brief_description>
     - ... (or "(none)" if `applied` is empty)
   - Disagreed-with by the spec author, with their reasoning:
     - [<catch.severity>] <catch.title> — reasoning: <reasoning>
     - ... (or "(none)")
   - Could not be located in the current spec (the parent could not
     mechanically apply these earlier; you may re-raise with a fresh
     anchor quote if they still apply):
     - [<catch.severity>] <prefix><catch.title>; was anchored to: "<catch.where>"; intent: <one-line summary>
     - ... (or "(none)")
   ```

   For the stale sub-list, `<prefix>` is the literal `REBUTTAL: ` when the original catch carried that prefix, empty otherwise. Embedded double quotes in `<catch.where>` are passed through verbatim.

4. **Pre-dispatch sanity assertion:** scan the rendered prompt for the literal strings `<PRIOR_ROUNDS_BLOCK>`, `</PRIOR_ROUNDS_BLOCK>`, `<SPEC_PATH>`. If any is present, this is a parent-side bug — emit `Status: template-bug` with `Error: template substitution leaked block tags or the <SPEC_PATH> placeholder into the rendered prompt` and stop.

### Step B — Dispatch the reviewer

Call the `Agent` tool with `subagent_type=general-purpose` and the rendered prompt. The `general-purpose` subagent already has `Read`; the Agent tool does not accept a per-call tool allowlist, so pass none.

If `Agent` itself raises (transport / platform failure), emit `Status: dispatch-error` with `Error: <repr(error)>` and stop.

### Step C — Parse the response

The parser is strict but permissive in specific places:

1. **Verdict line:** the last non-empty line of the response, after trimming whitespace, must equal exactly `VERDICT: NO_REMAINING_ISSUES` or `VERDICT: ISSUES_REMAIN`. Anything else → MALFORMED.

2. **Catches:** scan all `### ` headings in document order. A heading is a catch iff its content matches `<SEVERITY>. <short-id>: [REBUTTAL: ]<title>` where `<SEVERITY>` is the literal word `CRITICAL`, `IMPORTANT`, or `MINOR` followed by a period and space, and `<short-id>` matches `[CIM]\d+`. Non-matching `### ` headings are ignored without error.

3. **`REBUTTAL: ` prefix:** matched case-sensitively, immediately after `<short-id>: `, with exactly one space after the colon. Variants do not set `is_rebuttal_prefixed`; they remain part of the title.

4. **Catch body:** for each catch heading, capture `where` from a `**Where:** ...` line, `whats_wrong` from `**What's wrong:** ...`, `address` from `**Address:** ...`. Missing lines yield empty strings. At least one of the three must be non-empty for the heading to count as a catch; if all three are empty, the heading is silently skipped.

5. **Severity-vs-short-id mismatch** (e.g., `### IMPORTANT. M3: ...`): the heading's severity word wins; record the short_id as-is.

6. **Contradictory pair:** if the verdict is `ISSUES_REMAIN` AND zero parseable catches were collected → MALFORMED.

If MALFORMED, retry **once** with a correction nudge. Construct a new prompt: the original rendered prompt, plus two newlines, plus:

> A previous attempt at this review did not match the required format. Emit your review in the exact format described above; in particular, the LAST non-empty line must be exactly 'VERDICT: NO_REMAINING_ISSUES' or 'VERDICT: ISSUES_REMAIN'.

Dispatch again. If the retry also dispatches successfully but parses as MALFORMED → emit `Status: malformed` and stop. If the retry's `Agent.dispatch` raises → `Status: dispatch-error`.

### Step D — Clean verdict

If verdict is `NO_REMAINING_ISSUES`:
- If `catches` is non-empty, append to `notes`: `clean verdict accompanied by N catch(es), discarded: <comma-joined short_ids>`.
- Emit `Status: clean` final report and stop.

### Step E — Classify catches (per-round, before applying)

1. Snapshot `D0 = list(disagreements)` (the round-start state).
2. Snapshot `S0 = read(spec_path)` (the round-start spec content as a string).

3. For each `catch` in document order:
   - If `catch.is_rebuttal_prefixed AND disagreements == []`, append to `notes`: `catch <short_id>: REBUTTAL: prefix appeared with no current disagreements; treated as a new catch`.
   - Set `catch.is_rebuttal = (catch.is_rebuttal_prefixed AND disagreements != []) OR overlaps_any(catch, disagreements)`.

`overlaps_any(catch, disagreements)` is a judgment call you make by reading the catch's title and body against each disagreement's catch title and reasoning. True iff the catch substantially restates a prior disagreement, regardless of REBUTTAL prefix. Returns False on an empty list.

### Step F — Decide per catch and apply

Initialize `applied_this_round = []`, `disagreed_this_round = []`, `stale_this_round = []`.

For each `catch` in document order:

**If `catch.is_rebuttal` is True:**

- Decide `rebuttal_is_compelling(catch, D0)`: True iff the rebuttal points to a NEW fact, a NEW consequence, or a logical flaw in the disagreement's reasoning — not merely re-asserting the original concern. Evaluate against the round-start snapshot `D0` so sibling rebuttals earlier in the same round don't poison the check.

- **Compelling:**
  - Call `remove_matching_disagreement(catch, disagreements)` — i.e., remove every entry from `disagreements` (mutating the live list, not D0) for which `overlaps_any(catch, [d])` is True. Capture `removed_count`. If `removed_count == 0`, append to `notes`: `catch <short_id>: compelling rebuttal of no current disagreement; treated as a fresh accepted catch`.
  - Call `apply_edit(catch, S0)` (see below). Branch on result:
    - `OK` → `applied_this_round.append({severity: catch.severity, brief_description: brief_description(catch)})`.
    - `STALE` → `stale_this_round.append(catch)`.
    - `INVALID_ANCHOR` → `stale_this_round.append(catch)` AND append note: `catch <short_id>: compelling rebuttal but anchor not located; routed to stale`. (The parent already accepted the rebuttal logically; "could not apply" is a stale outcome, not a fresh disagreement.)

- **Not compelling (weak rebuttal):**
  - Do NOT add to any bucket — the prior disagreement (in its D0 form) already surfaces this issue.
  - Append a single note covering both classification and outcome:
    - if `catch.is_rebuttal_prefixed`: `round <round>: rebuttal <short_id> (explicit prefix) was not compelling against the D0 disagreement(s)`.
    - else: `round <round>: catch <short_id> classified as implicit rebuttal by overlap; judged non-compelling against D0 and dropped`.

**If `catch.is_rebuttal` is False:**

- Decide `parent_judges_catch_correct_or_uncertain(catch)`:
  - True if you agree the catch identifies a real gap, OR you're uncertain whether the gap is real but addressing it will not harm the spec.
  - False only when you have positive reason to believe the catch is wrong (factually, scope-wise, or a misreading).
  - **Asymmetric rule:** if your planned remediation primarily *removes* spec content rather than adds/refines, flip the uncertainty branch — apply only on positive agreement. (Does not apply to compelling rebuttals; those bypass this rubric.)

- **True:** call `apply_edit(catch, S0)`:
  - `OK` → `applied_this_round.append({severity: catch.severity, brief_description: brief_description(catch)})`.
  - `STALE` → `stale_this_round.append(catch)`.
  - `INVALID_ANCHOR` → `disagreed_this_round.append({catch, reasoning: "could not locate the anchor cited by the reviewer; treating as could-not-address"})`.

- **False:** `disagreed_this_round.append({catch, reasoning: <your one- or two-sentence rationale>})`.

After processing all catches:
- `applied += applied_this_round`
- `disagreements += disagreed_this_round`
- `stale += stale_this_round`

### Step G — `apply_edit(catch, S0)`

1. Read the current on-disk spec (this may differ from `S0` if previous catches in this round have made edits).
2. Locate the change site using `catch.where` as the primary anchor, with `catch.whats_wrong` and `catch.address` as guides. The reviewer was asked to quote in `where`, but may have paraphrased — author your own `old_string` from the current spec content.
3. Call the `Edit` tool (or `Write` for whole-section rewrites).
4. If the `Edit` succeeds → return `OK`.
5. If the `Edit` fails:
   - Check whether your authored `old_string` is a substring of `S0`.
   - Yes → return `STALE` (an earlier within-round edit clobbered the anchor).
   - No → return `INVALID_ANCHOR` (the anchor was never present at round start; the reviewer hallucinated or quoted with drift).

`brief_description(catch)`: a one-line summary in *your* voice (not the reviewer's wording), focused on the change you applied. Aim for ~100 characters. Example: `"specified subagent reads spec via Read; spec path always absolute"`.

### Step H — Commit the round (if any applied edits)

If `applied_this_round` is empty, skip this step.

Otherwise, run via `Bash`:

```
git -C "<repo>" add "<spec_path>"
git -C "<repo>" commit -m "spec(<topic>): round <round> revisions (applied <k>, disputed <m>, stale <s>)"
```

where `k = len(applied_this_round)`, `m = len(disagreed_this_round)`, `s = len(stale_this_round)`. Note lowercase per-round letters; the final report uses uppercase cumulative `K`, `M`, `S`.

If the commit command fails (non-zero exit), emit `Status: commit-failed` with `Error: <stderr or summary>` and stop.

After a successful commit, check whether a hook rewrote the spec without re-staging: run `git -C "<repo>" diff --quiet HEAD -- "<spec_path>"`. If it fails (i.e., there IS a diff), append a note: `round <round>: a commit hook modified the spec but did not re-stage; subsequent rounds read the modified content from disk`. (A hook that staged its rewrite produces no diff and needs no note.)

### Step I — Stuck check

If `catches` is non-empty AND `applied_this_round == []` AND `stale_this_round == []`, the round produced zero forward progress. Build a classifying note:

- `weak = count of catches with is_rebuttal == True`
- `disputed = len(disagreed_this_round)`
- if `weak == len(catches)`: kind = `all N catches were weak rebuttals`
- elif `disputed == len(catches)`: kind = `all N catches were disputed as fresh`
- else: kind = `mixed: W weak rebuttals, D disputed-fresh`

Append `stuck round <round>: <kind>` to `notes`, then emit `Status: stuck` final report and stop.

### Step J — Increment and check caps

1. `round += 1`.
2. If `round > HARD_CAP` → emit `Status: hard-cap` final report with `round - 1` and stop.
3. Compute `last_round_had_severity = any(catch.severity in {CRITICAL, IMPORTANT} for catch in catches)`.
4. If `round > MAX_ROUNDS AND NOT last_round_had_severity` → emit `Status: cap-hit` final report with `round - 1` and stop.
5. If `round == MAX_ROUNDS + 1 AND last_round_had_severity` (one-time crossing) → append note: `crossed soft cap (MAX_ROUNDS=5) because the previous round still surfaced CRITICAL/IMPORTANT catches; loop continues until either a clean/stuck/cap exit or HARD_CAP=20`.

Continue from Step A.

## Render the final report

Emit (to the user's assistant-turn output) a fenced block of the following form. `K = len(applied)`, `M = len(disagreements)`, `S = len(stale)`.

```
Spec:    `<absolute spec path>`
Status:  <status>
Rounds:  <round-or-round-minus-one per Step J>

Applied (K = <K>):
  - [<severity>] <brief_description>
  - ...
  (or `(none)` when K = 0)

Disputed (M = <M>):
  - [<catch.severity>] <catch.title> — reasoning: <reasoning>
  - ...
  (or `(none)` when M = 0)

Stale (S = <S>):
  - [<catch.severity>] <catch.title>
  - ...
  (or `(none)` when S = 0)

Notes:
  - <note>
  - ...
  (omit this section entirely if `notes` is empty)

Commits:
  - <sha> <commit-subject>
  - ...
  (or `(none)`)

Error (only when Status is one of: dispatch-error, commit-failed, template-bug, precondition-failed, resolve-failed):
  <error message>
```

Field explanations:
- `Rounds:` — the integer `round` argument. For `clean` and `stuck` exits, this is the current `round` (un-incremented, equal to the round whose work was just observed). For `cap-hit` and `hard-cap` exits, the loop passes `round - 1` because `round` was incremented in Step J before the cap check, and the user's mental model is "the last round that ran" not "the round we never started."

Render the `Commits:` section by running `git -C "<repo>" log --reverse --format='%h %s' <start_sha>..HEAD -- "<spec_path>"` and taking each output line as a bullet. If the command returns no lines, render `(none)`.

On `Status: commit-failed`, also render this line immediately above `Commits:`:

```
NOTE: round <round> edits are on disk but were not committed; the Applied (K) count includes them, but git log does not.
```

## Constants — runtime source of truth

- `MAX_ROUNDS = 5` (soft cap, severity-aware — see Step J)
- `HARD_CAP = 20` (hard cap, unconditional)

To change either, edit this file. They are deliberately not runtime-configurable in v1.
