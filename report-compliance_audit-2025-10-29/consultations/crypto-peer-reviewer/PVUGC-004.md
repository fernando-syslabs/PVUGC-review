# Crypto-Peer-Reviewer Response: PVUGC-004 (PoCE Soundness)

**Date:** 2025-10-28
**Reviewer:** Crypto-Peer-Reviewer
**Consultation ID:** PVUGC-004-regression-check
**Response Type:** Expert Cryptographic Assessment

---

## Vote

**PASS** — No Regression Detected

The PVUGC-2025-10-27.md specification (lines 274-278) maintains all essential cryptographic soundness properties of the Hardened PoCE-A Circuit. The mitigation that prevents the liveness griefing attack is preserved.

---

## Reasoning

### 1. Cryptographic Enforcement (Constraint C5 Validation)

**Assessment:** ✅ **PASS** — Ciphertext correctness is cryptographically enforced

**Evidence (Line 277):**
```
DEM correctness (SNARK-friendly): ct_i = (s_i|h_i) ⊕ Poseidon2(K_i, AD_core)
                                    and τ_i = Poseidon2(K_i, AD_core, ct_i)
```

**Analysis:**

The equation `ct_i = (s_i|h_i) ⊕ Poseidon2(K_i, AD_core)` on line 277 is a **constraint equation**, not just a description. In the NIZK context:

- The prover must witness `(ρ_i, s_i, h_i)` satisfying this equation
- The circuit verifies this equality holds using the public input `ct_i`
- The label "**SNARK-friendly**" confirms this is an in-circuit check

**Cryptographic Implication:**

By SNARK soundness, if `Verify_PoCE-A(π) = TRUE`, then the constraint `ct_i = (s_i|h_i) ⊕ Poseidon2(K_i, AD_core)` holds. This is precisely constraint C5.3 from the historical mitigation.

**Attack Prevention:**

If an adversary publishes `ct_i' = random_bytes()`:
- The equation `ct_i' ≟ (s_i|h_i) ⊕ Poseidon2(K_i, AD_core)` will be false
- The NIZK proof generation will **fail** (constraint unsatisfiable)
- No valid proof can be published with garbage ciphertext

**Conclusion:** Constraint C5 is present and enforced. The consolidated form is sufficient.

### 2. In-Circuit Key Derivation (Constraint C4 Validation)

**Assessment:** ✅ **PASS** — Key derivation binds ciphertext to KEM randomness

**Evidence (Line 276):**
```
Key derivation (in-circuit): K_i = Poseidon2(ser_GT(R(vk,x)^{ρ_i}) ||
                                              H_bytes(ctx_hash) ||
                                              GS_instance_digest)
```

**Analysis:**

The term `R(vk,x)^{ρ_i}` on line 276 requires **in-circuit G_T exponentiation**:

- `R(vk,x)` is the pairing product target (equivalent to historical `G_{G16}`)
- `R(vk,x)^{ρ_i}` is the KEM key `M_i` (constraint C4.1)
- The label "**in-circuit**" confirms this computation occurs within the NIZK

**Cryptographic Binding Chain:**

```
ρ_i (witness) → M_i = R(vk,x)^{ρ_i} (C4.1) → K_i = Poseidon2(ser(M_i) || ...) (C4.2)
              → keystream = Poseidon2(K_i, AD_core) (C5.2)
              → ct_i = pt ⊕ keystream (C5.3)
```

This chain binds the published `ct_i` to the mask randomness `ρ_i` through in-circuit constraints. An adversary cannot:
- Use different `ρ_i'` for masks vs ciphertext (C4.1 enforces same exponent)
- Skip key derivation (C4.2 enforces Poseidon2 computation)
- Publish arbitrary ciphertext (C5.3 enforces XOR relation)

**Conclusion:** Constraint C4 is present. The key derivation binding is cryptographically sound.

### 3. Attack Prevention Analysis

**Assessment:** ✅ **PASS** — Garbage ciphertext attack is cryptographically prevented

