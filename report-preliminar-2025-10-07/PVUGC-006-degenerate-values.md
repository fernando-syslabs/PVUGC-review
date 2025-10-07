# PVUGC-006: Degenerate Values - Edge Case Coverage

**Flaw Code:** PVUGC-006
**Severity:** 🟠 **HIGH**
**Status:** 🔓 Open
**Date Identified:** 2025-10-07

---

## Component
Input validation and boundary checks

## Location
- **Section:** §6 (KEM encapsulation/decapsulation)
- **Lines:** 146-148 (degenerate checks)
- **Related:** Line 221 (mandatory hygiene)

---

## Description

Protocol specifies some edge case checks:
- G_G16 = 1 → abort
- Subgroup membership checks
- Reject non-canonical encodings

**The gap:** Many boundary conditions not explicitly covered. Missing per-share validation, serialization format unspecified, edge scalars (ρᵢ=1, sᵢ=0) not fully addressed.

---

## Security Impact

**Liveness failures:** Implementations accept edge cases, then fail at different stages
**Interoperability breaks:** Different 𝔾_T serializations → different KEM keys
**Secret leakage:** ρᵢ=1 makes M publicly computable (see PVUGC-004)

---

## Specific Concerns

### 1. Point at Infinity Timing

**T = O check:** Line 203 (after aggregation)
**Missing:** Per-share Tᵢ = O check before aggregation

**Risk:** Invalid shares accepted, detected late

### 2. Zero Scalars

**ρᵢ = 0:** Checked via auxiliary relation (line 200) ✓
**sᵢ = 0:** No explicit check ✗
- If sᵢ = 0: Tᵢ = O (caught at aggregation but late)
- Should reject at per-share level

### 3. Boundary Scalar Values

**Not covered:**
- ρᵢ = 1: M = G_G16 (public), breaks secrecy
- ρᵢ = r-1: Near-boundary
- sᵢ = 1, sᵢ = r-1: Trivial values

**Recommendation:** Range restrict (see PVUGC-004)

### 4. 𝔾_T Serialization (Line 147)

> "Use canonical, subgroup-checked encoding ser_𝔾_T(·)"

**Unspecified:**
- What is "canonical" for 𝔾_T?
- 𝔾_T is Fp12 for BLS12 curves (12 field elements)
- Multiple representations: compressed, uncompressed, different bases
- **Critical:** HKDF IKM = ser(M), different encodings → different keys

**Risk:**
```
Impl A: ser_A(M) = compressed_Fp12(M)  // 540 bytes
Impl B: ser_B(M) = uncompressed_Fp12(M) // 576 bytes
Same M, different ser() → K_A ≠ K_B → decryption fails
```

### 5. G_G16 = 1 Adaptive Attack

**Line 146:** "Abort if G_G16 = 1"

**Question:** When is x committed relative to this check?
- If x chosen adaptively after CRS → attacker could force G_G16 = 1
- Requires x commitment before check (see PVUGC-003)

### 6. Small-Order Elements

**Mentioned:** Subgroup checks (line 221)
**Unclear:** Are decrypted sᵢ values checked for small order?
- If sᵢ has small order: Tᵢ also small order
- May pass individual checks but create issues in aggregation

---

## Attack Vectors

### Attack 6.1: ρᵢ = 1 Secret Leakage

```
1. Armer chooses ρᵢ = 1 (passes ρᵢ ≠ 0 check)
2. Publishes D₁,ⱼ = Uⱼ^1 = Uⱼ, D₂,ₖ = Vₖ^1 = Vₖ
3. Encapsulated: M_i = G_G16^1 = G_G16
4. G_G16 computable from (vk, x) publicly
5. Anyone derives K_i = HKDF(ctx_hash, ser(G_G16), "...")
6. Decrypts ct_i without proof

Impact: Share i leaked; if multiple shares use ρᵢ=1: α leaked
```

### Attack 6.2: Serialization Divergence

```
1. Armer uses impl A: encrypts with K_i = HKDF(..., ser_A(M_i), ...)
2. Decapper uses impl B: derives K'_i = HKDF(..., ser_B(M_i), ...)
3. K_i ≠ K'_i (different serialization)
4. Decryption fails despite valid proof
5. Funds locked until timeout

Impact: Liveness failure, interoperability break
```

### Attack 6.3: Edge Case DoS

