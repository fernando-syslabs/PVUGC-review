# Recommended Standards Framework (Protocol-Specific)

**Version:** 1.0
**Date:** 2025-10-28
**Purpose:** Protocol-specific standards recommendations derived from gap-remediation research and expert validation
**Status:** PROVISIONAL (recommendations, not official protocol specifications)

---

## Overview

This document contains **validated recommendations** for filling specification gaps identified during standards compliance validation. All recommendations are:

1. ✅ Grounded in authoritative external sources (IETF RFCs, NIST standards, academic papers)
2. ✅ Validated by domain experts (mathematician + crypto-peer-reviewer)
3. ✅ Sufficient to enable interoperable implementations
4. ⚠️ PROVISIONAL until adopted into official protocol specifications

### Relationship to STANDARDS-REFERENCE-FRAMEWORK.md

- **STANDARDS-REFERENCE-FRAMEWORK.md:** General cryptographic and Bitcoin protocol standards (established)
- **RECOMMENDED-STANDARDS.md:** Protocol-specific recommendations for PVUGC (provisional)

---

## Status Taxonomy

| Status | Meaning | Action Required |
|--------|---------|-----------------|
| **🟢 ESTABLISHED** | Adopted into official protocol specification | None (reference spec) |
| **🟡 PROVISIONAL** | Expert-validated, awaiting protocol adoption | Implementers may use; auditors should verify |
| **🔴 DEPRECATED** | Superseded by newer recommendation or protocol update | Do not use for new implementations |

All recommendations start as 🟡 PROVISIONAL and transition to 🟢 ESTABLISHED only when protocol author adopts them into normative specification.

---

## Table of Contents

