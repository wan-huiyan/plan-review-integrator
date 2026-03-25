---
name: plan-review-integrator
version: 1.1.0
author: wan-huiyan
description: >
  Integrate structured review panel findings into an implementation plan document.
  Takes output from agent-review-panel (or any structured review with severity-rated
  findings) and cross-references each finding against the plan, classifies it into
  an action category, applies concrete edits, and produces a traceability summary.
  Trigger when the user says "update the plan with review findings", "incorporate
  review feedback into the plan", "integrate review results", "apply review
  recommendations to the plan", "cross-reference review output against the plan",
  "merge review findings into the implementation plan", "what needs to change in the
  plan based on the review", "take the review panel output and update my plan",
  "reconcile the review feedback with the current plan", or invokes
  /plan-review-integrator. Does NOT trigger for running a review panel (use
  agent-review-panel), writing a plan from scratch, general code review, summarizing
  review findings without applying them, or brainstorming implementation approaches.
consumes_from: agent-review-panel
hands_off_to: implementation-executor
output_contract: >
  Returns: (1) updated plan document with edits applied, (2) traceability summary
  table mapping each finding to its disposition, (3) optional peripheral updates
  (ADRs, runbooks, memory files). Idempotent: re-running on the same review+plan
  produces the same result if no user overrides are applied.
---

# Plan-Review Integrator v1.1

Consumes structured review panel output and integrates findings into an
implementation plan document -- turning review feedback into concrete plan updates
with full traceability.

> **Key insight:** Review panels often identify correct *symptoms* but prescribe
> wrong *fixes* when they lack domain context. Always validate recommendations
> against domain-specific constraints before applying them.

---

## Quick Reference

| Phase | Action | Output |
|-------|--------|--------|
| 1. Gather | Collect review reports + plan + domain context | Input set |
| 2. Extract | Parse findings with severity, source, citations | Structured finding list |
| 3. Cross-ref | Match each finding against plan content | Category per finding |
| 4. Classify | Assign action category | must-fix / bundle / defer / info |
| 5. Apply | Edit plan document | Updated plan |
| 6. Peripherals | Update ADRs, runbooks, memory | Supporting docs |
| 7. Summary | Traceability table | Audit trail |

---

## Phase 1: Gather Inputs

Collect three things:

1. **Review report(s)** -- file path, inline paste, or reference to prior conversation
2. **Plan document** -- markdown plan, design doc, RFC, or architecture proposal
3. **Domain context** -- memory files, config files, related docs, session history

Domain context is essential for validating reviewer recommendations. Do NOT skip it.

---

## Phase 2: Extract Findings

For each finding, capture:

| Field | Description |
|-------|-------------|
| ID | Sequential: R1-F01, R1-F02, ... R2-F01, ... |
| Severity | CRITICAL / HIGH / MEDIUM / LOW |
| Source | Reviewer name, "Completeness Audit", or "Judge" |
| Summary | One-line description |
| Detail | Full context with code snippets/citations |
| Prescribed | Reviewer's recommended fix |
| Consensus | Consensus / disputed / unilateral |
| Location | Plan section, line, or code block |

**Multiple reports:** Deduplicate and merge overlapping findings (note all sources). Flag conflicting recommendations. Keep unique findings separate.

**High-signal items:** Completeness audit findings (systematic gaps), judge rulings (authoritative), items "resolved during debate" (may still need documentation).

---

## Phase 3: Cross-Reference

Categorize each finding's relationship to the plan:

| Category | Meaning |
|----------|---------|
| Already addressed | Plan handles it; reviewer may have missed it |
| Gap | Plan should address this but doesn't |
| Correction | Plan addresses it but contains an error |
| New concern | Affects scope/timeline/approach; may need structural changes |
| Pre-existing | Valid concern, but not introduced or worsened by this plan |