```
1. Armer publishes T_i = O (point at infinity) or s_i = 0
2. Per-share validation missing
3. Accepted at arm-time
4. Aggregation: T = Σ T_i may be O if multiple bad shares
5. Pre-signing may complete before detection
6. Spend fails at finalization

Impact: Griefing, wasted setup cost
```

---

## Recommendations

### Immediate Actions

#### 1. Standardize 𝔾_T Serialization
**Priority:** P0
**Timeline:** 1-2 weeks

**Specification:**
```
For BLS12-381 (Type-3 pairing):
- 𝔾_T = Fp12 (tower field extension)
- Canonical encoding: Compressed Fp12 per BLS12-381 spec §X
  - 48 bytes per Fp element × 12 = 576 bytes uncompressed
  - OR: Compressed format (implementation-specific, must document)
- Byte order: Little-endian (or specify)
- Point: After pairing, apply final exponentiation, serialize result

MUST:
- All implementations produce bit-identical ser_𝔾_T(M) for same M
- Reject non-canonical (wrong length, out-of-range field elements)
```

**Provide test vectors:**
```
Input: M ∈ 𝔾_T (as result of specific pairing computation)
Output: ser(M) = [byte array, hex-encoded]
```

#### 2. Per-Share Validation
**Priority:** P0
**Timeline:** 1 week

Add normative checklist:
```
When armer i publishes (T_i, {D₁,ⱼ}, {D₂,ₖ}, ct_i):

MUST verify before accepting:
☐ T_i ≠ O (point at infinity)
☐ T_i on curve and in correct subgroup
☐ ρᵢ ∈ [2^128, r - 2^128] (from PoCE-A range proof, see PVUGC-004)
☐ {D₁,ⱼ}, {D₂,ₖ} canonical encodings
☐ {D₁,ⱼ}, {D₂,ₖ} all in correct subgroups
☐ ct_i well-formed (length, structure)
```

#### 3. Comprehensive Boundary Tests
**Priority:** P1
**Timeline:** 2 weeks

Test all edge cases:
- [ ] ρᵢ = 0 → rejected (existing check)
- [ ] ρᵢ = 1 → rejected (needs new check)
- [ ] ρᵢ = r-1 → test behavior
- [ ] sᵢ = 0 → rejected (T_i = O check)
- [ ] sᵢ = 1 → test behavior
- [ ] T_i = O → rejected at per-share level
- [ ] T = O (aggregate) → rejected before pre-signing
- [ ] G_G16 = 1 → rejected at setup
- [ ] Small-order M → rejected
- [ ] Non-canonical encodings → rejected

#### 4. G_G16 Validation
**Priority:** P1
**Timeline:** 1 week

Add to setup phase:
```
After (vk, x) fixed:
1. Compute G_G16(vk, x)
2. Verify G_G16 ≠ 1_𝔾_T
3. Verify order(G_G16) = r (full order, not subgroup)
4. Abort if checks fail

Note: x must be committed before CRS generation (see PVUGC-003)
```

### Specification Improvements

#### 5. Add Validation Section
**Timeline:** 1 week

New section in spec:
```markdown
## §6.5: Input Validation and Edge Cases (Normative)

### Scalar Bounds
- ρᵢ ∈ [2^128, r - 2^128]
- sᵢ ∈ [1, r-1]

### Point Checks
- All points: subgroup membership (cofactor clearing where applicable)
- T_i, T: ≠ O (point at infinity)
- Reject small-order points

### 𝔾_T Serialization
[As specified in Recommendation 1]

### G_G16 Validation
[As specified in Recommendation 4]
```

### Testing

#### 6. Edge Case Test Suite
**Timeline:** 2-3 weeks

Comprehensive tests:
- Boundary scalars (0, 1, 2, r-2, r-1, r)
- Identity/infinity points in all groups
- Small-order subgroup elements
- Non-canonical encodings (wrong length, invalid field elements)
- Cross-implementation serialization compatibility
- Malformed ciphertexts

---

## Acceptance Criteria

- [ ] 𝔾_T serialization standardized with test vectors
- [ ] Per-share validation implemented
- [ ] ρᵢ, sᵢ range restrictions enforced
- [ ] Comprehensive edge case test suite passes
- [ ] All implementations interoperable (serialization compatible)
- [ ] Specification includes normative validation section

---

## Related Flaws
- **PVUGC-004:** PoCE soundness (ρᵢ=1, sᵢ=0 overlap)
- **PVUGC-009:** DEM interoperability (serialization affects both)

---

**Last Updated:** 2025-10-07
