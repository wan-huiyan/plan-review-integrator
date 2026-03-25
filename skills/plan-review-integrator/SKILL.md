---
name: plan-review-integrator
version: 1.0.0
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
  plan based on the review", or invokes /plan-review-integrator. Does NOT trigger for
  running a review panel (use agent-review-panel), writing a plan from scratch, or
  general code review.
---

# Plan-Review Integrator v1.0

Consumes structured review panel output and integrates findings into an
implementation plan document — turning review feedback into concrete plan updates
with full traceability.

## Design Rationale

Review panels produce rich findings but leave a gap: the findings sit in a report
while the plan remains unchanged. Manual integration is error-prone — items get
dropped, severity gets misread, pre-existing tech debt gets confused with plan
defects, and reviewers' prescribed fixes may be wrong without domain context.

This skill closes the loop by systematically processing every finding, classifying
it, and either applying it to the plan or explicitly deferring it with rationale.

### Key insight from production use

Review panels often identify correct *symptoms* but prescribe wrong *fixes* when
they lack domain context. Always validate recommendations against domain-specific
constraints before applying them. A "CRITICAL" finding may turn out to be a
verification step (not a design flaw), or a correct observation about pre-existing
tech debt that the current plan does not worsen.

---

## Process Overview

```
Phase 1: Gather Inputs       -> Collect review reports + plan document
Phase 2: Extract Findings    -> Parse all findings with severity, context, citations
Phase 3: Cross-Reference     -> Match each finding against plan content
Phase 4: Classify            -> Assign action category to each finding
Phase 5: Apply Edits         -> Modify plan document with fixes, additions, caveats
Phase 6: Update Peripherals  -> ADRs, runbooks, memory files as needed
Phase 7: Produce Summary     -> Traceability table: finding -> action taken
```

---

## Phase 1: Gather Inputs

### Identify the review report(s)

The user provides or points to review panel output. This can be:
- A file path to a review report markdown file
- Multiple review reports (e.g., different panels reviewed the same plan)
- Inline review findings pasted into the conversation
- A reference to a previous conversation's review output

If multiple reports exist, collect all of them. Later phases will deduplicate.

### Identify the plan document

The user provides or points to the implementation plan. This can be:
- A file path to a plan markdown document
- A design doc, RFC, or architecture proposal
- Any structured document with implementation steps

Read the full plan document. Understand its structure: phases, sections, code
snippets, checklists, verification steps.

### Identify domain context

Check for domain-specific context that reviewers may have lacked:
- Project memory files (e.g., `memory/` directory, `MEMORY.md`)
- Configuration files that define valid values, feature lists, constants
- Related documentation that clarifies design decisions
- Previous session context about why certain choices were made

This context is essential for validating reviewer recommendations. Do NOT skip
this step.

---

## Phase 2: Extract Findings

For each review report, extract every finding into a structured list:

```
For each finding, capture:
- ID:          Sequential identifier (R1-F01, R1-F02, ... R2-F01, ...)
- Severity:    CRITICAL | HIGH | MEDIUM | LOW (as assigned by the review panel)
- Source:      Which reviewer(s) raised it, or "Completeness Audit", or "Judge"
- Summary:     One-line description of the finding
- Detail:      Full context including any code snippets or citations
- Prescribed:  What the reviewer recommended doing about it
- Consensus:   Was this a consensus point, disputed, or unilateral?
- Location:    Where in the plan this relates to (section, line, code block)
```

### Handling multiple reports

When multiple review panels reviewed the same plan:
- Findings that appear in multiple reports: merge into one entry, note all sources
- Conflicting recommendations between panels: flag for Phase 4 classification
- Unique findings from each panel: keep as separate entries

### Pay special attention to

- **Completeness audit findings** — these often catch things all reviewers missed.
  They deserve extra scrutiny because they represent systematic gaps, not just
  individual reviewer opinions.
- **Judge rulings on disputed items** — the judge's resolution is authoritative
  for classification purposes.
- **Items marked "resolved during debate"** — these may still need documentation
  even if no plan change is required.

---

## Phase 3: Cross-Reference

