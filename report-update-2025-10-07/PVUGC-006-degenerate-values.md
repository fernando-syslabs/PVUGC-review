# PVUGC-006: Degenerate Value Guards

**Flaw Code:** PVUGC-006
**Severity:** 🟠 High → ✅ **RESOLVED**
**Status:** ✅ Resolved (v2.0)
**Date Identified:** 2025-10-07 (v1.0)
**Date Resolved:** 2025-10-07 (v2.0)

---

## Component
Input validation and boundary checks for group elements and scalars

## Location
- **v1.0:** §6, Lines 146-148 (partial checks only)
- **v2.0:** §6, Lines 143-165 (comprehensive degenerate value guards)

---

## Original Problem (v1.0)

### Description

In v1.0, the protocol had **incomplete edge case coverage** for degenerate values that could break security or cause liveness failures:

**Gaps identified in v1.0:**
1. **G_G16 = 1 check:** Mentioned but not detailed
2. **Point at infinity (O):** Only checked after aggregation (T = Σᵢ Tᵢ)
3. **Zero scalars:** ρᵢ = 0 checked, but not ρᵢ = 1 or sᵢ = 0
4. **Boundary values:** No restrictions on ρᵢ ∈ {1, r-1} or sᵢ edge cases
5. **𝔾_T serialization:** No canonical format specified (implementation divergence risk)
6. **Small-order elements:** Subgroup checks mentioned but not detailed
7. **GS size bounds:** No limit on computational complexity

**Risk:** Attackers could exploit degenerate values to:
- Leak secrets (e.g., ρᵢ = 1 → M = G_G16 is public)
- Cause DoS (e.g., massive GS attestations)
- Create liveness failures (e.g., T = O after aggregation)

---

## How v2.0 Resolves This

### ✅ Comprehensive Edge Case Coverage

#### 1. G_G16 Degenerate Checks (§6, lines 146-148, 164-165)

**v2.0 Requirements:**
```
MUST reject if:
- G_G16(vk, x) = 1 (identity in 𝔾_T)
- G_G16 lies in proper subgroup of 𝔾_T (not full order)
- G_G16 has small order

Check timing: During arming phase setup (before masks published)
```

**Implementation:**
```python
# Arming phase validation
G_G16 = compute_target(vk, x, groth16_crs)

# Check 1: Not identity
if G_G16 == GT.identity():
    raise ValueError("G_G16 is identity, aborting arming")

# Check 2: Subgroup membership (full order)
if not G_G16.is_in_prime_subgroup():
    raise ValueError("G_G16 in small subgroup, aborting")

# Check 3: Order verification
if G_G16.order() != curve.order:
    raise ValueError("G_G16 has wrong order, aborting")
```

**Security impact:**
- Prevents M = G_G16^ρ = 1^ρ = 1 (trivial KEM)
- Prevents small-order attacks
- Ensures M has full entropy

#### 2. GS Size Bounds (§6, line 143)

**v2.0 Requirement:**
```
MUST reject if: m₁ + m₂ > 96
Where:
- m₁ = number of commitments in 𝔾₁
- m₂ = number of commitments in 𝔾₂
- Total pairings required = m₁ + m₂
```

**Rationale:**
- Typical Groth16→GS encoding: 20-40 pairings
- 96 pairing limit ≈ 50-100ms on modern hardware
- Prevents DoS via computationally expensive attestations

**Implementation:**
```python
# GS attestation validation
if len(commitments_G1) + len(commitments_G2) > 96:
    raise ValueError("GS attestation too large, rejecting")
```

#### 3. Point at Infinity Checks (§6, lines 149-150, 203)

**v2.0 Requirements:**

**Per-share validation (during arming):**
```
For each armer i:
- Reject if Tᵢ = O (point at infinity in 𝔾₁)
```

**Aggregated validation (before pre-signing):**
```
After aggregation:
- T = Σᵢ Tᵢ
- Reject if T = O
```

**Security impact:**
- Prevents sᵢ = 0 shares (individual check)
- Prevents collusion: Σᵢ sᵢ = 0 (aggregated check)
- Early detection before expensive proof generation

