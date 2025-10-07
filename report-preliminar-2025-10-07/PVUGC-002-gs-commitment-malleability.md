# PVUGC-002: GS Attestation - Commitment Malleability

**Flaw Code:** PVUGC-002
**Severity:** 🔴 **CRITICAL**
**Status:** 🔓 Open
**Date Identified:** 2025-10-07

---

## Component
Groth-Sahai attestation layer (witness encryption construction)

## Location
- **Section:** §6 (WE via Product-Key KEM)
- **Lines:** 118-131 (GS PPE verification)
- **Related:** §7 Lemma 1 (GS product determinism), Line 190 (CRS binding requirement)

---

## Description

The protocol wraps Groth16 verification in a Groth-Sahai proof system to enable witness encryption. The security property "every valid attestation for fixed (vk,x) yields the same product G_G16" depends critically on GS commitment binding.

The GS verifier reduces to a product-of-pairings equation (GS-PPE):
```
∏ⱼ e(C¹ⱼ, Uⱼ(x)) · ∏ₖ e(Vₖ(x), C²ₖ) = G_G16(vk,x)
```

where C¹ⱼ, C²ₖ are commitments to Groth16 proof elements (π_A, π_B, π_C).

**The concern:** An attacker might craft commitments that pass GS verification and produce the target product G_G16, but don't correspond to any valid underlying Groth16 proof.

---

## Security Impact

**If exploited:** Attacker can decapsulate KEM keys and complete Bitcoin signatures without satisfying the computation predicate, completely breaking the protocol's core security property.

**Attack enables:**
- Spending Bitcoin without valid proof
- Bypassing the "proof existence" requirement
- Rendering the witness encryption ineffective

---

## Specific Concerns

### 1. Commitment Crafting / Product Forcing

**Problem:** Can an attacker construct commitments {C¹ⱼ, C²ₖ} such that:
- The PPE equation verifies: `∏ e(C¹ⱼ, Uⱼ) · ∏ e(Vₖ, C²ₖ) = G_G16` ✓
- But the commitments don't open to valid Groth16 proof elements (π_A, π_B, π_C)

**Why this matters:** The protocol never explicitly extracts or verifies the underlying Groth16 proof elements. It only checks the GS attestation (PPE equation). If commitments can be crafted to satisfy the equation without valid underlying proof, the gating mechanism fails.

**GS soundness question:** Does binding CRS guarantee that only commitments opening to valid (π_A, π_B, π_C) can satisfy the PPE equation?

### 2. Re-randomization Properties

**Issue:** GS commitments can be re-randomized. Line 176 claims:
> "randomizers cancel in the verifier's bilinear equations"

**Verification needed:** This must hold for the *specific* PPE layout used. Different GS proof structures have different re-randomization properties. The protocol doesn't specify:
- Exact GS commitment scheme variant
- How re-randomization is handled
- Whether re-randomization could be exploited

### 3. CRS Binding Requirement

**Line 190 states:**
> "CRS Requirement: GS must use a binding CRS (SXDH/DLIN). If not binding (e.g., WI/hiding), Lemma 1 may fail."

**Concerns:**
- **Runtime validation:** "Runtime MUST reject non-binding CRS via a tag embedded in GS_instance_digest"
  - Tag format unspecified (see PVUGC-010)
  - Validation procedure undefined
  - Could malicious CRS with forged tag pass validation?

- **CRS trust:** Who generates the GS CRS?
  - If coordinator generates CRS, they might know trapdoor
  - With trapdoor, can create commitments to arbitrary values
  - Could forge attestations without valid proof

- **Binding vs. WI tradeoff:** Binding CRS doesn't provide witness hiding. Is zero-knowledge preserved? If GS CRS is binding but not hiding, does Groth16's zero-knowledge property still hold at the attestation layer?

### 4. Specific PPE Structure

**Unknown:** The protocol doesn't specify the exact GS PPE layout:
- How many commitments (m₁, m₂)?
- Which Groth16 elements committed in 𝔾₁ vs 𝔾₂?
- Are all proof elements committed, or only some?
- Single-sided vs. two-sided PPE?