**Attack Scenario (Historical):**
```pseudocode
Adversary:
  1. Generate valid (ρ_i, s_i)
  2. Compute correct masks {D_j = Y_j^{ρ_i}}
  3. Publish garbage ct_i' = random_bytes()
  4. Attempt to generate PoCE-A proof

Result under v2.7 specification:
  - Circuit computes M_i = R(vk,x)^{ρ_i} (line 276)
  - Circuit derives K_i = Poseidon2(ser_GT(M_i) || ...) (line 276)
  - Circuit checks ct_i' ≟ (s_i|h_i) ⊕ Poseidon2(K_i, AD_core) (line 277)
  - Check FAILS (ct_i' ≠ (s_i|h_i) ⊕ ...)
  - Proof generation FAILS (constraint unsatisfied)
```

**Why the Attack Fails:**

The adversary cannot satisfy line 277's constraint without computing the correct ciphertext:
- `ct_i` is a **public input** to the NIZK (verifier checks published value matches circuit output)
- The circuit enforces `ct_i = (s_i|h_i) ⊕ Poseidon2(K_i, AD_core)`
- Publishing `ct_i'` ≠ correct value → constraint violated → no valid proof

**SNARK Soundness Guarantee:**

Under standard SNARK soundness (Groth16, Plonk, etc.):
```
Pr[∃ π: Verify(π, ct_i) = TRUE ∧ ct_i ≠ (s_i|h_i) ⊕ Poseidon2(K_i, AD_core)] ≤ negl(λ)
```

This is the Decrypt-on-Proof Soundness property from the historical analysis (lines 659-701).

**Conclusion:** The attack is cryptographically impossible under the v2.7 specification.

### 4. Notation Equivalence

**Assessment:** ✅ **PASS** — Notation changes are semantically equivalent

**Mapping:**

| Historical (v2.0) | Current (v2.7) | Equivalence Check |
|-------------------|----------------|-------------------|
| `G_{G16}` | `R(vk,x)` | ✅ Both represent pairing product target in G_T |
| `M_i = G_{G16}^{ρ_i}` | `R(vk,x)^{ρ_i}` | ✅ Same G_T exponentiation operation |
| `U_j, V_k` | `Y_j, [δ]_2` | ✅ CRS bases (different variable names, same role) |
| `ser()` | `ser_GT()` | ✅ Explicit group serialization (improved clarity) |

**Cryptographic Properties:**

All notation changes are **syntactic** (variable renaming) or **clarifying** (explicit group annotation). No semantic changes to:
- Group structure (G_T pairing product)
- Exponentiation operation (discrete log hardness)
- Serialization function (injective encoding)
- Key derivation function (Poseidon2 collision resistance)

**Risk Assessment:** No security impact from notation changes.

### 5. Constraint Completeness

**Assessment:** ✅ **PASS** — All critical constraints present (explicitly or implicitly)

**Constraint Checklist:**

| Constraint | Line 274-278 Status | Form |
|------------|---------------------|------|
| C1: Mask correctness | ✅ Line 275: `D_j=Y_j^{ρ_i}`, `D_δ=[δ]_2^{ρ_i}` | Explicit |
| C2: Adaptor share | ✅ Line 275: `T_i=s_iG` | Explicit |
| C3.1: Randomness link | ✅ Line 275: `ρ_link=H_tag(ρ_i)` | Explicit |
| C3.2: Non-zero | ✅ Line 278: `ρ_i≠0` (via `ρ_i·u_i=1`) | Explicit |
| C4.1: G_T exp | ✅ Line 276: `R(vk,x)^{ρ_i}` | Implicit (in K_i derivation) |
| C4.2: Key derivation | ✅ Line 276: `K_i = Poseidon2(...)` | Explicit |
| C5.1: Plaintext | ✅ Line 277: `s_i|h_i` | Implicit (in ct_i equation) |
| C5.2: Keystream | ✅ Line 277: `Poseidon2(K_i, AD_core)` | Implicit (in ct_i equation) |
| C5.3: Ciphertext | ✅ Line 277: `ct_i = (s_i|h_i) ⊕ Poseidon2(...)` | Explicit |
| C5.4: Tag | ✅ Line 277: `τ_i = Poseidon2(K_i, AD_core, ct_i)` | Explicit |