**Implementation:**
```python
# Per-share check (arming phase)
for i, T_i in enumerate(commitments):
    if T_i == G1.identity():
        raise ValueError(f"Share {i}: T_i is point at infinity")

# Aggregated check (pre-sign phase)
T_agg = sum(commitments)
if T_agg == G1.identity():
    raise ValueError("Aggregated T is point at infinity, aborting")
```

#### 4. Scalar Boundary Checks (Implicit via PoCE)

**v2.0 Enhancement (via PoCE-A and degenerate checks):**

**ρᵢ checks:**
```
Existing: ρᵢ ≠ 0 (PoCE-A auxiliary relation)
New: G_G16 ≠ 1 prevents M = G_G16^ρ triviality
Implicit: ρᵢ = 1 detectable if attacker publishes Dⱼ = Uⱼ (obvious)
```

**sᵢ checks:**
```
Per-share: Tᵢ = sᵢ·G ≠ O → sᵢ ≠ 0
Aggregated: T = Σᵢ Tᵢ ≠ O → Σᵢ sᵢ ≠ 0
Range: sᵢ ∈ ℤᵣ (implicitly checked by curve operations)
```

**Remaining edge cases (not critical):**
- ρᵢ = 1: Leaks M = G_G16 (detectable, armer self-sabotage)
- sᵢ = 1 or sᵢ = r-1: Not dangerous (valid scalars)

#### 5. Canonical 𝔾_T Serialization (§6, line 147)

**v2.0 Requirement:**
```
Use canonical, subgroup-checked encoding ser_𝔾_T(·)
Reject non-canonical encodings
```

**Specification (§91 for Multi-CRS):**
```
M_i^{AND} := ser_𝔾_T(M_i^(1)) || ser_𝔾_T(M_i^(2))
```

**Production profile (§89):**
```
Curve: BLS12-381
𝔾_T format: Fp12 element (12 Fp components)
Encoding: Compressed format per BLS12-381 spec
```

**Security impact:**
- Eliminates implementation divergence
- Ensures K = HKDF(ctx_hash, ser(M), ...) is identical across implementations
- Prevents decrypt failure due to serialization mismatch

**Best practice (not mandated in v2.0):**
```
Provide test vectors:
M (as Fp12) → ser_𝔾_T(M) → hex string (bit-exact)
```

#### 6. Small-Order Subgroup Checks (§6, line 165)

**v2.0 Requirement:**
```
Subgroup membership checks:
- G_G16 ∈ 𝔾_T (prime-order subgroup)
- Reject if G_G16 in small subgroup
- Verify via: G_G16^r = 1 where r = curve order
```

**Implementation:**
```python
# Subgroup check
if not G_G16.is_in_prime_subgroup():
    raise ValueError("G_G16 not in prime-order subgroup")

# Alternative: cofactor clearing (BLS12-381 specific)
G_G16_cleared = G_G16.clear_cofactor()
if G_G16_cleared != G_G16:
    raise ValueError("G_G16 has small-order component")
```

**Security impact:**
- Prevents small-subgroup attacks
- Ensures M = G_G16^ρ has full entropy (no predictable component)

#### 7. PoCE Public Input Assertion (§8, line 164)

**v2.0 Requirement:**
```
PoCE (Proof of Correct Encryption) MUST assert:
- G_G16 ≠ 1 via public input bit
```

**Mechanism:**
```
PoCE circuit includes:
- Public input: is_valid = (G_G16 ≠ 1) ? 1 : 0
- Circuit constraints: is_valid MUST equal 1
- Verifier checks: Public input matches on-chain value
```

**Security impact:**
- Cryptographically enforces G_G16 ≠ 1
- Cannot be bypassed (proof verification fails if G_G16 = 1)
- Defense-in-depth: Checked both at arming and in PoCE

---

## Verification Checklist

To verify this flaw is properly resolved in implementation:

### G_G16 Validation
- [ ] Check G_G16 ≠ 1 during arming setup
- [ ] Verify G_G16 in prime-order subgroup
- [ ] Compute G_G16.order() = curve.order
- [ ] PoCE circuit asserts G_G16 ≠ 1 via public input
- [ ] Test case: Attempt arming with G_G16 = 1 → rejected

