# `/spec-review` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `spec-review` Claude Code personal plugin specified at `docs/superpowers/specs/2026-05-16-spec-review-skill-design.md`. After this plan is complete, `/spec-review` is available in every Claude Code project on this machine.

**Architecture:** A single Claude Code personal plugin in the existing repo at `C:\Users\krzys\claude-personal-plugin\`. Four artifacts: a JSON plugin manifest, a slash-command shim, a `SKILL.md` containing the parent-agent loop instructions, and a `reviewer-prompt.md` containing the verbatim template the parent dispatches to each fresh subagent. No executable code — the "logic" is natural-language instructions that the parent agent in a Claude Code session follows at invocation time.

**Tech Stack:** Markdown (for skills and slash commands), JSON (plugin manifest), git (per-round commits and the verification trail), the Claude Code `Agent` tool (subagent dispatch), `Read` / `Edit` / `Write` / `Glob` / `Bash` tools (spec inspection, edits, discovery, git ops).

---

## Pre-flight assumptions

- The plugin repo is initialized at `C:\Users\krzys\claude-personal-plugin\` with the spec at `docs/superpowers/specs/2026-05-16-spec-review-skill-design.md`. (Verified: 11 commits already present.)
- The implementer has Claude Code installed and can install local plugins via `/plugin install <path>`.
- The implementer can read the spec while implementing; nothing in this plan replaces the spec — the plan is the *order* of work, the spec is the *content*.

## TDD adaptation note (important — read before starting)

The spec explicitly defines verification as **manual sanity checks, not regression tests** (see the "Verification (manual sanity checks)" section in the spec). Reasons:

- The artifacts are prompts consumed by a stochastic LLM.
- There is no deterministic input/output to assert on.
- The plugin's contract is satisfied by the LLM's behavior across many runs, not by a single response.

We therefore deviate from strict red-green-refactor TDD. Each build task constructs an artifact and verifies its **shape** (valid JSON, correct frontmatter, presence of required template tokens). **Behavioral verification is concentrated in Tasks 6–9**, each of which runs one of the spec's sanity checks against a freshly installed plugin.

When an artifact is essentially a verbatim block from the spec (the reviewer prompt), the "test" is a `diff` against the spec to confirm the copy is accurate.

## File structure

```
claude-personal-plugin/
├── .claude-plugin/
│   └── plugin.json                     # NEW (Task 1)
├── commands/
│   └── spec-review.md                  # NEW (Task 2)
├── skills/
│   └── spec-review/
│       ├── SKILL.md                    # NEW (Task 4)
│       └── reviewer-prompt.md          # NEW (Task 3)
├── docs/
│   └── superpowers/
│       ├── specs/
│       │   └── 2026-05-16-spec-review-skill-design.md   # already exists
│       └── plans/
│           └── 2026-05-16-spec-review-skill.md          # this file
├── README.md                           # already exists
└── .gitignore                          # already exists
```

Why this decomposition: each file has one clear responsibility — `plugin.json` declares the plugin to Claude Code, `commands/spec-review.md` is the user-facing entry point, `SKILL.md` is the parent agent's instruction set, `reviewer-prompt.md` is the subagent's instruction set. SKILL.md and `reviewer-prompt.md` are kept separate because the parent loads the latter from disk on every round; if they were merged, the parent would need to re-emit half its own context as the subagent's prompt.

---

## Task 1: Plugin manifest

**Files:**
- Create: `.claude-plugin/plugin.json`

The plugin manifest declares the plugin's identity to Claude Code so that `/plugin install` knows what it's loading and so the slash command and skill get discovered.

- [ ] **Step 1: Verify the parent directory exists**

```bash
ls /c/Users/krzys/claude-personal-plugin/.claude-plugin/ 2>&1 || mkdir -p /c/Users/krzys/claude-personal-plugin/.claude-plugin
```

- [ ] **Step 2: Write `plugin.json` with the manifest fields**

```json
{
  "name": "claude-personal-plugin",
  "version": "0.1.0",
  "description": "Personal Claude Code plugin — currently provides /spec-review for iterative spec review via a fresh-context subagent loop.",
  "author": {
    "name": "Krzysztof Sikora",
    "email": "krzyssikora@gmail.com"
  }
}
```

- [ ] **Step 3: Verify the file is valid JSON**

Run: `python -c "import json; json.load(open('/c/Users/krzys/claude-personal-plugin/.claude-plugin/plugin.json'))" && echo OK`
Expected: `OK`

- [ ] **Step 4: Commit**

```bash
cd /c/Users/krzys/claude-personal-plugin
git add .claude-plugin/plugin.json
git commit -m "feat(plugin): add plugin manifest"
```

---

## Task 2: Slash command shim

**Files:**
- Create: `commands/spec-review.md`

The slash command is a thin shim. When the user types `/spec-review [path]`, Claude Code loads this file's body as a prompt for the parent agent in the current session. The body's job is to invoke the `spec-review` skill, forwarding the user's argument.

- [ ] **Step 1: Verify the parent directory**

```bash
mkdir -p /c/Users/krzys/claude-personal-plugin/commands
```

- [ ] **Step 2: Write `commands/spec-review.md`**

```markdown
---
description: Iteratively review a spec via a fresh-context reviewer subagent until it reports no remaining issues (or a safety cap fires). Pass an optional path to the spec; with no path, the most recent file under docs/superpowers/specs/ is used.
argument-hint: "[path-to-spec.md]"
---