**Risk:** Different layouts have different security properties. Without specification, implementations may diverge or choose insecure layouts.

---

## Attack Vectors

### Attack 2.1: Algebraic Commitment Crafting

```
Preconditions:
- Attacker understands exact PPE structure
- Finds algebraic method to satisfy PPE without valid proof

Attack steps:
1. Target: Need commitments such that ∏ e(C¹ⱼ, Uⱼ) · ∏ e(Vₖ, C²ₖ) = G_G16
2. Craft C¹ⱼ, C²ₖ algebraically to satisfy equation
   - Example: Choose C¹₁ s.t. e(C¹₁, U₁) = G_G16 and set others to identity
   - Or distribute product across multiple commitments
3. Verify crafted attestation passes GS verification
4. Use attestation to decapsulate: compute M = ∏ e(C¹ⱼ, D₁,ⱼ) · ∏ e(D₂,ₖ, C²ₖ)
5. Derive K, decrypt α, finalize signature
6. Spend Bitcoin without valid Groth16 proof

Result: Complete break of witness encryption
```

### Attack 2.2: CRS Trapdoor Exploitation

```
Preconditions:
- Attacker controls or influences GS CRS generation
- Knows CRS trapdoor

Attack steps:
1. Generate malicious GS CRS with known trapdoor
2. CRS still appears "binding" (passes structure checks)
3. Include forged "binding tag" in GS_instance_digest
4. During protocol execution, create commitments using trapdoor:
   - Commitments satisfy PPE equation
   - But don't correspond to valid Groth16 proof
5. Publish as valid attestation
6. Decapsulate and spend as in Attack 2.1

Result: Coordinator can spend without proof; insider attack
```

### Attack 2.3: Re-randomization Exploitation

```
Preconditions:
- Attacker observes valid attestation for (vk, x)
- Finds that re-randomization doesn't fully cancel in PPE

Attack steps:
1. Take valid attestation: {C¹ⱼ, C²ₖ}
2. Re-randomize commitments: C'¹ⱼ = C¹ⱼ · r_j, C'²ₖ = C²ₖ · s_k
3. If re-randomization factors don't fully cancel, product changes:
   ∏ e(C'¹ⱼ, Uⱼ) · ∏ e(Vₖ, C'²ₖ) ≠ G_G16
4. Attacker chooses randomizers to force product to equal G_G16 for different x'
5. Creates valid-looking attestation for x' without proof for x'

Result: Cross-context attestation forgery
```

---

## Recommendations

### Immediate Actions

#### 1. Formal GS Soundness Verification
**Priority:** P0 (Critical)
**Timeline:** 2-4 weeks

Tasks:
- [ ] Specify exact GS PPE layout used:
  - Number of commitments m₁ (𝔾₁) and m₂ (𝔾₂)
  - Mapping: which Groth16 elements (π_A, π_B, π_C) committed where
  - Commitment scheme variant (SXDH-based, DLIN-based, etc.)
- [ ] Prove: For this specific layout, GS soundness + binding CRS implies:
  - Only commitments opening to valid (π_A, π_B, π_C) can satisfy PPE
  - Valid (π_A, π_B, π_C) must satisfy Groth16 verification equation
  - Therefore: GS attestation passes ⟹ valid Groth16 proof exists
- [ ] Verify re-randomization properties for chosen layout
- [ ] Engage GS proof system experts for review

#### 2. CRS Generation and Validation Procedure
**Priority:** P0 (Critical)
**Timeline:** 1-2 weeks

Tasks:
- [ ] Specify exact GS CRS generation algorithm (see PVUGC-010)
- [ ] Define "binding tag" format and verification
- [ ] Specify CRS setup ceremony:
  - Who generates GS CRS?
  - Multi-party computation for trapdoor-free generation?
  - Public verifiability of CRS correctness?
