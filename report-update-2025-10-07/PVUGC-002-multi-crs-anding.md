# PVUGC-002: Multi-CRS AND-ing Now Mandatory

**Flaw Code:** PVUGC-002
**Severity:** 🔴 Critical → ✅ **RESOLVED**
**Status:** ✅ Resolved (v2.0)
**Date Identified:** 2025-10-07 (v1.0)
**Date Resolved:** 2025-10-07 (v2.0)

---

## Component
Product-Key KEM security amplification

## Location
- **v1.0:** Missing (only single CRS mentioned)
- **v2.0:** §89-91 (Production Profile - Multi-CRS AND-ing), §6 (KEM construction)

---

## Original Problem (v1.0)

### Description

In v1.0, the protocol relied on a **single GS-CRS transcript** for the Product-Key KEM construction. This created a single point of failure:

**Single CRS vulnerability:**
- If CRS generated with known trapdoor → attacker can decrypt without proof
- If CRS has exploitable algebraic structure → GT-XPDH assumption might be weaker
- If ceremony compromised → entire protocol security breaks
- No defense-in-depth against CRS-level attacks

**Original risk assessment:**
- **Severity:** 🔴 Critical
- **Impact:** Complete protocol break if single CRS is malicious or weak
- **Likelihood:** Medium (depends on CRS ceremony integrity)

### Security Impact (v1.0)

**Attack scenario (v1.0):**
```
1. Attacker participates in or compromises CRS ceremony
2. CRS generated with known trapdoor τ
3. Attacker can compute M = G_G16^ρ directly using trapdoor
4. Decrypts all arming shares without valid proof
5. Spends Bitcoin without satisfying computation predicate
```

**No mitigation in v1.0:** Single CRS meant single point of failure.

---

## How v2.0 Resolves This

### ✅ Multi-CRS AND-ing (Mandatory)

**v2.0 Production Profile (§89-91):**

> **MUST:** Production deployments MUST use at least **2 independently generated binding GS-CRS transcripts**.

**Key changes:**

#### 1. Minimum 2 Independent CRS (§89)
```
CRS₁: First independently generated binding GS-CRS
CRS₂: Second independently generated binding GS-CRS
...
(minimum n=2, can use more for additional security)
```

#### 2. Separate Mask Sets per CRS (§90)
For each armer i and each CRS transcript k:
```
D₁,ⱼ⁽ᵏ⁾ = Uⱼ⁽ᵏ⁾^ρᵢ  (masks in 𝔾₂ for CRS #k)
D₂,ₘ⁽ᵏ⁾ = Vₘ⁽ᵏ⁾^ρᵢ  (masks in 𝔾₁ for CRS #k)
```

**Same ρᵢ across all CRS** (critical for determinism).

#### 3. Logical AND Verification (§90)
Each CRS has its own PPE verification:
```
PPE₁: ∏ⱼ e(C¹ⱼ, Uⱼ⁽¹⁾) · ∏ₘ e(Vₘ⁽¹⁾, C²ₘ) ?= G_G16
PPE₂: ∏ⱼ e(C¹ⱼ, Uⱼ⁽²⁾) · ∏ₘ e(Vₘ⁽²⁾, C²ₘ) ?= G_G16
```

**Both must pass** (logical AND).

#### 4. Combined KDF Construction (§91)
```
M_i^{AND} := ser_𝔾_T(M_i^(1)) || ser_𝔾_T(M_i^(2)) || ...

K_i = HKDF(
    salt = ctx_hash,
    IKM = M_i^{AND},
    info = "PVUGC/KEM/v1"
)
```

**Attacker must decrypt using ALL CRS outputs** to derive correct key.

#### 5. CRS Digest Pinning (§91)
```
GS_instance_digest includes:
- Hash(CRS₁)
- Hash(CRS₂)
- ...

header_meta includes:
- Both CRS digests
- Binding verification tags
```

**Prevents CRS substitution attacks.**

---

## Security Impact (Resolved)

### Defense Against CRS Compromise

**Before (v1.0):**
- Compromise 1 CRS → Full break

**After (v2.0):**
- Compromise CRS₁ only → Attack fails (CRS₂ blocks it)
- Compromise CRS₂ only → Attack fails (CRS₁ blocks it)
- Must compromise **ALL** CRS → Attack succeeds (exponentially harder)

### Exponential Security Amplification

**Theorem (Informal, §7 v2.0):**
If GT-XPDH has security level λ against single-instance attacks, then n-CRS AND-ing provides security ≈ n·λ (under independence assumption).

**Example:**
- GT-XPDH: 128-bit security per instance
- 2-CRS: ~256-bit combined security
- 3-CRS: ~384-bit combined security

**Critical assumption:** CRS must be independently generated (no shared randomness or collusion).

### Attack Complexity Analysis

**Attack type: Trapdoor-based decryption**

| CRS Count | Attacker Requirements | Complexity |
|-----------|----------------------|------------|
| 1 (v1.0) | Compromise 1 ceremony | 1 × difficulty |
| 2 (v2.0) | Compromise 2 ceremonies | 2^128 × difficulty |
| 3 | Compromise 3 ceremonies | 2^256 × difficulty |

**Attack type: Algebraic relation discovery**

If probability of finding exploitable structure is ε per CRS:
- 1 CRS: Success probability = ε
- 2 CRS: Success probability = ε²
- 3 CRS: Success probability = ε³

---

## Verification Checklist

To verify this flaw is properly resolved in implementation:

### CRS Generation
- [ ] At least 2 independent CRS ceremonies performed
- [ ] Different participant sets for each ceremony (no overlap preferred)
- [ ] Both CRS verified as **binding** (not witness-indistinguishable)
- [ ] CRS digests computed and pinned in protocol artifacts
- [ ] Ceremony transcripts published for auditability

### Protocol Implementation
- [ ] Arming phase generates masks for **all** CRS transcripts
- [ ] Same ρᵢ used across all CRS (verified via PoCE-A for each CRS)
- [ ] GS verification runs for **each** CRS independently
- [ ] All PPE verifications must pass (logical AND enforced)
- [ ] KDF combines outputs from all CRS: M^{AND} = ser(M₁) || ser(M₂) || ...

### Context Binding
- [ ] All CRS digests included in GS_instance_digest
- [ ] header_meta contains all CRS hashes
- [ ] Cannot substitute or omit any CRS (rejection on mismatch)

### Test Coverage
- [ ] Test case: Valid proof, all CRS → decrypt succeeds
- [ ] Test case: Valid proof, PPE₁ passes but PPE₂ fails → decrypt fails
- [ ] Test case: Attempt to substitute CRS → rejected
- [ ] Test case: Attempt to decrypt with only M₁ (missing M₂) → fails
- [ ] Test case: Different ρᵢ across CRS → PoCE-A fails

---

## Remaining Considerations

While this flaw is **resolved**, implementations should consider:

### 1. CRS Ceremony Design
**Recommendation:** Use multi-party computation (MPC) ceremony for each CRS:
- Public ceremony with verifiable randomness
- Multiple participants (compromise requires n-of-n collusion)
- Published ceremony transcripts
- Digest verification tools

**Example ceremony structure:**
```
CRS₁ Ceremony: Participants A, B, C (3-of-3 MPC)
CRS₂ Ceremony: Participants D, E, F (3-of-3 MPC)
Result: Requires A∧B∧C∧D∧E∧F to compromise both
```

### 2. Scalability vs Security Trade-off
**More CRS = Higher Security but More Cost:**

| CRS Count | Security Level | Arming Cost | Verification Cost |
|-----------|----------------|-------------|-------------------|
| 2 | 2×λ | 2× masks | 2× PPE |
| 3 | 3×λ | 3× masks | 3× PPE |
| 4 | 4×λ | 4× masks | 4× PPE |

**Recommendation:** Start with n=2 (mandatory minimum), increase if threat model requires.

### 3. CRS Version Management
**Consideration:** How to upgrade CRS in production?

**Options:**
- **Hard fork:** New CRS set → incompatible with old transactions
- **Soft transition:** Support multiple CRS sets, phase out old ones
- **Time-bounded:** CRS valid for epoch, auto-rotate

**Recommendation:** Design CRS rotation mechanism before mainnet.

### 4. Independence Verification
**Question:** How to verify CRS are truly independent?

**Checks:**
- Different ceremony coordinators
- Different participant sets
- Different randomness sources
- Statistical independence tests on CRS structure
- No shared secret material

**Risk:** Collusion between ceremonies weakens AND security.

---

## Related Flaws

- **PVUGC-001:** GT-XPDH assumption (mitigated by this resolution)
- **PVUGC-003:** Independence property (ensures G_G16 independent from bases for each CRS)
- **PVUGC-010:** CRS validation (✅ resolved - binding requirement added)

---

## Implementation References

### v2.0 Specification Sections
- **§89:** Production profile - Multi-CRS requirement
- **§90:** Separate mask derivation per CRS
- **§91:** KDF construction with combined M^{AND}
- **§6:** Product-Key KEM (updated for multi-CRS)
- **§7:** Security analysis (q-GT-XPDH multi-instance hardness)

### Test Vectors Needed
- [ ] Sample CRS₁, CRS₂ (binding, independent)
- [ ] Sample ρᵢ value
- [ ] Expected masks: {D₁,ⱼ⁽¹⁾}, {D₂,ₘ⁽¹⁾}, {D₁,ⱼ⁽²⁾}, {D₂,ₘ⁽²⁾}
- [ ] Expected M₁, M₂ (for valid proof)
- [ ] Expected M^{AND} = ser(M₁) || ser(M₂)
- [ ] Expected K_i = HKDF(ctx_hash, M^{AND}, "PVUGC/KEM/v1")

---

## Acceptance Criteria (Met in v2.0)

This flaw is considered **RESOLVED** because:

✅ **Specification updated:** Multi-CRS AND-ing is **MUST** requirement (§89)
✅ **Construction defined:** Separate masks, logical AND, combined KDF (§90-91)
✅ **Security analysis:** Multi-instance hardness (q-GT-XPDH) analyzed (§7)
✅ **Context binding:** CRS digests pinned in GS_instance_digest and header_meta (§91)
✅ **Validation mechanism:** Binding CRS requirement with verification (PVUGC-010)

**Next step:** Verify in reference implementation.

---

## Notes

- **v1.0 → v2.0:** Most significant security improvement
- **Impact:** Transforms single point of failure into exponential hardening
- **Defense-in-depth:** Even if one CRS is weak, others provide security
- **Ceremony importance:** Security depends on CRS independence (social/procedural)
- **Recommended deployment:** 2 CRS minimum, 3 CRS for high-value applications

---

**Version History:**
- **v1.0 (2025-10-07):** Identified as critical missing feature
- **v2.0 (2025-10-07):** Resolved - Multi-CRS AND-ing mandatory

**Last Updated:** 2025-10-07
**Status:** ✅ RESOLVED (pending implementation verification)