**Note on Implicit vs Explicit:**

The v2.7 specification uses **compact mathematical notation** where some constraints are implicit in equations:
- C4.1 is implicit in the term `R(vk,x)^{ρ_i}` (computing this requires G_T exponentiation)
- C5.1-C5.2 are implicit in the XOR equation (constructing the operands requires these steps)

This is **standard practice** in cryptographic specifications where the implementation details follow from the mathematical formulation. The "SNARK-friendly" label confirms these are in-circuit constraints.

**Comparison to Historical Specification:**

The historical specification (report-peer_review-2025-10-26/PVUGC-004.md, lines 413-492) provided:
- Explicit enumeration of C4.1, C4.2, C5.1-C5.4
- Implementation guidance for each constraint
- Complexity analysis

The v2.7 specification provides:
- Consolidated mathematical formulation
- Same cryptographic requirements
- Less implementation detail (appropriate for protocol specification vs implementation guide)

**Conclusion:** All critical constraints are present. The consolidated form is sufficient for unambiguous implementation.

---

## Risk Assessment

**Security Impact:** ✅ **NONE** — No regression detected

**Attack Scenarios Enabled:** ❌ **NONE** — Garbage ciphertext attack remains prevented

**Quantitative Analysis:**

| Metric | Original Design (Vulnerable) | v2.0 (Hardened) | v2.7 (Current) |
|--------|------------------------------|-----------------|----------------|
| **Attack Success Probability** | 100% (deterministic) | 0% (cryptographically impossible) | 0% (cryptographically impossible) |
| **Proof Generation with Garbage ct** | ✅ Succeeds (no C5 check) | ❌ Fails (C5.3 unsatisfied) | ❌ Fails (line 277 constraint) |
| **Arm-time Verification** | ✅ Passes (insufficient checks) | ✅ Passes (only with correct ct) | ✅ Passes (only with correct ct) |
| **Decapsulation Outcome** | ❌ Fails (garbage ct) | ✅ Succeeds (correct ct guaranteed) | ✅ Succeeds (correct ct guaranteed) |

**Soundness Property:**

```
∀ PPT A: Pr[
  (π, ct_i, ...) ← A(pp, CRS)
  ∧ Verify_PoCE-A(π, ct_i, ...) = TRUE
  ∧ Decaps(..., ct_i) → ⊥
] ≤ negl(λ)
```

This property holds under both v2.0 and v2.7 specifications.

---

## Supporting Observations

### 1. Explicit Security Rationale (Line 307)

The specification explicitly acknowledges the attack vector:

```markdown
**Why PoCE is needed:** Without PoCE-A, a malicious armer could publish masks/ciphertexts
not derived from the same ρ_i, breaking decrypt-on-proof soundness.
```

This demonstrates awareness of the security requirement and confirms the mitigation is intentional.

### 2. Normative Ceremony Rule (Line 296)

```markdown
Ceremony rule (MUST): do not pre-sign unless all PoCE-A proofs and PoK verify for all shares.
```

This enforces the arm-time verification requirement, ensuring the PoCE-A constraints are checked before protocol continuation.

### 3. Implementation Security (Line 306)

```markdown
**Mandatory hygiene:** ... constant-time pairings, constant-time DEM decryption, ...
```

Additional defense-in-depth requirements for timing attack prevention (cross-reference to PVUGC-007).

### 4. Liveness Protection (Line 315)

```markdown
**Timeout/Abort path (MUST for production).** ... Required to prevent indefinite liveness
failure if any armer posts malformed masks/ciphertexts that pass PoK but fail PoCE-B
at decap time.
```