- [ ] Add CRS hash to ctx_core (see PVUGC-005)
- [ ] Implement CRS validation at runtime with test vectors

#### 3. Reference Implementation
**Priority:** P1 (High)
**Timeline:** 2-3 weeks

Tasks:
- [ ] Implement exact GS attestation layer with specified layout
- [ ] Test vectors: (vk, x, π) → GS attestation → product verification
- [ ] Negative tests: Invalid proofs should fail GS attestation
- [ ] Attack simulation: Attempt commitment crafting (should fail)

### Specification Improvements

#### 4. Detailed GS Integration Specification
**Timeline:** 1 week

Add to protocol document:
```markdown
## GS Attestation Specification

### Layout
- Commit π_A ∈ 𝔾₁ using commitment scheme C¹₁
- Commit π_C ∈ 𝔾₁ using commitment scheme C¹₂
- Commit π_B ∈ 𝔾₂ using commitment scheme C²₁
- Total: m₁ = 2 commitments in 𝔾₁, m₂ = 1 commitment in 𝔾₂

### PPE Equation
[Exact equation with indices specified]

### CRS Structure
[Precise CRS generation algorithm]

### Verification Algorithm
[Step-by-step GS attestation verification]
```

#### 5. Zero-Knowledge Analysis
**Timeline:** 2 weeks

Tasks:
- [ ] Verify: Does binding GS CRS preserve Groth16's zero-knowledge?
- [ ] If not: Specify privacy leakage (acceptable or needs mitigation?)
- [ ] Document: What information does attestation reveal beyond "proof exists"?

### Testing & Validation

#### 6. Security Testing Suite
**Timeline:** 3-4 weeks

Tests:
- [ ] **Soundness tests:**
  - Invalid Groth16 proof → GS attestation must fail
  - Forged commitments → PPE equation must not verify
  - Crafted commitments → should be rejected

- [ ] **Determinism tests:**
  - Two valid proofs for (vk, x) → same product G_G16
  - Re-randomized attestations → same product

- [ ] **Malicious CRS tests:**
  - Non-binding CRS → must be rejected at runtime
  - CRS with wrong structure → must be rejected

- [ ] **Cross-implementation tests:**
  - Multiple GS libraries should produce compatible attestations
  - Verify interoperability

---

## Acceptance Criteria for Resolution

This flaw can be considered resolved when **ALL** of the following are achieved:

- [ ] **Exact GS PPE layout specified** with formal justification
- [ ] **Soundness proof:** GS attestation passes ⟹ valid Groth16 proof exists
- [ ] **CRS generation procedure** specified and implemented
- [ ] **Runtime CRS validation** implemented with binding verification
- [ ] **Reference implementation** with passing test suite
- [ ] **External review:** GS proof system expert validates approach
- [ ] **Zero-knowledge preservation** verified or documented

---

## Related Flaws
- **PVUGC-001:** Power-Target Hardness (KEM security also depends on this)
- **PVUGC-003:** Independence violation (affects CRS relationship)
- **PVUGC-010:** CRS validation (directly related to binding check)

---

## References

### Groth-Sahai Proofs
- **GS08:** Groth & Sahai, "Efficient Non-interactive Proof Systems for Bilinear Groups" (EUROCRYPT 2008)
- **EG14:** Escala & Groth, "Fine-Tuning Groth-Sahai Proofs" (PKC 2013) - re-randomization properties
- **AGOT14:** Abe et al., "Structure-Preserving Signatures from Type II Pairings" - GS applications

### Commitment Binding
- **SXDH:** Symmetric External Diffie-Hellman assumption
- **DLIN:** Decision Linear assumption (Boneh et al., CRYPTO 2004)

---

## Notes

- This is a **critical** finding but potentially resolvable with proper specification
- Unlike PVUGC-001 (novel assumption), GS proofs are well-studied
- Key task: Ensure specific instantiation maintains soundness
- May require consulting original GS proof authors or experts

---

**Last Updated:** 2025-10-07
**Next Review:** After GS layout specification and soundness proof