1. **[GS-CRS Transparent Derivation (PVUGC-010)](#pvugc-010-gs-crs-transparent-derivation)** - 🟡 PROVISIONAL (2025-10-28)
2. **Additional Recommendations** - Future gap-remediation promotions

---

## Validation Methodology

All recommendations in this document follow this validation pipeline:

```
Gap Identified (Regression/Blocker)
         ↓
Gap Classification (Type 1-5)
         ↓
standards-researcher Agent (External research)
         ↓
Gap-Remediation Report (Recommendations + sources)
         ↓
Expert Validation (Mathematician + Crypto-Peer-Reviewer)
         ↓
Retry Validation (Re-assess with recommendations)
         ↓
Promotion to RECOMMENDED-STANDARDS.md (If resolved)
         ↓
Periodic Review (Update if sources change)
```

### Gap Types

1. **Type 1: Algorithm Specification Gap** - Mechanism described but algorithm not specified
2. **Type 2: Parameter Specification Gap** - Algorithm named but parameters/constants missing
3. **Type 3: Test Vector Gap** - Algorithm specified but no test vectors
4. **Type 4: Security Analysis Gap** - Mechanism present but security properties unproven
5. **Type 5: Implementation Guidance Gap** - What is clear but how to implement securely is unclear

---

## How to Use This Document

### For Implementers

When implementing PVUGC protocol:
1. Follow official specification (`PVUGC-2025-10-27.md`) as primary reference
2. For underspecified areas, consult this document for validated recommendations
3. Verify that recommendations are still 🟡 PROVISIONAL (not yet adopted by protocol author)
4. Follow algorithm specifications, parameter choices, and test vectors exactly
5. Pass all test vectors to ensure interoperability

### For Protocol Authors

When updating protocol specification:
1. Review recommendations marked 🟡 PROVISIONAL
2. Consider adopting recommendations into normative specification
3. Update recommendation status to 🟢 ESTABLISHED when adopted
4. Reference IETF RFCs and standards explicitly in specification
5. Include test vectors in specification appendix

### For Auditors

When auditing implementations:
1. Check conformance to official specification first
2. For underspecified areas, verify conformance to PROVISIONAL recommendations
3. Validate expert validation pedigree (who validated, when, based on what sources)
4. Check test vector passage
5. Verify security properties (constant-time, subgroup checks, etc.)

---

## Adding New Recommendations

New recommendations are added through the research-remediation workflow when:
- Regression detected during Stage 1 validation
- Blocker detected during Stage 2 validation
- Specification gap prevents interoperable implementation

**Process:**
1. Gap analysis and classification (auditor)
2. Research authoritative sources (standards-researcher agent)
3. Generate gap-remediation report with recommendations
4. Expert validation (mathematician + crypto-peer-reviewer)
5. Retry validation with recommendations
6. Promotion to this document (if retry succeeds)

---

## Recommendation Template

```markdown
## N. [Issue Title] - [Recommended Component]

**Status:** 🟡 PROVISIONAL (validated YYYY-MM-DD, awaiting protocol adoption)
**Gap-Remediation Report:** `report-compliance_audit-2025-10-29/gap-remediation/PVUGC-XXX-gap-remediation-report.md`
**Validation:** ✅ Mathematician + Crypto-Peer-Reviewer (ACCEPT)

### Background

**Issue:** [Description of specification gap]

**Gap Type:** [Type 1-5]

**Research Date:** YYYY-MM-DD
**Expert Validation Date:** YYYY-MM-DD

---

### Recommended [Algorithm/Parameter/Guidance]

**Source:** [IETF RFC XXXX §Y.Z / NIST Publication / Academic Paper]

#### [Component] Specification

```
[Complete pseudocode or specification text]
```

**Authoritative Source Citations:**
- [Citation 1 with DOI/URL]
- [Citation 2 with DOI/URL]

---

### Parameter Specifications

| Parameter | Value | Source | Rationale |
|-----------|-------|--------|-----------|
| [Param 1] | [Value] | [Source] | [Why] |
| [Param 2] | [Value] | [Source] | [Why] |

---

### Implementation Guidance

**Reference Implementations:**
1. **[Library Name]** ([License])
   - Repository: [URL]
   - Function: `[function_name]()`
   - Version: [X.Y.Z]+
   - Status: ✅ [Production-grade / Reference / etc.]

**Security Considerations:**
- [Consideration 1]
- [Consideration 2]

**Constant-Time Requirements:**
- [Requirement 1]
- [Requirement 2]

---

### Test Vectors

**Test Vector 1: [Description]**
```
Input:
  [inputs]

Output:
  [expected outputs]

Verification: [How to verify]
```

[Additional test vectors 2-5]

---

### Validation Pedigree

**Gap-Remediation Research:**
- Agent: standards-researcher
- Date: YYYY-MM-DD
- Sources: [List of sources consulted]
- Report: [Link to gap-remediation report]

**Expert Validation:**
- Mathematician: ✅ ACCEPT (YYYY-MM-DD)
  - Reasoning: [Summary]
- Crypto-Peer-Reviewer: ✅ ACCEPT (YYYY-MM-DD)
  - Reasoning: [Summary]

**Retry Validation:**
- Issue: PVUGC-XXX regression check
- Retry Status: ✅ RESOLVED WITH GAP-REMEDIATION
- Date: YYYY-MM-DD
- Report: [Link to retry validation report]

**Promotion Decision:**
- Promoted to RECOMMENDED-STANDARDS.md: YYYY-MM-DD
- Status: PROVISIONAL (awaiting protocol author adoption)

---

### Usage Guidelines

**For Implementers:**
1. [Guideline 1]
2. [Guideline 2]

**For Protocol Authors:**
1. [Guideline 1]
2. [Guideline 2]

**For Auditors:**
1. [Guideline 1]
2. [Guideline 2]

---

### Open Questions / Future Work

**[Status]:** [Description or "None currently"]

**Future Considerations:**
- [Consideration 1]
- [Consideration 2]
```

---

## PVUGC-010: GS-CRS Transparent Derivation

**Status:** 🟡 **PROVISIONAL** (Expert-validated, pending protocol adoption)
**Research Date:** 2025-10-28
**Authoritative Sources:** RFC 9380, CFRG Drafts (BLS Signatures, Pairing-Friendly Curves), Groth-Sahai EUROCRYPT '08, Poseidon2 AFRICACRYPT '23

**Validation Pedigree:**
- **Gap-Remediation Report:** `/sandbox/report-compliance_audit-2025-10-29/gap-remediation/PVUGC-010-gap-remediation-report.md` (~12,500 words, 8 gaps, 8 authoritative sources)
- **Mathematician Expert (M2):** ACCEPT - Gap classifications accurate, references appropriate, recommendations mathematically sound, gaps adequately closed
- **Crypto-Peer-Reviewer Expert (C1):** ACCEPT - Transparent CRS cryptographically sound, defense-in-depth adequate, net security advancement
- **Retry Validation:** ✅ No Regression (with remediation) - Specification now implementable, interoperable, and secure

### Summary

The v2.7 protocol specification introduces a **transparent hash-to-curve CRS derivation** approach that eliminates ceremony trust requirements (no MPC needed). This architectural change is **theoretically superior** to v2.0's ceremony-based approach but requires complete implementation specifications. This section provides validated recommendations grounded in IETF RFC 9380 and CFRG drafts.

### Key Recommendations

**1. Hash-to-Curve Algorithm: RFC 9380 Suite BLS12381G1_XMD:SHA-256_SSWU_RO_**
- Simplified Shallue-van de Woestijne-Ulas (SWU) method for BLS12-381 G₁
- Domain separation tags: "PVUGC-v2.7/GS-CRS-TRANSPARENT/u1" and "/u2"
- Independence checks: u₁ ≠ identity, u₂ ≠ identity, u₁ ≠ u₂, DLOG independence

**2. Optional Binding Verification: Pairing-Based Defense-in-Depth**
- SHOULD perform e(u₁, v₁) ≠ e(u₂, v₂) check (MUST for high-value deployments)
- Catches catastrophic hash-to-curve implementation bugs
- Cost: ~5-10ms (one-time setup phase)

**3. GS_instance_digest Computation: 218-byte Preimage with SHA-256**
- Exact structure: domain_tag (26B) || u₁ (48B) || u₂ (48B) || vk (32B) || x (32B) || ctx (32B)
- Binds CRS to protocol context

**4. Canonical Serialization: CFRG BLS Signatures Draft §2.3**
- Compressed encoding (48 bytes G₁, 96 bytes G₂, big-endian)
- Subgroup checks (h₁=1 automatic, h₂≈2^128 requires is_torsion_free())

**5. Test Vectors: Minimum 5 (RFC 9380 Appendix J Format)**
- Zero inputs, non-zero inputs, maximum inputs, independence check, multi-CRS

**6. Poseidon2 KDF Parameters: Width-3, BLS12-381 Scalar Field**
- Rate=2, capacity=1, R_F=8, R_P=56 per Poseidon2 Table 2
- Alternative: HKDF-SHA256 (IETF standard option)

### Implementation Guidance

**Reference Implementation:** Rust with arkworks (~250 lines), published to GitHub + crates.io

**Interoperability:** Cross-library validation with arkworks, blst, py_ecc using test vectors

**Security Analysis:** Comprehensive attack resistance analysis (implementation bugs, cryptographic attacks, specification attacks)

### Acceptance Criteria for Protocol Adoption

- [ ] Normative text added to PVUGC specification (§6.5, §6.6, §3.2, Appendices X/Y) (~2000 lines)
- [ ] Test vectors generated and published (minimum 5 from Gap 4)
- [ ] Reference implementation published to GitHub repository
- [ ] Community validation (2+ independent implementations verified)
- [ ] External security audit (optional but recommended)

**Timeline Estimate:** 6-10 weeks (2-4 weeks specification updates, 4-6 weeks community validation)

### Detailed Technical Specifications

For complete technical details, algorithms, test vector formats, and security analysis, see:
- **Gap-Remediation Report:** `/sandbox/report-compliance_audit-2025-10-29/gap-remediation/PVUGC-010-gap-remediation-report.md`
- **Retry Validation:** `/sandbox/report-compliance_audit-2025-10-29/stage1-regression-testing/PVUGC-010-retry-validation.md`
- **Expert Consultations:** `.consultation-pvugc-010-mathematician-response.md`, `.consultation-pvugc-010-crypto-response.md`

---

## Cross-References

**Related Documents:**
- `/sandbox/STANDARDS-REFERENCE-FRAMEWORK.md` - General cryptographic standards
- `/sandbox/report-compliance_audit-2025-10-29/standards-compliance-validation-plan.md` - Validation methodology
- `/sandbox/report-compliance_audit-2025-10-29/gap-remediation/` - Gap-remediation reports

---

## License

This document is licensed under CC-BY 4.0.
Protocol specification authored by sidhujag.
Gap-remediation research and validation by Claude (multiple agents).

---

*Last updated: 2025-10-28*
