# PVUGC-010: CRS Validation - Underspecified Tag Mechanism

**Flaw Code:** PVUGC-010
**Severity:** 🟢 **LOW**
**Status:** 🔓 Open
**Date Identified:** 2025-10-07

---

## Component
Groth-Sahai CRS validation

## Location
- **Section:** §7 (Security Analysis)
- **Line:** 190 (CRS binding requirement)

---

## Description

Line 190 states:
> **CRS Requirement:** GS must use a **binding** CRS (SXDH/DLIN). Runtime MUST reject non-binding CRS via a tag embedded in GS_instance_digest.

**The gap:** Tag format, computation, and verification procedure are completely unspecified. Without clear specification, implementations may:
- Accept non-binding (malicious) CRS with forged tag
- Implement incompatible validation methods
- Fail to detect binding vs. witness-indistinguishable CRS

---

## Specific Issues

### 1. Binding vs. WI CRS

**GS CRS types:**
- **Binding (SXDH/DLIN):** Commitments extractable, enables soundness
- **Witness-Indistinguishable (WI):** Commitments hiding, enables zero-knowledge but may break soundness for PVUGC

**Problem:** Distinguishing at runtime is non-trivial
- Requires knowing CRS structure
- Not a simple field check

### 2. Tag Mechanism Unclear

**Questions:**
- What is the tag? A hash? A structural property?
- How is it "embedded in GS_instance_digest"?
- Who computes it? When?
- How to verify?

**No specification provided**

### 3. Forgery Risk

```
Attack scenario:
1. Attacker generates non-binding (WI) GS CRS
2. Computes "tag" that appears valid
3. Includes forged tag in GS_instance_digest
4. Runtime validation passes (incorrect implementation)
5. Uses WI CRS to break soundness (see PVUGC-002)
```

---

## Recommendations

### Specification

#### 1. Define CRS Generation Procedure
**Priority:** P3, **Timeline:** 1-2 weeks

**For binding CRS (Type-3 pairing, SXDH):**
```
GS CRS generation:
1. Sample random α, β ← ℤ_r
2. Compute basis vectors in 𝔾₁:
   u₁ = (α·G₁, 1·G₁)
   u₂ = (β·G₁, 1·G₁)
3. Compute dual basis in 𝔾₂:
   v₁ = (1·G₂, α·G₂)
   v₂ = (1·G₂, β·G₂)
4. CRS = (u₁, u₂, v₁, v₂)

Binding property (SXDH):
- DDH hard in both 𝔾₁ and 𝔾₂
- Commitments are extractable
```

#### 2. Specify Tag Computation
**Priority:** P3, **Timeline:** 1 week

**Tag definition:**
```
tag_binding = SHA256("PVUGC/GS-BINDING-v1" ||
                     ser_CRS(u₁, u₂, v₁, v₂) ||
                     "SXDH" ||
                     curve_id)

Where:
- ser_CRS: Canonical serialization of CRS elements
- "SXDH": String indicating binding assumption
- curve_id: E.g., "BLS12-381" or "BN254"
```

**Embed in GS_instance_digest:**
```
GS_instance_digest = SHA256("PVUGC/GS-INSTANCE" ||
                            GS_CRS_hash ||
                            tag_binding ||
                            {Uⱼ} || {Vₖ} ||
                            public_inputs)
```

#### 3. Define Validation Procedure
**Priority:** P3, **Timeline:** 1 week

**Runtime CRS validation:**
```
Function: ValidateBindingCRS(CRS, tag_claimed)

1. Parse CRS → (u₁, u₂, v₁, v₂)
2. Verify structural properties:
   a) All elements on curve and in correct groups
   b) Subgroup membership (cofactor clearing)
3. Compute tag_actual = SHA256("PVUGC/GS-BINDING-v1" ||
                                ser_CRS(CRS) ||
                                "SXDH" ||
                                curve_id)
4. Check: tag_actual == tag_claimed
5. Verify binding property (pairing checks):
   - e(u₁[0], v₁[1]) / e(u₁[1], v₁[0]) ≠ 1_𝔾ₜ
   - (This checks that α is not 0 or 1)
   - [Additional pairing checks based on GS variant]
6. If all checks pass: Accept CRS as binding
7. Else: Reject

Return: ACCEPT or REJECT
```

#### 4. Test Vectors
**Priority:** P3, **Timeline:** 1 week

Provide reference test vectors:
```
Test Case 1: Valid binding CRS
- Input: CRS (u₁, u₂, v₁, v₂) [hex-encoded]
- Expected tag: [hex]
- Expected result: ACCEPT

Test Case 2: Non-binding CRS (WI)
- Input: Malicious CRS
- Expected result: REJECT (tag mismatch or pairing checks fail)

Test Case 3: Binding CRS with forged tag
- Input: Valid CRS but wrong tag
- Expected result: REJECT (tag mismatch)
```

### Testing

#### 5. CRS Validation Test Suite
**Timeline:** 1 week

Tests:
- [ ] Valid binding CRS → accepted
- [ ] Non-binding CRS → rejected
- [ ] Forged tag → rejected
- [ ] Malformed CRS (wrong structure) → rejected
- [ ] CRS with small-order elements → rejected

---

## Acceptance Criteria

- [ ] CRS generation procedure specified
- [ ] Tag format and computation defined
- [ ] Validation procedure documented with pseudocode
- [ ] Test vectors provided
- [ ] Reference implementation available
- [ ] Test suite passes

---

## Related Flaws
- **PVUGC-002:** GS commitment malleability (CRS binding directly affects this)
- **PVUGC-003:** Independence violation (CRS validation relates to setup)
- **PVUGC-005:** Context binding (CRS hash should be in ctx_core)

---

**Last Updated:** 2025-10-07