Invoke the `spec-review` skill. Pass `$ARGUMENTS` as the `spec` argument — verbatim, no tokenization, no quoting. The skill handles empty-argument discovery and validation.
```

- [ ] **Step 3: Verify the frontmatter is well-formed**

Run: `python -c "import sys; from pathlib import Path; t = Path('/c/Users/krzys/claude-personal-plugin/commands/spec-review.md').read_text(); assert t.startswith('---\n') and t.count('---\n') >= 2, 'malformed frontmatter'; print('OK')"`
Expected: `OK`

- [ ] **Step 4: Commit**

```bash
cd /c/Users/krzys/claude-personal-plugin
git add commands/spec-review.md
git commit -m "feat(command): add /spec-review slash command shim"
```

---

## Task 3: Reviewer prompt template

**Files:**
- Create: `skills/spec-review/reviewer-prompt.md`

This is a **verbatim** copy of the prompt block in the spec's "The reviewer subagent prompt" section (the spec is canonical for this content). Three tokens must survive the copy unchanged so the parent's substitution and assertion logic can locate them: `<SPEC_PATH>`, `<PRIOR_ROUNDS_BLOCK>`, `</PRIOR_ROUNDS_BLOCK>`.

- [ ] **Step 1: Verify the parent directory**

```bash
mkdir -p /c/Users/krzys/claude-personal-plugin/skills/spec-review
```

- [ ] **Step 2: Write `reviewer-prompt.md` with the verbatim prompt content**

The full content (copy exactly):

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

- [ ] **Step 3: Verify the required tokens are present**

Run:
```bash
grep -c '<SPEC_PATH>' /c/Users/krzys/claude-personal-plugin/skills/spec-review/reviewer-prompt.md
grep -c '<PRIOR_ROUNDS_BLOCK>' /c/Users/krzys/claude-personal-plugin/skills/spec-review/reviewer-prompt.md
grep -c '</PRIOR_ROUNDS_BLOCK>' /c/Users/krzys/claude-personal-plugin/skills/spec-review/reviewer-prompt.md
grep -c 'VERDICT: NO_REMAINING_ISSUES' /c/Users/krzys/claude-personal-plugin/skills/spec-review/reviewer-prompt.md
grep -c 'VERDICT: ISSUES_REMAIN' /c/Users/krzys/claude-personal-plugin/skills/spec-review/reviewer-prompt.md
```
Expected: each `grep -c` returns at least `1`. If any returns `0`, the copy is broken — fix it.

- [ ] **Step 4: Commit**

```bash
cd /c/Users/krzys/claude-personal-plugin
git add skills/spec-review/reviewer-prompt.md
git commit -m "feat(skill): add reviewer prompt template"
```

---

## Task 4: SKILL.md — the parent loop

**Files:**
- Create: `skills/spec-review/SKILL.md`

This is the longest task. `SKILL.md` is what the parent agent (the one running in the user's Claude Code session) loads when `/spec-review` is invoked. It must contain the entire loop as natural-language instructions — there is no separate code file. Where the spec's pseudocode uses Python-like syntax, this file uses prose plus concrete tool calls.

Three sub-steps, each producing one section of the file, plus one assembly and one verification.

- [ ] **Step 1: Write the frontmatter and the "When to use" section**

Open `/c/Users/krzys/claude-personal-plugin/skills/spec-review/SKILL.md` and write:

```markdown
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
```

- [ ] **Step 2: Append the "Resolve the target spec" section**

Append to the same file:

```markdown
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
```

- [ ] **Step 3: Append the "Check git preconditions" section**

```markdown
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
```

- [ ] **Step 4: Append "Topic derivation" and "State" sections**

```markdown
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
```

- [ ] **Step 5: Append the "Main loop" section**

```markdown
## Main loop