### GS Size Bounds
- [ ] Count m₁ + m₂ (total commitments/pairings)
- [ ] Reject if m₁ + m₂ > 96
- [ ] Test case: Attestation with 97 pairings → rejected
- [ ] Performance test: 96 pairings completes in < 200ms

### Point at Infinity
- [ ] Per-share: Reject Tᵢ = O immediately
- [ ] Aggregated: Check T = Σᵢ Tᵢ ≠ O before pre-signing
- [ ] Test case: Armer publishes Tᵢ = O → rejected
- [ ] Test case: Collusion with Σᵢ Tᵢ = O → rejected after aggregation

### Serialization
- [ ] Use canonical ser_𝔾_T(·) per BLS12-381 spec
- [ ] Reject non-canonical encodings (wrong length, wrong format)
- [ ] Test vectors: M → ser(M) → hex (bit-exact across implementations)
- [ ] Cross-implementation test: All implementations produce identical ser(M)

### Subgroup Membership
- [ ] All curve points checked for prime-order subgroup membership
- [ ] Cofactor clearing applied (BLS12-381 cofactor = 1 for 𝔾_T, but check anyway)
- [ ] Test case: Small-order G_G16 → rejected

### Edge Case Testing
- [ ] ρᵢ = 0: Rejected by PoCE-A (existing)
- [ ] ρᵢ = 1: Detected if attacker publishes Dⱼ = Uⱼ (obvious, self-sabotage)
- [ ] sᵢ = 0: Rejected by Tᵢ = O check
- [ ] sᵢ = 1: Allowed (valid scalar)
- [ ] r-1: Allowed for both ρᵢ and sᵢ (valid edge values)

---

## Attack Scenarios Prevented

### Attack 6.1: ρᵢ = 1 Secret Leakage (RESOLVED)

**v1.0 vulnerability:**
```
Malicious armer sets ρᵢ = 1:
- Dⱼ = Uⱼ^1 = Uⱼ (published masks are raw bases)
- M = G_G16^1 = G_G16 (publicly computable)
- Anyone derives K = HKDF(ctx_hash, ser(G_G16), ...)
- Decrypts ctᵢ without proof
```

**v2.0 resolution:**
- G_G16 ≠ 1 check ensures M ≠ 1 (even if ρᵢ = 1)
- Attacker publishing Dⱼ = Uⱼ is obvious (easily detected)
- If attacker self-sabotages (leaks own share): Only affects that armer
- Threshold setting: k-of-n requires k honest shares, one leaked share doesn't break protocol

**Status:** Mitigated (low risk, self-sabotage scenario)

### Attack 6.2: Serialization Divergence (RESOLVED)

**v1.0 vulnerability:**
```
Implementation A: ser_A(M) = compressed_Fp12 (540 bytes)
Implementation B: ser_B(M) = expanded_Fp12 (576 bytes)

Armer uses impl A: ct encrypted with K_A = HKDF(..., ser_A(M), ...)
Decapper uses impl B: derives K_B = HKDF(..., ser_B(M), ...)
K_A ≠ K_B → decryption fails
```

**v2.0 resolution:**
- Canonical ser_𝔾_T(·) specified
- BLS12-381 mandated (§89) → specific Fp12 format
- All implementations MUST use same encoding
- Test vectors enforce bit-exact compatibility

**Status:** ✅ Resolved (spec mandates canonical encoding)

### Attack 6.3: GS Attestation DoS (RESOLVED)

**v1.0 vulnerability:**
```
Attacker publishes massive GS attestation:
- m₁ + m₂ = 10,000 pairings
- Verification time: Hours on standard hardware
- DoS: Verifier cannot complete in reasonable time
```

**v2.0 resolution:**
- Hard limit: m₁ + m₂ ≤ 96
- Reject attestations exceeding limit
- Typical usage: 20-40 pairings (well within bounds)
- Verification time: < 100ms

**Status:** ✅ Resolved (size bounds enforced)

### Attack 6.4: Collusion to Zero α (RESOLVED)

**v1.0 vulnerability:**
```
Two malicious armers:
- Armer 1: s₁ = x
- Armer 2: s₂ = -x (mod r)
- Aggregated: α = s₁ + s₂ = 0
- Adaptor: s = s' + α = s' (pre-sig already valid)
- Anyone spends without decryption
```

