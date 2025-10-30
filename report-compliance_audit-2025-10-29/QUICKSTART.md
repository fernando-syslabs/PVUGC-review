# Compliance Audit Quickstart Guide

**Purpose:** Fast orientation for resuming work on this compliance audit

---

## 1. Check Current Status

```bash
# Read these in order:
1. 00-INDEX.md          # Progress tracker - shows what's done/pending
2. VALIDATION-STATUS.md # Current gate status and blockers
3. README.md            # Latest findings summary
```

**Key question:** What's the next pending issue?

---

## 2. Understand the Workflow

```
Assembly Line: One agent at a time, batch processing

Round 1: Lead Auditor    → Initial assessment (solo or flag for experts)
Round 2: Mathematician   → Batch process all flagged math issues
Round 3: Crypto-Reviewer → Batch process all flagged crypto issues
Round 4: Lead Auditor    → Synthesize expert votes, finalize verdicts
Round 5: Researcher      → Batch research gaps (if needed)
Round 6: Experts         → Validate remediation reports (if needed)
Round 7: Lead Auditor    → Retry validation (if needed)
```

**CRITICAL: Use Agent Tool**
- **DO:** Launch `standards-compliance-auditor` agent via Task tool for validation work
- **DON'T:** Perform validation directly in main conversation
- **WHY:** Agents follow workflow rigorously; direct work bypasses assembly line

---

## 3. Find Inputs for Next Task

**For validation:**
- Baseline: `report-peer_review-2025-10-26/PVUGC-XXX.md`
- Target spec: `PVUGC-2025-10-27.md`

**For expert consultation:**
- Validation report: `validations/stage2/PVUGC-XXX.md`
- Or: `consultations/[agent]/PVUGC-XXX.md`

**For gap remediation:**
- Initial validation identifying gaps
- Expert consultation outputs

---

## 4. Write Outputs

**Location depends on task type:**
- Validations: `validations/stage1/` or `validations/stage2/`
- Consultations: `consultations/mathematician/` or `consultations/crypto-peer-reviewer/`
- Gap remediation: `gap-remediations/`
- Retry: `retry-validations/`

**Templates:** See `templates/` directory or `REORGANIZATION-PLAN.md` §Report Templates

**Length:** Agent decides appropriate detail (concise preferred, typically 1500-2500 words)

---

## 5. Update Progress Tracker

After completing work:
```bash
# Always update:
- 00-INDEX.md    # Update status, decision method, timestamp
```

---

## 6. Key Principles

1. **Batch processing** - Do all similar tasks together
2. **Concise reports** - Detail as needed, not by default
3. **Sequential execution** - One agent at a time
4. **Clear handoffs** - Each round produces inputs for next
5. **Check status first** - Don't duplicate completed work

---

## Quick Reference

**Standards Framework:** `STANDARDS-REFERENCE-FRAMEWORK.md`
**Workflow Details:** `REORGANIZATION-PLAN.md`
**Agent Architecture:** `agents-snapshot/`
**Project Context:** `CLAUDE.md` (root) - explains document structure, issue codes, severity levels

---

**Start here:** Read `00-INDEX.md` to see what's next.