Repeat until a stop condition fires.

### Step A — Render the reviewer prompt

1. Read the template at `skills/spec-review/reviewer-prompt.md` (use the `Read` tool with the absolute path to this skill directory).

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
  - Remove every entry from `disagreements` (mutating the live list, not D0) for which `overlaps_any(catch, [d])` is True. Capture `removed_count`. If `removed_count == 0`, append to `notes`: `catch <short_id>: compelling rebuttal of no current disagreement; treated as a fresh accepted catch`.
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
```

- [ ] **Step 6: Append the "Render the final report" section**

```markdown
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

Render the `Commits:` section by running `git -C "<repo>" log --reverse --format='%h %s' <start_sha>..HEAD -- "<spec_path>"` and taking each output line as a bullet. If the command returns no lines, render `(none)`.

On `Status: commit-failed`, also render this line immediately above `Commits:`:

```
NOTE: round <round> edits are on disk but were not committed; the Applied (K) count includes them, but git log does not.
```

## Constants — runtime source of truth

- `MAX_ROUNDS = 5` (soft cap, severity-aware — see Step J)
- `HARD_CAP = 20` (hard cap, unconditional)

To change either, edit this file. They are deliberately not runtime-configurable in v1.
```

- [ ] **Step 7: Sanity-check the assembled SKILL.md**

Run:
```bash
wc -l /c/Users/krzys/claude-personal-plugin/skills/spec-review/SKILL.md
grep -c '^## ' /c/Users/krzys/claude-personal-plugin/skills/spec-review/SKILL.md
grep -c 'MAX_ROUNDS' /c/Users/krzys/claude-personal-plugin/skills/spec-review/SKILL.md
grep -c 'HARD_CAP' /c/Users/krzys/claude-personal-plugin/skills/spec-review/SKILL.md
```
Expected: line count > 250, at least 8 `## ` H2 sections, at least 4 mentions each of `MAX_ROUNDS` and `HARD_CAP`. If any is off, you're missing a section — review against the spec.

- [ ] **Step 8: Cross-check against the spec**

Open `docs/superpowers/specs/2026-05-16-spec-review-skill-design.md`. For each of the following spec concepts, confirm it has a corresponding instruction block in `SKILL.md` — fix any gaps inline:

- Spec discovery: argument + no-argument + Glob primitive + error messages
- Skill direct-invocation contract (`spec` argument)
- Git preconditions (in-repo, detached HEAD, in-progress ops, spec clean, no unrelated staged)
- `start_sha` capture
- `topic` derivation
- Pre-dispatch leaked-tag assertion (covers `<PRIOR_ROUNDS_BLOCK>`, `</PRIOR_ROUNDS_BLOCK>`, `<SPEC_PATH>`)
- Reviewer dispatch via `general-purpose`
- Parser rules: verdict line, catch heading shape, REBUTTAL prefix (case-sensitive, exactly one space), severity-word authoritative, contradictory-pair → MALFORMED
- Malformed retry with nudge
- D0 + S0 snapshots
- Per-catch classification (rebuttal-prefixed-with-empty-disagreements note, implicit-rebuttal-via-overlap)
- Compelling rebuttal: remove matching → apply → branches (OK / STALE / INVALID_ANCHOR routes to stale)
- Weak rebuttal: single combined note
- Non-rebuttal: parent_judges_correct_or_uncertain with asymmetric removal rule; INVALID_ANCHOR routes to disagreements
- Commit BEFORE stuck/cap checks
- Hook-modified-spec post-commit diff check
- Stuck classifying note
- Increment, HARD_CAP first, then severity-aware soft cap, then crossing note
- Final report rendering with `(none)` for empty buckets, omitted Notes when empty, Error rendered on five failure statuses