For each extracted finding, determine its relationship to the plan:

### Categories

1. **Already addressed** — The plan already handles this concern. The reviewer
   may have missed it or the plan addresses it differently than the reviewer
   expected. Note the relevant plan section.

2. **Gap** — The plan does not address this concern and should. This is a
   missing section, missing step, or missing consideration.

3. **Correction** — The plan addresses this topic but contains an error.
   Wrong code snippet, incorrect value, missing edge case, wrong assumption.

4. **New concern** — Something not previously considered that affects the plan's
   scope, timeline, or approach. May require structural changes to the plan.

5. **Pre-existing** — A valid concern about the broader system, but not
   introduced or worsened by this plan. The plan is not the right place to fix it.

### Validation against domain context

For each finding, ask:
- Does the reviewer's prescribed fix make sense given domain constraints?
- Is the reviewer assuming something about the system that isn't true?
- Would the prescribed fix break something the reviewer doesn't know about?
- Is this concern already mitigated by a mechanism the reviewer didn't see?

Document any cases where the finding is valid but the prescribed fix is wrong.
You will need to supply the correct fix in Phase 5.

---

## Phase 4: Classify

Assign each finding to one of four action categories:

### Must-fix before implementing

Criteria: CRITICAL or HIGH severity AND cross-reference category is "Correction"
or "Gap" AND affects correctness, data integrity, or security.

Examples:
- Wrong SQL logic in a code snippet
- Missing temporal guard that would cause data leakage
- Incorrect display values or mapping tables
- Missing error handling for a failure mode that corrupts data

These get applied to the plan immediately in Phase 5.

### Bundle into implementation

Criteria: Items that are valid and should be addressed, but are implementation
details rather than plan defects. The plan's design is correct; the execution
needs to include these items.

Examples:
- Pipeline quality gates to add during build
- Stale default values to update when deploying
- Additional test cases to include in the verification suite
- Performance optimizations to apply during implementation

These get added to the plan's implementation checklist or verification section.

### Defer

Criteria: LOW severity items, OR pre-existing tech debt not worsened by the plan,
OR items that belong to a different workstream.

Examples:
- Refactoring suggestions for code not touched by this plan
- Style/convention improvements unrelated to the plan's goals
- Known limitations acknowledged in project documentation
- Future enhancements that are out of scope

These get documented in a "Deferred Items" section or in project memory, with
rationale for deferral.

### Informational

Criteria: Items that were raised, debated, and resolved during the review. The
finding was a false alarm, was clarified by judge ruling, or represents a valid
observation that requires no action.

Examples:
- Reviewer flagged a potential issue that the judge ruled is not applicable
- Concern about a design choice that was debated and the current approach confirmed
- Observation about system behavior that is documented and intentional

These get a brief note in the summary but no plan changes.

### Handling conflicts

When the classification is ambiguous:
- If severity is CRITICAL but the item is pre-existing: classify as "defer" but
  add a caveat to the plan noting the risk and linking to a future TODO.
- If multiple panels disagree on severity: use the higher severity for
  classification but note the disagreement.
- If the prescribed fix is wrong but the finding is valid: classify based on the
  finding's severity, not the fix quality. Supply the correct fix.

---

## Phase 5: Apply Edits

### For "must-fix" items

Edit the plan document directly:
- Fix incorrect code snippets with the correct version
- Add missing temporal guards, edge cases, error handling
- Correct mapping tables, display values, constants
- Add explanatory comments noting the fix came from review

### For "bundle into implementation" items

Add to the plan's implementation checklists:
- Add items to existing verification/testing sections
- Create new checklist items in the appropriate phase
- Add quality gates where reviewers identified risks
- Note any sequencing dependencies

### For "gap" items requiring new sections

