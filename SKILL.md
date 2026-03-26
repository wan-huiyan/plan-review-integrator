---
name: plan-review-integrator
version: 1.2.0
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
  (ADRs, runbooks, memory files), (4) integration_log.jsonl append. Idempotent:
  re-running on the same review+plan produces the same result if no user overrides
  are applied.
---

# Plan-Review Integrator v1.2

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
| 3.5. Filter | Score actionability, drop low-signal findings | Filtered finding list |
| 4. Classify | Assign action category (epistemic-weighted) | must-fix / bundle / defer / info |
| 5. Apply | Edit plan document with rollback on coherence break | Updated plan |
| 5.5. Verify | Re-read modified plan, check coherence | Verified plan or rollback |
| 6. Peripherals | Update ADRs, runbooks, memory | Supporting docs |
| 7. Summary | Traceability table | Audit trail |
| 7.5. Log | Append decisions to persistent integration log | integration_log.jsonl |

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

## Phase 3.5: Actionability Filter

> Inspired by Atlassian's RovoDev comment ranker (ICSE 2026, arXiv:2601.01129),
> which found that filtering LLM-generated review comments with a quality predictor
> improved resolution rates from ~33% to 40-45%, approaching human reviewer performance.

Score each finding on two dimensions before classification:

| Dimension | Question | Score |
|-----------|----------|-------|
| **Actionability** | Does this finding identify a specific, objectively verifiable issue with a concrete action? | 0.0 - 1.0 |
| **Groundedness** | Is the finding supported by code citations, line references, or verifiable claims? | 0.0 - 1.0 |

**Filter rules:**
- **Drop** findings with actionability < 0.3 (conversational, acknowledgements, vague concerns)
- **Flag for human review** findings with actionability 0.3-0.5 (valid concern, unclear action)
- **Pass through** findings with actionability >= 0.5 (specific issue, concrete fix path)
- Groundedness < 0.3 caps maximum severity at MEDIUM regardless of original rating

**Epistemic label weighting** (from upstream agent-review-panel):
- `[VERIFIED]` or `[CMD_CONFIRMED]`: +0.2 actionability bonus
- `[CONSENSUS]`: no adjustment
- `[SINGLE-SOURCE]`: -0.1 actionability penalty
- `[UNVERIFIED]` or `[DISPUTED]`: -0.2 actionability penalty

Record the filter decision (pass/flag/drop) and scores for each finding in the traceability table.

---

## Phase 4: Classify

Assign each finding to one action category. Classification uses **epistemic-weighted
severity** -- the effective priority of a finding depends on both its raw severity and
the confidence of the evidence behind it.