**Domain validation checklist** -- for each finding ask:
- Does the prescribed fix make sense given domain constraints?
- Is the reviewer assuming something untrue about the system?
- Would the fix break something the reviewer doesn't know about?
- Is the concern already mitigated by a mechanism the reviewer didn't see?

Document cases where the finding is valid but the prescribed fix is wrong.

---

## Phase 4: Classify

Assign each finding to one action category:

### Must-fix
CRITICAL/HIGH severity + Correction or Gap + affects correctness/data integrity/security. For example, a wrong SQL WHERE clause or missing temporal guard. Applied immediately.

### Bundle into implementation
Valid items that are implementation details, not plan defects, e.g., pipeline quality gates or additional test cases. Added to checklists.

### Defer
LOW severity, pre-existing debt not worsened by plan, or different workstream. For instance, refactoring code not touched by this plan. Documented with rationale.

### Informational
Raised, debated, and resolved during review. No plan changes needed.

### Conflict resolution
- CRITICAL + pre-existing: defer, but add caveat with risk note and future work item
- Panels disagree on severity: use higher severity, note disagreement
- Valid finding + wrong fix: classify on finding severity, supply correct fix

---

## Phase 5: Apply Edits

| Category | Edit type |
|----------|-----------|
| Must-fix | Direct plan edits: fix code, add guards, correct values |
| Bundle | Add to implementation checklists and verification sections |
| Gap (new section) | Add pre-impl checks, go/no-go criteria, rollback, monitoring |
| Caveat | Inline `> **Review note:** [concern + mitigation]` near relevant content |

**Edit discipline:**
- Minimum necessary change per finding
- Preserve plan voice and structure
- Do not rewrite correct sections
- Link edits to finding IDs for traceability

---

## Phase 6: Update Peripherals

- **Memory files:** Document findings affecting future sessions, new conventions, updated action items
- **ADRs:** Create/update for architectural decisions validated or modified by review
- **Runbooks:** Update deployment/rollback/operational procedures if changed
- **Config:** Note specific files and values to change; add to pre-implementation checklist

---

## Phase 7: Produce Summary

Example output:

| ID | Severity | Summary | Category | Action Taken |
|----|----------|---------|----------|--------------|
| R1-F01 | CRITICAL | Missing temporal guard | Must-fix | Added guard to Phase 2 |
| R1-F02 | HIGH | Stale default in config | Bundle | Added to checklist item 3 |
| R1-F03 | MEDIUM | Refactor identity resolution | Defer | Pre-existing, tracked as future work |

Include statistics (total/must-fix/bundle/defer/info counts) and key decisions (e.g., why a reviewer fix was overridden). Present summary to user and ask if they want to adjust classifications before finalizing.

---

## Composability

This skill expects structured review output as input (requires a file path or inline findings).
Do not use for running review panels or writing plans from scratch.
If integration fails, gracefully degrade by presenting unmodified findings for manual triage.
After integration, then use the implementation executor to begin building.
This skill depends on `agent-review-panel` upstream. Works with review v1.0 output format and above.

| Field | Value |
|-------|-------|
| Consumes from | `agent-review-panel` (or any structured review with severity-rated findings) |
| Hands off to | Implementation executor; updated plan feeds into build phases |
| Output contract | Updated plan + traceability summary + optional ADRs/runbooks |
| Namespace | All edits scoped to provided plan document; no global state modification |
| Idempotency | Re-running on same inputs produces same result (absent user overrides) |

---

## Anti-Patterns

1. **Applying fixes blindly** -- always validate against domain context
2. **CRITICAL != plan defect** -- some are verification steps (add to checklist, not correction)
3. **Ignoring completeness audit** -- highest-signal items; systematic gaps all reviewers missed
4. **Pre-existing debt as must-fix** -- if plan doesn't worsen it, defer with backlog link
5. **Plan as findings dump** -- each finding lands in exactly the right place, not a giant appendix
6. **Skipping user confirmation** -- present classification summary; let user adjust before applying