- [ ] **Step 9: Commit**

```bash
cd /c/Users/krzys/claude-personal-plugin
git add skills/spec-review/SKILL.md
git commit -m "feat(skill): add SKILL.md with parent loop instructions"
```

---

## Task 5: Install the plugin locally and verify load

**Files:**
- Modify: `README.md` (optional small update to reflect installed state)

Claude Code's `/plugin install` works against a **marketplace**, not a raw filesystem path. This repo's `.claude-plugin/marketplace.json` declares itself as a single-plugin marketplace (`name: claude-personal-plugin`) listing one plugin (`name: claude-personal-plugin`, `source: "."`).

- [ ] **Step 1: Register the marketplace**

In a Claude Code session (any project; the command is global):

```
/plugin marketplace add C:\Users\krzys\claude-personal-plugin
```

Expected: Claude Code reports the marketplace was added.

- [ ] **Step 2: Install the plugin from the marketplace**

```
/plugin install claude-personal-plugin@claude-personal-plugin
```

(Both names happen to be the same — the first is the plugin name, the second is the marketplace name. The `@` is Claude Code's marketplace syntax.)

Expected: Claude Code reports the plugin installed successfully and lists the available slash commands and skills (`spec-review` should appear in both).

**Alternative for one-shot smoke testing without a persistent install:** start Claude Code with `claude --plugin-dir "C:\Users\krzys\claude-personal-plugin"`. This loads the plugin only for that session.

- [ ] **Step 2: Confirm `/spec-review` is registered**

In a Claude Code session, type `/` and confirm `spec-review` appears in the slash-command palette with the description from `commands/spec-review.md`.

- [ ] **Step 3: Confirm the skill is discoverable**

Ask Claude (in a session): "list available skills". Confirm `spec-review` appears.

- [ ] **Step 4: If load fails, troubleshoot**

If the plugin fails to load, inspect Claude Code's plugin error output. Common causes: malformed `plugin.json` (re-run the JSON validation from Task 1), missing required fields, or a path mismatch. Fix and re-install.

- [ ] **Step 5: Commit any README updates**

If you updated `README.md` to note the install path, commit:

```bash
cd /c/Users/krzys/claude-personal-plugin
git add README.md
git commit -m "docs: note plugin install command in README"
```

---

## Task 6: Sanity check 1 — convergence

**Files:**
- Create (temporary, do not commit): a deliberately weak spec file under `docs/superpowers/specs/` in any existing repo, OR in this plugin repo as `docs/superpowers/specs/2026-05-16-test-weak-spec.md`.

**Goal:** verify the loop converges in 2–3 rounds with `clean` or `stuck` on a spec full of obvious gaps.

- [ ] **Step 1: Create a deliberately weak spec**

In some repo with a `docs/superpowers/specs/` directory, create a file like `2026-05-16-test-weak-spec.md`:

```markdown
# Background queue worker — design

We need a worker that processes queue messages and updates the database. It should also retry on failures.

## Implementation

Use Python. Have it run as a service.
```

This spec is intentionally vague (no queue name, no message schema, no retry policy, no service-management framework named, no database mentioned by type, no error handling, no concurrency model).

- [ ] **Step 2: Run `/spec-review` against it**

In Claude Code, from the same repo:

```
/spec-review docs/superpowers/specs/2026-05-16-test-weak-spec.md
```

- [ ] **Step 3: Observe the run**

Expected:
- The parent emits `Resolved spec: \`<abs path>\``.
- A subagent dispatches; you see its output.
- The parent applies catches (or disputes a few), runs `git commit`, and loops.
- After 2–3 rounds, the loop exits with `Status: clean` or `Status: stuck`.
- `git log <start_sha>..HEAD -- <spec>` shows one commit per applied round.

- [ ] **Step 4: Pass / fail decision**

PASS if the loop terminated with `clean` or `stuck` within 3 rounds AND the commit trail shows one commit per applied round. FAIL otherwise — inspect the notes and the final report to identify which step misbehaved, fix SKILL.md, and rerun.

- [ ] **Step 5: Clean up the test spec**

Delete the temporary file and the commits it produced (interactive — only on a branch you don't need):
```bash
git checkout -- .  # or git reset --hard <start_sha> if a branch is dedicated to this
```

Or just leave them; the test spec is harmless.

---

## Task 7: Sanity check 2 — push-back

**Goal:** verify the parent disputes weak catches and the reviewer accepts the silence (or REBUTTALs once, not silently re-raises).

- [ ] **Step 1: Use the just-implemented plugin's own design doc as the test spec**

The spec at `claude-personal-plugin/docs/superpowers/specs/2026-05-16-spec-review-skill-design.md` is already tight — it survived 10 rounds. Many reviewers will over-reach on it. Run:

```
/spec-review docs/superpowers/specs/2026-05-16-spec-review-skill-design.md
```

from the `claude-personal-plugin` directory.

- [ ] **Step 2: Observe**

Expected:
- The reviewer surfaces some catches (possibly low-severity).
- The parent applies some and disputes at least one.
- The final report shows at least one entry under `Disputed (M)`.
- If the loop continues, the next round either drops the disputed item or emits it as `REBUTTAL: ...` — not as a silent re-raise.

- [ ] **Step 3: Pass / fail decision**

PASS if at least one entry appears under `Disputed`, AND the reviewer did not silently re-raise a disputed item in a later round. FAIL otherwise.

- [ ] **Step 4: Reset the spec to its pre-test state**

The test will probably create new commits on the spec. Reset:

```bash
cd /c/Users/krzys/claude-personal-plugin
git reset --hard <sha-before-test>  # use git reflog to find it
```

**Important:** confirm with the user before running `git reset --hard`. Only do this if the test commits are unwanted.

---

## Task 8: Sanity check 3 — spec discovery

**Goal:** verify `/spec-review` with no argument picks the newest spec by mtime; with an explicit path it overrides discovery.

- [ ] **Step 1: Confirm the plugin repo has at least two specs**

Currently only one (`2026-05-16-spec-review-skill-design.md`). Create a second decoy:

```bash
echo "# decoy spec" > /c/Users/krzys/claude-personal-plugin/docs/superpowers/specs/2026-05-17-decoy.md
touch -d "2026-05-17 10:00" /c/Users/krzys/claude-personal-plugin/docs/superpowers/specs/2026-05-17-decoy.md
```

The decoy's mtime is set to be later than the existing spec.

- [ ] **Step 2: Run `/spec-review` with no argument**

In Claude Code, from `claude-personal-plugin`:

```
/spec-review
```

Expected: the `Resolved spec:` line names `2026-05-17-decoy.md` (the newer one).

- [ ] **Step 3: Stop the run before it actually reviews the decoy**

Send ctrl-c or "stop". You've already verified discovery.

- [ ] **Step 4: Run `/spec-review` with an explicit path**

```
/spec-review docs/superpowers/specs/2026-05-16-spec-review-skill-design.md
```

Expected: the `Resolved spec:` line names the explicit path, regardless of the decoy's newer mtime.

- [ ] **Step 5: Stop the run.**

- [ ] **Step 6: Pass / fail decision**

PASS if both observations match. FAIL otherwise — fix the spec-discovery section of SKILL.md.

- [ ] **Step 7: Delete the decoy**

```bash
rm /c/Users/krzys/claude-personal-plugin/docs/superpowers/specs/2026-05-17-decoy.md
```

---

## Task 9: Sanity check 4 — severity-aware soft cap

**Goal:** verify the loop continues past `MAX_ROUNDS=5` while the latest round still surfaces CRITICAL or IMPORTANT catches, with the crossing note appearing once.

- [ ] **Step 1: Pick a spec rich enough to produce 5+ rounds of CRITICAL/IMPORTANT material**

The plugin's own design doc, fresh (before any review), is rich enough. Reset to the `7800647` commit on a throwaway branch:

```bash
cd /c/Users/krzys/claude-personal-plugin
git checkout -b spec-cap-test 7800647
```

- [ ] **Step 2: Run `/spec-review`**

```
/spec-review docs/superpowers/specs/2026-05-16-spec-review-skill-design.md
```

- [ ] **Step 3: Watch for the crossing event**

After round 5 completes, expect:
- The loop continues into round 6 (no `cap-hit` exit).
- The `Notes:` section in the eventual final report contains exactly one line that starts with `crossed soft cap (MAX_ROUNDS=5) because...`.
- The loop either exits `clean`, `stuck`, or `cap-hit` (on an eventually all-MINOR round), or hits `hard-cap` at round 20 (unlikely on this spec — convergence usually happens between rounds 8–12).

- [ ] **Step 4: Pass / fail decision**

PASS if the crossing note appears exactly once AND the loop terminates before round 21 AND the final status is one of `clean` / `stuck` / `cap-hit` / `hard-cap`. FAIL otherwise.

- [ ] **Step 5: Discard the throwaway branch**

```bash
cd /c/Users/krzys/claude-personal-plugin
git checkout master
git branch -D spec-cap-test
```

---

## Task 10: Final commit and tag

- [ ] **Step 1: Verify the plan and all artifacts are committed**

```bash
cd /c/Users/krzys/claude-personal-plugin
git status
```
Expected: clean working tree.

- [ ] **Step 2: Tag the v0.1.0 release**

```bash
cd /c/Users/krzys/claude-personal-plugin
git tag -a v0.1.0 -m "v0.1.0 — /spec-review skill, first usable version"
```

- [ ] **Step 3: Confirm tag**

```bash
git tag --list
```
Expected: `v0.1.0` appears.

---

## Spec-coverage cross-check (run after Task 4 is complete)

Confirm each requirement in the spec maps to at least one task above:

| Spec section / concept | Task that implements |
|------------------------|----------------------|
| Plugin layout (file tree) | Task 1, 2, 3, 4 |
| Subagent prerequisites | Task 4 (Step B, Step E) |
| Spec discovery (Glob, mtime, errors) | Task 4 (Step "Resolve the target spec") |
| Skill direct-invocation contract | Task 4 (frontmatter) |
| Reviewer subagent prompt + template substitution | Task 3 (prompt), Task 4 (substitution) |
| Pre-dispatch leaked-tag assertion | Task 4 (Step A.4) |
| Parent loop algorithm | Task 4 (Step A–J) |
| Parser output schema | Task 4 (Step C) |
| Rubrics (apply_edit, brief_description, parent_judges, rebuttal_is_compelling, overlaps_any, remove_matching_disagreement) | Task 4 (Steps E, F, G) |
| Report rendering | Task 4 ("Render the final report") |
| Topic derivation | Task 4 ("Topic derivation") |
| Severity tiers (dual role) | Task 4 (Step J) |
| State across rounds | Task 4 ("State maintained across rounds") |
| Git preconditions + failure handling | Task 4 ("Check git preconditions"; Step H hook check) |
| Stop conditions (10 statuses) | Task 4 (Steps D, H, I, J) + report Error rendering |
| Verification sanity checks | Tasks 6–9 |

If any row's task does not in fact cover its concept, return to Task 4 and fix.

---

## Execution choices

After Task 4 is complete and SKILL.md cross-checks against the spec, the implementer may proceed in either of two modes:

1. **Subagent-driven (recommended for parallel tasks).** Dispatch each task to a fresh subagent via `superpowers:subagent-driven-development`. Tasks 1–4 are file-creation tasks; subagents handle them well. Tasks 5–9 require an interactive Claude Code session for the manual sanity checks, so they should be done by the user (or a session-bound parent agent).

2. **Inline execution.** Run all tasks in the current session via `superpowers:executing-plans`, pausing between tasks for review.