Acknowledges that **if** PoCE-A were bypassed or failed, a timeout path is needed. This defense-in-depth layer is preserved.

---

## Comparison with Standards Framework

### §3A.1 (Decrypt-on-Proof Soundness)

**Framework Requirement:**
```
For all PPT adversaries, probability that arm-time verification passes
but decapsulation fails is negligible
```

**v2.7 Compliance:**
- ✅ Line 277 enforces ciphertext correctness in-circuit
- ✅ SNARK soundness + constraint C5.3 → decapsulation succeeds with overwhelming probability
- ✅ Requirement met

### §5A.3 (Circuit Constraint Implementation)

**Framework Requirement:**
```
Implementations MUST include all constraints specified in the circuit definition
```

**v2.7 Compliance:**
- ✅ Line 280: "**Implementation note (MUST):** PoCE-A MUST use per-column Schnorr commitments..."
- ✅ Normative language present
- ✅ Requirement met

### §8.1 (Constant-Time Execution)

**Framework Requirement:**
```
Timing-sensitive operations MUST use constant-time implementations
```

**v2.7 Compliance:**
- ✅ Line 306: "constant-time pairings, constant-time DEM decryption"
- ✅ Cross-reference to PVUGC-007
- ✅ Requirement met

---

## Recommendation

**Final Determination:** ✅ **No Regression**

**Justification:**

1. **All critical constraints present:** C4 (key derivation) and C5 (DEM correctness) are explicitly specified on lines 276-277

2. **Attack prevention mechanism intact:** The garbage ciphertext attack is cryptographically prevented by constraint C5.3 (line 277)

3. **Notation changes benign:** All variable renamings are semantically equivalent with no security impact

4. **Normative requirements preserved:** MUST language for ceremony rules and implementation notes

5. **Standards compliance maintained:** All applicable framework sections (§3A.1, §5A.3, §8.1) satisfied

**Confidence Level:** **HIGH** — The v2.7 specification unambiguously requires the Hardened PoCE-A Circuit constraints that prevent the identified attack.

---

## Additional Notes

### Consolidated vs Enumerated Constraints

The v2.7 specification uses **consolidated mathematical notation** (line 277 combines C5.1-C5.4 into compact form) rather than **enumerated implementation steps** (historical report's explicit C5.1, C5.2, C5.3, C5.4).

**Assessment:** This is **appropriate** for a protocol specification:
- Protocol specs define "what" (cryptographic requirements)
- Implementation guides define "how" (step-by-step algorithms)

The consolidated form is **sufficient** for unambiguous implementation because:
- The equation `ct_i = (s_i|h_i) ⊕ Poseidon2(K_i, AD_core)` unambiguously specifies the computation
- The "SNARK-friendly" label indicates in-circuit verification
- Standard NIZK implementation practices will decompose this into constraints

### Cross-Issue Dependencies

**PVUGC-010 (CRS Validation):** The PoCE-A soundness depends on:
- Correct CRS bases `{Y_j}, [δ]_2`
- Proper subgroup membership checks
- Non-degenerate pairing product target `R(vk,x)`

If PVUGC-010 regresses (CRS validation weakened), the PoCE-A soundness could be undermined. However, this is a **dependency**, not a regression in PVUGC-004 itself.

**Recommendation:** Validate PVUGC-010 to confirm CRS validation requirements are preserved.

---

## Conclusion

**Vote:** **PASS**

**Summary:** The PVUGC-2025-10-27.md specification (lines 274-278) maintains all cryptographic soundness properties of the Hardened PoCE-A Circuit. The mitigation that prevents the liveness griefing attack (PVUGC-004) is preserved with no regression detected.

**Regression Check Result:** ✅ **No Regression**

---

**Prepared by:** Crypto-Peer-Reviewer
**Date:** 2025-10-28 16:30
**Consultation ID:** PVUGC-004-regression-check
**File:** `/sandbox/report-compliance_audit-2025-10-29/.consultation-pvugc-004-crypto-response.md`
