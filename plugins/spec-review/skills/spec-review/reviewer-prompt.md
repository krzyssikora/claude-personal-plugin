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