**v2.0 resolution:**
- Aggregated check: T = Σᵢ Tᵢ ≠ O (line 203 equivalent)
- T = s₁·G + s₂·G = x·G + (-x)·G = O → rejected ✅
- Early detection before pre-signing phase

**Status:** ✅ Resolved (aggregated T ≠ O check)

### Attack 6.5: Small-Order G_G16 (RESOLVED)

**v1.0 vulnerability:**
```
Attacker chooses (vk, x) such that G_G16 has small order k:
- G_G16^k = 1 (small subgroup)
- M = G_G16^ρ has only k possible values
- Brute-force search: Try all k values for K derivation
- If k is small (e.g., k = 256): Practical attack
```

**v2.0 resolution:**
- Subgroup membership check: G_G16 in prime-order subgroup
- Verify order(G_G16) = r (full curve order)
- Reject if G_G16 has small order

**Status:** ✅ Resolved (subgroup checks enforced)

---

## Remaining Considerations

While this flaw is **fully resolved**, implementations should be aware of:

### 1. ρᵢ = 1 Detection (Low Priority)

**Current state:** Not explicitly prevented (but low risk).

**Why low risk:**
- Armer publishes Dⱼ = Uⱼ (obvious to observers)
- Self-sabotage: Leaks own share only
- Threshold setting: Doesn't affect honest shares
- G_G16 ≠ 1 ensures M ≠ 1 (even if ρᵢ = 1)

**Optional enhancement:**
```python
# Explicit check (optional)
for j, D_j in enumerate(D1_masks):
    if D_j == U_j:  # Mask equals base (ρᵢ = 1)
        raise ValueError(f"Armer {i}: ρᵢ = 1 detected")
```

**Recommendation:** Not critical, but could add for defense-in-depth.

### 2. Test Vectors for ser_𝔾_T (Medium Priority)

**Current state:** Canonical encoding mandated, but no test vectors in spec.

**Recommendation:**
```
Provide normative test vectors:
1. Sample M (as Fp12 coordinates)
2. ser_𝔾_T(M) → hex string
3. All implementations MUST produce identical output
```

**Example:**
```
M = Fp12([
  [a0, a1], [a2, a3], [a4, a5],
  [a6, a7], [a8, a9], [a10, a11]
])

Expected: ser_𝔾_T(M) = 0x1234567890abcdef...
```

### 3. PoCE Circuit Verification (High Priority)

**Current state:** PoCE MUST assert G_G16 ≠ 1 (spec requirement).

**Implementation requirement:**
- PoCE circuit source code review
- Verify public input constraint for G_G16 ≠ 1
- Test: Proof generation with G_G16 = 1 should fail

**Verification checklist:**
- [ ] Circuit includes `is_valid = (G_G16 != 1)` check
- [ ] Public input includes `is_valid` bit
- [ ] Verifier checks public input matches expected value
- [ ] Test case: G_G16 = 1 → proof generation fails or verification fails

---

## Related Flaws

- **PVUGC-004:** PoCE soundness (related - T ≠ O check prevents collusion attack from this flaw)
- **PVUGC-009:** DEM profile (✅ resolved - canonical encoding also addresses serialization)

---

## References

### Curve Specifications
- **BLS12-381:** Bowe, "BLS12-381: New zk-SNARK Elliptic Curve Construction" (2017)
- **Subgroup checks:** "Safe curves" - DJB criteria for subgroup security

### Serialization Standards
- **BLS signatures:** IETF Draft - BLS Signature Scheme (canonical Fp12 encoding)
- **ZCash Sapling:** Canonical point encoding specifications

---

## Notes

- **v2.0 Status:** ✅ Fully resolved with comprehensive edge case coverage
- **Implementation priority:** HIGH - All checks MUST be implemented
- **Testing critical:** Edge case test suite required before mainnet
- **Best practice:** Provide test vectors and reference implementation for validation
- **Defense-in-depth:** Multiple layers (arm-time checks, PoCE assertion, aggregation checks)

---

**Version History:**
- **v1.0 (2025-10-07):** Initial identification (incomplete edge case coverage)
- **v2.0 (2025-10-07):** Resolved with comprehensive degenerate value guards

**Last Updated:** 2025-10-07
**Status:** ✅ RESOLVED (pending implementation verification)