> Informed by research on multi-agent debate quality (arXiv:2511.07784, "Can LLM Agents
> Really Debate?") showing that eloquent-but-wrong agents can sway consensus, and that
> debate gains may reduce to ensembling effects. Epistemic labels from upstream reviews
> are the best available signal for evidence quality.

### Epistemic-Weighted Severity

| Raw Severity | Epistemic Label | Effective Priority |
|---|---|---|
| CRITICAL | `[VERIFIED]` / `[CMD_CONFIRMED]` | P0 -- immediate must-fix |
| CRITICAL | `[CONSENSUS]` | P0 -- must-fix with verification step |
| CRITICAL | `[SINGLE-SOURCE]` / `[DISPUTED]` | P1 -- flag for human review before acting |
| HIGH | `[VERIFIED]` | P1 -- must-fix |
| HIGH | `[CONSENSUS]` | P1 -- must-fix or bundle |
| HIGH | `[SINGLE-SOURCE]` / `[DISPUTED]` | P2 -- bundle or defer |
| MEDIUM/LOW | any | P2 -- bundle, defer, or informational |

A `[VERIFIED]` HIGH finding outranks a `[SINGLE-SOURCE]` CRITICAL finding. When in doubt,
present the conflict to the user rather than auto-resolving.

### Must-fix
P0 or P1 effective priority + Correction or Gap + affects correctness/data integrity/security. For example, a wrong SQL WHERE clause or missing temporal guard. Applied immediately.

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

**Rollback on coherence break:**

> Inspired by AutoDW's dual rollback strategy (arXiv:2512.04445), which achieved 90%
> completion on document workflow tasks by validating each state change and reverting
> when edits break document coherence.

After each must-fix edit, re-read the surrounding section (2 paragraphs before and after).
If the edit introduces inconsistency, contradiction, or breaks the logical flow:

1. **Argument-level rollback** (first attempt): Rephrase the edit to fit the surrounding
   context while preserving the finding's intent. Keep the same finding ID linkage.
2. **API-level rollback** (if rephrasing fails): Revert the edit entirely. Add the finding
   to the traceability table with disposition "ROLLBACK -- requires manual integration"
   and include the coherence issue encountered.

Do not force edits that break the plan. A clean plan with a flagged finding is better than
a corrupted plan with a forced fix.

---

## Phase 5.5: Post-Integration Verification

> Inspired by Self-Refine (NeurIPS 2023, arXiv:2303.17651), which demonstrated 20%
> average improvement from generate-critique-refine cycles, and ARIS's auto-review-loop
> (github.com/wanshuiyin/Auto-claude-code-research-in-sleep) which chains cross-model
> review for overnight autonomous quality improvement.

After all Phase 5 edits are applied, perform a single verification pass:

1. **Coherence check** -- Re-read the full updated plan. Flag any section where edits
   from different findings contradict each other or create logical gaps.
2. **Completeness check** -- Verify every must-fix finding has a corresponding edit.
   Every bundle finding appears in a checklist. Every defer finding has a rationale.
3. **Voice check** -- Confirm edits match the plan's existing tone and style. Rewrite
   any edit that reads like injected review commentary rather than plan content.
4. **Cross-reference check** -- If edits reference other plan sections (e.g., "see Phase 3"),
   verify those references are still valid after modifications.

If verification surfaces issues, apply targeted fixes (not a full re-integration).
Record verification findings in the traceability summary as "V-01", "V-02", etc.

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

## Phase 7.5: Persistent Integration Log

> Inspired by pi-autoresearch's append-only `autoresearch.jsonl` experiment log
> (github.com/davebcn87/pi-autoresearch), which enables fresh agent sessions to resume
> exactly where previous sessions stopped and learn from prior optimization decisions.

Append one JSON line per finding to `integration_log.jsonl` in the project root:

```json
{
  "timestamp": "2026-03-26T14:30:00Z",
  "plan": "tasks/implementation_plan.md",
  "review_source": "review_panel_report.md",
  "finding_id": "R1-F01",
  "severity": "CRITICAL",
  "epistemic_label": "[VERIFIED]",
  "effective_priority": "P0",
  "actionability_score": 0.85,
  "category": "must-fix",
  "disposition": "applied",
  "rollback": false,
  "verification_passed": true,
  "notes": "Added temporal guard to Phase 2 query"
}
```

**Why:** Over multiple integration runs, this log reveals patterns -- which finding types
are consistently deferred, which sources produce the most actionable findings, and which
severity levels correlate with actual plan changes. Future runs can use this history to
calibrate actionability scoring.

If `integration_log.jsonl` already exists, read it before Phase 3.5 to inform actionability
scoring: findings matching historically-deferred patterns get a -0.1 penalty; findings
matching historically-applied patterns get a +0.1 bonus.

---

## Upstream Schema Contract

> This section defines the expected input format from `agent-review-panel` to prevent
> silent misparse when the upstream skill evolves. Version-pinned for compatibility.

**Compatible with:** `agent-review-panel` v2.0+

**Required fields per finding:**
- Severity tier: `P0` / `P1` / `P2` (maps to CRITICAL / HIGH / MEDIUM-LOW)
- Epistemic label: one of `[VERIFIED]`, `[CMD_CONFIRMED]`, `[CONSENSUS]`, `[SINGLE-SOURCE]`, `[UNVERIFIED]`, `[DISPUTED]`
- Defect type: `[EXISTING_DEFECT]` or `[PLAN_RISK]`
- Source attribution: reviewer persona name or "Completeness Audit" or "Judge"

**Required report sections:**
- `## Action Items` -- tagged findings with severity + epistemic labels
- `## Consensus Points` -- agreed findings (can be processed as `[CONSENSUS]`)
- `## Disagreement Points` -- with Side A, Side B, Judge's ruling

**Optional but utilized:**
- `## Completeness Audit Findings` -- elevated weight in classification
- `## Severity Verification Table` -- used to validate P0 claims
- Verification annotations: `[CMD_CONFIRMED]`, `[CMD_CONTRADICTED]`, `[CMD_INCONCLUSIVE]`

**Graceful degradation:** If input lacks epistemic labels, treat all findings as
`[SINGLE-SOURCE]`. If input lacks defect types, infer from context (code citations
suggest `[EXISTING_DEFECT]`, speculative concerns suggest `[PLAN_RISK]`).

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
7. **Severity-only triage** -- a `[SINGLE-SOURCE]` CRITICAL is weaker than a `[VERIFIED]` HIGH; always use epistemic-weighted severity
8. **Forcing incoherent edits** -- if an edit breaks plan coherence, rollback; a flagged finding beats a corrupted plan
9. **Ignoring integration history** -- check `integration_log.jsonl` for patterns before classifying; don't repeat deferred decisions without re-evaluation

---

## Research Credits

Design decisions in v1.2 are informed by the following research:

| Feature | Source | Reference |
|---------|--------|-----------|
| Actionability filter | Atlassian RovoDev Code Reviewer | ICSE 2026 SEIP, arXiv:2601.01129 |
| Epistemic-weighted severity | "Can LLM Agents Really Debate?" | arXiv:2511.07784 |
| Dual rollback strategy | AutoDW document workflow orchestration | arXiv:2512.04445 |
| Post-integration verification | Self-Refine (Madaan et al.) | NeurIPS 2023, arXiv:2303.17651 |
| Cross-model review pattern | ARIS (Auto-Research-In-Sleep) | github.com/wanshuiyin/Auto-claude-code-research-in-sleep |
| Persistent experiment log | pi-autoresearch (davebcn87) | github.com/davebcn87/pi-autoresearch |
| Fine-grained comment classification | Review comment taxonomy | arXiv:2508.09832 |
| Multi-agent debate protocols | Voting vs Consensus (Kaesberg et al.) | ACL 2025 Findings, arXiv:2502.19130 |
| Anti-sycophancy mechanisms | CONSENSAGENT | ACL 2025 Findings |
| Feedback-to-section mapping | Friction (Zhang et al.) | CHI 2025 |