Add new sections to the plan:
- Pre-implementation checks (verify prerequisites before starting)
- Go/no-go criteria (conditions that must be met to proceed)
- Rollback procedures (if the plan doesn't have them)
- Monitoring/alerting requirements
- Migration steps for data or schema changes

### For items needing caveats

Add inline notes where reviewers identified risks:
- Use a consistent format: `> **Review note:** [description of concern and mitigation]`
- Place caveats near the relevant plan content, not in a separate section
- Include the finding ID for traceability

### Structural changes

If the review findings suggest the plan needs reordering or restructuring:
- Move sections to reflect correct dependency ordering
- Split phases that are too large or have mixed concerns
- Add summary tables if the plan has grown complex

### Edit discipline

- Make the minimum necessary change for each finding
- Preserve the plan's existing voice and structure
- Do not rewrite sections that are correct
- Mark every edit with a brief comment linking to the finding ID if the plan
  is long enough that traceability matters

---

## Phase 6: Update Peripherals

### Memory files

If the project uses memory files (e.g., `MEMORY.md`, `memory/*.md`):
- Document key findings that affect future sessions
- Note any new conventions or constraints discovered during review
- Update known TODOs if the review revealed new ones

### Architecture Decision Records

If the plan introduces architectural decisions that were validated or modified
by the review:
- Create or update ADRs documenting the decision, alternatives considered,
  and rationale (including review panel input)

### Runbooks

If the review modified deployment, rollback, or operational procedures:
- Update relevant runbooks to match the revised plan

### Configuration

If the review identified configuration values that need updating:
- Note the specific files and values to change
- Add to pre-implementation checklist if not yet actionable

---

## Phase 7: Produce Summary

Create a traceability table mapping every finding to its disposition:

```markdown
## Review Integration Summary

### Statistics
- Total findings: N
- Must-fix: X (applied to plan)
- Bundle into implementation: Y (added to checklists)
- Deferred: Z (documented with rationale)
- Informational: W (no action needed)

### Finding Disposition

| ID     | Severity | Summary                          | Category    | Action Taken                              |
|--------|----------|----------------------------------|-------------|-------------------------------------------|
| R1-F01 | CRITICAL | Missing temporal guard in SQL     | Must-fix    | Added guard to Phase 2 SQL snippet        |
| R1-F02 | HIGH     | Stale default in config           | Bundle      | Added to implementation checklist item 3  |
| R1-F03 | MEDIUM   | Refactor identity resolution      | Defer       | Pre-existing tech debt, tracked in TODO   |
| R1-F04 | LOW      | Style: use f-strings              | Informational | No action — resolved in debate          |
| ...    | ...      | ...                              | ...         | ...                                       |

### Key Decisions
- [Any significant judgment calls made during classification]
- [Any cases where reviewer's prescribed fix was overridden and why]
- [Any structural changes made to the plan]
```

Present this summary to the user. Ask if they want to adjust any classifications
before finalizing.

---

## Composability

- **Input:** Output from `agent-review-panel` skill, or any structured review
  with severity-rated findings.
- **Output:** Updated plan document, optional ADRs/runbooks, traceability summary.
- **Downstream:** Updated plan feeds into implementation phases. The traceability
  summary serves as an audit trail.
- **Upstream:** `agent-review-panel` produces the review findings this skill
  consumes. `brainstorming` may produce the initial plan that gets reviewed.

---

## Anti-Patterns to Avoid

1. **Applying prescribed fixes blindly.** Always validate against domain context.
   Reviewers lack project history and may suggest fixes that break invariants.

2. **Treating all CRITICAL items as plan defects.** Some CRITICAL findings are
   verification steps ("check that column X exists in BigQuery") — these belong
   in checklists, not as plan corrections.

3. **Ignoring completeness audit findings.** These are often the highest-signal
   items because they represent systematic gaps that all reviewers overlooked.

4. **Over-classifying pre-existing debt as must-fix.** If the plan doesn't worsen
   an existing issue, it belongs in "defer" with a link to the relevant TODO,
   not as a blocker for this plan.

5. **Making the plan document a dump of all findings.** Each finding should land
   in exactly the right place — inline caveats near relevant content, checklist
   items in verification sections, deferrals in project memory. Do not add a
   giant "Review Findings" section that duplicates the summary.

6. **Forgetting to ask the user before finalizing.** Present the classification
   summary and let the user adjust before applying edits. Some classifications
   require domain judgment that only the user has.
