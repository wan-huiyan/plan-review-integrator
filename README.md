# Plan-Review Integrator
[![GitHub release](https://img.shields.io/github/v/release/wan-huiyan/plan-review-integrator)](https://github.com/wan-huiyan/plan-review-integrator/releases) [![Claude Code](https://img.shields.io/badge/Claude_Code-skill-orange)](https://claude.com/claude-code) [![license](https://img.shields.io/github/license/wan-huiyan/plan-review-integrator)](LICENSE) [![last commit](https://img.shields.io/github/last-commit/wan-huiyan/plan-review-integrator)](https://github.com/wan-huiyan/plan-review-integrator/commits)

After a review panel generates 14 findings with mixed severity, someone has to cross-reference each one against the plan, classify what's a real blocker versus pre-existing debt, and edit the document without breaking coherence. This skill does it in minutes with full traceability.

A Claude Code skill that closes the gap between review reports and implementation plans. Instead of manually cross-referencing findings, classifying severity, and editing plan documents, this skill systematically processes every finding, validates it against domain context, and applies concrete updates with an audit trail.

## Quick Start

```
You: Here's the review panel output for our API migration plan:
     review_panel_api_migration.md
     And here's the plan: docs/plan_api_migration.md
     Integrate the review findings into the plan.

Claude: [Reads both documents + project context]
        [Extracts 14 findings across 3 reviewers]
        [Cross-references each against the plan]
        [Classifies: 3 must-fix, 5 bundle, 4 defer, 2 informational]
        [Applies edits to plan document]
        [Produces traceability summary]

        ## Review Integration Summary
        - Total findings: 14
        - Must-fix: 3 (applied to plan)
        - Bundle into implementation: 5 (added to checklists)
        - Deferred: 4 (documented with rationale)
        - Informational: 2 (no action needed)

        | ID     | Severity | Summary                        | Category      | Action Taken                          |
        |--------|----------|--------------------------------|---------------|---------------------------------------|
        | R1-F01 | CRITICAL | Missing auth check in step 3   | Must-fix      | Added guard to Phase 2 code snippet   |
        | R1-F02 | HIGH     | Stale timeout config           | Bundle        | Added to implementation checklist      |
        | R1-F03 | MEDIUM   | Refactor legacy adapter        | Defer         | Pre-existing debt, tracked in TODO    |
        | ...    | ...      | ...                            | ...           | ...                                   |

        Would you like to adjust any classifications before finalizing?
```

## Installation

### Claude Code

```bash
# Clone the repo
git clone https://github.com/wan-huiyan/plan-review-integrator.git ~/.claude/skills/plan-review-integrator

# Or add as a project skill
git clone https://github.com/wan-huiyan/plan-review-integrator.git .claude/skills/plan-review-integrator
```

### Cursor

Add to your `.cursor/rules/plan-review-integrator.mdc`:

```
---
description: Integrate structured review panel findings into implementation plan documents
globs:
alwaysApply: false
---

@plan-review-integrator/SKILL.md
```

## What You Get

- **Actionability-filtered findings** — Low-signal noise dropped before classification; epistemic labels from upstream reviews weight effective severity
- **Classified findings** — Every finding sorted into must-fix, bundle, defer, or informational using epistemic-weighted severity (a `[VERIFIED]` HIGH outranks a `[SINGLE-SOURCE]` CRITICAL)
- **Updated plan document** — Concrete edits with dual rollback (rephrase, then revert) when edits break coherence
- **Post-integration verification** — Automated coherence, completeness, voice, and cross-reference checks
- **Peripheral updates** — ADRs, runbooks, and memory files updated where findings affect them
- **Traceability summary** — Full audit trail mapping each finding to its disposition and the action taken
- **Persistent integration log** — `integration_log.jsonl` enables cross-session learning from prior integration decisions
- **VoltAgent specialist verification** (optional) — When VoltAgent agents are installed, P0/P1 must-fix edits get domain-expert second opinions before application, catching wrong fixes that general context might miss

## Comparison

| Aspect | Manual Review Integration | With This Skill |
|--------|--------------------------|-----------------|
| Finding extraction | Read report, copy items to spreadsheet | Automated parsing with severity + source |
| Cross-referencing | Ctrl-F through plan, hope nothing is missed | Systematic matching against every plan section |
| Domain validation | Reviewer fixes applied as-is | Each fix validated against project context |
| Pre-existing debt | Mixed in with plan defects | Explicitly classified and deferred |
| Completeness audit items | Often ignored as "minor" | Elevated — systematic gaps get extra scrutiny |
| Traceability | Informal "I think we got everything" | Table linking every finding to its action |
| Time to integrate | 30-60 min per review report | 5-10 min including user confirmation |

## How It Works

| Phase | What Happens |
|-------|-------------|
| **Gather** | |
| 1. Gather Inputs | Collect review reports + plan document + domain context |
| 2. VoltAgent Detection | Scan for available VoltAgent specialists; suggest install if content signals match |
| **Analyze** | |
| 3. Extract Findings | Parse all findings with severity, source, citations, consensus status |
| 4. Cross-Reference | Match each finding against plan content (already addressed, gap, correction, new concern, pre-existing) |
| 5. Actionability Filter | Score findings on actionability + groundedness; drop low-signal noise |
| 6. Classify | Assign action category using epistemic-weighted severity |
| **Apply** | |
| 7. Apply Edits | Modify plan document with rollback on coherence breaks |
| 8. Verify | Re-read modified plan; check coherence, completeness, voice, cross-references |
| **Finalize** | |
| 9. Update Peripherals | ADRs, runbooks, memory files as needed |
| 10. Produce Summary | Traceability table with statistics and key decisions |
| 11. Persistent Log | Append decisions to `integration_log.jsonl` for cross-session learning |

## Key Design Decisions

### Domain context validation

Review panels often identify correct symptoms but prescribe wrong fixes when they lack domain context. This skill explicitly gathers project memory, configuration files, and related documentation before applying any recommendations. A CRITICAL finding might turn out to be a verification step (not a design flaw), or a correct observation about pre-existing tech debt that the current plan does not worsen.

### Completeness audit emphasis

Completeness audit findings from review panels deserve extra scrutiny because they represent systematic gaps that all individual reviewers overlooked. The skill elevates these above regular findings during classification.

### Four-category classification

Rather than a binary "apply / skip" decision, findings land in one of four categories. This prevents the common failure modes of (a) treating all CRITICAL items as plan defects, (b) losing implementation details that aren't plan-level fixes, and (c) mixing pre-existing tech debt into must-fix blockers.

## Requirements

- Claude Code v1.0+
- Structured review output from agent-review-panel (or any review with severity-rated findings)

## Related Skills

- **[agent-review-panel](https://github.com/wan-huiyan/agent-review-panel)** — Upstream. Produces the structured review findings this skill consumes.
- **[brainstorming](https://github.com/obra/superpowers)** — Plan creation. Can produce the initial plan that gets reviewed and then integrated.

## Limitations

- Requires structured review input with severity ratings. Informal code review comments without severity classification will need manual annotation first.
- Domain context validation is only as good as the project's documentation. Poorly documented projects may still get incorrect fixes applied.
- Does not re-run the review panel — it only processes existing findings. If the plan changes substantially, a new review panel run is recommended.
- The skill asks for user confirmation before finalizing, but in autonomous mode it may apply all edits without pause.

## Research Credits

v1.2 design decisions are informed by academic research and open-source projects:

| Feature | Source | Reference |
|---------|--------|-----------|
| Actionability filter | Atlassian RovoDev Code Reviewer | [ICSE 2026 SEIP](https://arxiv.org/abs/2601.01129) |
| Epistemic-weighted severity | "Can LLM Agents Really Debate?" | [arXiv:2511.07784](https://arxiv.org/abs/2511.07784) |
| Dual rollback strategy | AutoDW document workflows | [arXiv:2512.04445](https://arxiv.org/abs/2512.04445) |
| Post-integration verification | Self-Refine (Madaan et al.) | [NeurIPS 2023](https://arxiv.org/abs/2303.17651) |
| Cross-model review pattern | ARIS (Auto-Research-In-Sleep) | [wanshuiyin/Auto-claude-code-research-in-sleep](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep) (separate GitHub account from wan-huiyan) |
| Persistent experiment log | pi-autoresearch | [github.com/davebcn87/pi-autoresearch](https://github.com/davebcn87/pi-autoresearch) |
| Fine-grained comment classification | Review comment taxonomy | [arXiv:2508.09832](https://arxiv.org/abs/2508.09832) |
| Multi-agent debate protocols | Voting vs Consensus (Kaesberg et al.) | [ACL 2025 Findings](https://arxiv.org/abs/2502.19130) |
| Anti-sycophancy mechanisms | CONSENSAGENT | [ACL 2025 Findings](https://aclanthology.org/2025.findings-acl.1141/) |
| Feedback-to-section mapping | Friction (Zhang et al.) | [CHI 2025](https://dl.acm.org/doi/10.1145/3706598.3714316) |
| VoltAgent specialist routing | awesome-claude-code-subagents | [github.com/VoltAgent/awesome-claude-code-subagents](https://github.com/VoltAgent/awesome-claude-code-subagents) |

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.3.0 | 2026-03-30 | VoltAgent specialist verification for P0/P1 findings. Optional domain-expert second opinions during cross-referencing and edit application. Max 3 specialist spawns per run. Installation prompts (once per session, non-blocking). |
| 1.2.0 | 2026-03-27 | Actionability filter, epistemic-weighted severity, dual rollback, post-integration verification, persistent integration log, upstream schema contract. 10 research sources credited. |
| 1.0.0 | 2026-03-25 | Initial release. 7-phase workflow with domain context validation. |

## License

MIT
