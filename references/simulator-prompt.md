# Simulator Prompt Template

Copy this prompt and use it with any LLM that has your spec documents in context.

## Full Simulator Prompt

```
You are a spec validation simulator. You have been given all specification
documents for [SYSTEM NAME].

Read the following vibe test case. Simulate executing the scenario step by
step against the specs.

For each step:
1. Identify the governing spec document and section
2. Trace the data flow through the system primitives
3. Answer every Q-numbered question by citing the spec

For each question, classify as one of:

- COVERED — The spec answers this clearly. Cite the exact section.
- GAP — The spec is silent. No document addresses this.
- CONFLICT — Two specs give contradictory answers. Cite both.
- AMBIGUITY — The spec addresses this but the answer is unclear or
  could be interpreted multiple ways.

After completing all steps, produce a structured report with:

1. **Per-Step Results**
   For each step, list every question with its classification and evidence.

2. **Gap Summary Table**
   | ID | Gap Description | Severity | Affected Steps | Recommended Fix |
   |----|----------------|----------|----------------|-----------------|

   Severity levels:
   - BLOCKING: Cannot proceed. Spec has no answer. Implementation impossible.
   - DEGRADED: Workaround exists but it's fragile or inelegant.
   - COSMETIC: Missing convenience. Not a correctness issue.

3. **Spec Coverage Heatmap**
   | Spec Doc | Steps Hit | Classification |
   |----------|-----------|---------------|
   List every spec doc. Mark each as: EXERCISED, PARTIALLY EXERCISED, or
   NOT EXERCISED. Note any doc that is never referenced by any step.

4. **Recommended Spec Changes**
   For each gap, recommend: which document to update, what section to add
   or modify, and a one-sentence summary of what the addition should cover.

5. **Cross-Document Consistency Check**
   Flag any cases where two documents make conflicting assumptions about
   the same concept (e.g., the payment spec assumes a 10-minute retry
   window, but the inventory spec releases holds after 5 minutes).

Be thorough. Do not fill in gaps with assumptions. If the spec does not
explicitly define something, classify it as GAP even if the answer seems
"obvious." The purpose of this exercise is to find what the spec does NOT say.
```

## Quick Version (for faster iteration)

```
Given all the spec docs in context, trace through this scenario step by step.
For each step, cite the governing spec. For each question, answer COVERED
(cite section), GAP (no spec addresses this), CONFLICT (two specs disagree),
or AMBIGUITY (spec is unclear). Produce a gap summary table at the end.
```

## Regression Prompt (after spec updates)

```
I previously ran a vibe test that identified these gaps:
[PASTE PREVIOUS GAP SUMMARY]

The following specs have been updated since then:
[LIST UPDATED DOCS]

Re-run the same vibe test case against the updated specs. For each
previously identified gap, classify as:
- RESOLVED: The updated spec now covers this. Cite the new section.
- STILL OPEN: The gap persists despite the update.
- REGRESSED: A previously covered item is now broken by the update.

Also flag any NEW gaps introduced by the spec changes.
```
