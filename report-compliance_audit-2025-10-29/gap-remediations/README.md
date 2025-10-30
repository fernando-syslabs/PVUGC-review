# Gap-Remediation Reports

This directory contains gap-remediation reports generated during the research-remediation workflow when specification gaps are detected.

## Purpose

When validation detects regressions or blockers (underspecified algorithms, missing parameters, absent test vectors), the standards-researcher agent researches authoritative external sources to fill gaps. Each report:

1. Identifies specification gaps with precise citations
2. Provides recommendations grounded in IETF RFCs, NIST standards, academic papers
3. Includes implementation guidance, test vectors, and risk assessment
4. Undergoes expert validation (mathematician + crypto-peer-reviewer)
5. Enables retry validation with recommendations as proxy specification

## Workflow

```
Regression Detected → Gap Analysis → standards-researcher
                                           ↓
                            Gap-Remediation Report (this directory)
                                           ↓
                    Expert Validation → Retry Validation
                                           ↓
                    (If successful) → Promote to RECOMMENDED-STANDARDS.md
```

## Reports

*Currently empty - reports will be added when research-remediation workflow executes*

**Planned:**
- `PVUGC-010-gap-remediation-report.md` - Hash-to-curve algorithm specification for GS-CRS transparent derivation

## See Also

- `/sandbox/RECOMMENDED-STANDARDS.md` - Promoted recommendations (validated and ready for use)

