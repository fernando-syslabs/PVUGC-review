# PVUGC-001: Non-Standard Cryptographic Assumption (Power-Target Hardness)

**Flaw Code:** PVUGC-001
**Severity:** 🔴 **CRITICAL**
**Status:** 🔓 Open
**Date Identified:** 2025-10-07

---

## Component
Core security foundation (KEM hardness assumption)

## Location
- **Section:** §7 (Security Analysis - Lemma 3)
- **Lines:** 186-187
- **Related Sections:** §6 (Product-Key KEM), §16 (Minimal equations)

---

## Description

The PVUGC protocol's core security property ("no-proof-spend") relies on a novel cryptographic assumption that has not been peer-reviewed or reduced to known hardness problems. The assumption, stated in §7, line 186:

> **Assumption (Power-Target Hardness in bilinear groups):** Given random base sets {Uⱼ} ⊂ 𝔾₂, {Vₖ} ⊂ 𝔾₁, their r-powers {Uⱼ^r}, {Vₖ^r} for unknown r ← ℤ_r*, and an independent target T ∈ 𝔾_T, it is hard to compute T^r.

This assumption is the foundation of Lemma 3 (No-Proof-Spend), which claims that without a valid attestation, computing M_i = G_G16^ρᵢ from published masks {D₁,ⱼ = Uⱼ^ρᵢ} and {D₂,ₖ = Vₖ^ρᵢ} is infeasible.

---

## Security Impact

**Consequence if assumption is false:** An attacker could spend Bitcoin without satisfying the computation predicate, completely breaking the protocol's security.

### Why This Is Critical

1. **Foundation of entire protocol:** The ability to gate Bitcoin spends on proof existence depends entirely on this assumption
2. **No fallback security:** If this assumption breaks, there is no secondary defense
3. **Unvalidated in literature:** Not reducible to DDH, CDH, SXDH, DLIN, or any standard pairing assumption
4. **Novel problem structure:** Adversary has exponentiations of *two* base sets (across different groups 𝔾₁, 𝔾₂) and must compute target in 𝔾_T

---

## Specific Concerns

### 1. Hidden Algebraic Structure

**Problem:** The protocol operates in a bilinear pairing setting where algebraic relations may exist between:
- **G_G16(vk, x)** (the target) - defined as:
  ```
  G_G16 = e([α]₁, [β]₂) · e(Σᵢ xᵢ[lᵢ]₁, [γ]₂)
  ```
  Where [α]₁, [β]₂, [lᵢ]₁, [γ]₂ are Groth16 CRS elements

- **{Uⱼ, Vₖ}** (the bases) - derived from the Groth-Sahai CRS

**Concern:** Both G_G16 and {Uⱼ, Vₖ} involve elements from pairing-friendly curve CRS structures. There may be computable relations that allow deriving G_G16^r from {Uⱼ^r, Vₖ^r} without breaking discrete log.

### 2. Target Expressibility

**Question:** Can G_G16 be expressed as a product of pairings involving {Uⱼ} and {Vₖ}?

If yes, then:
```
G_G16 = ∏ᵢ e(Aᵢ, Bᵢ)  where Aᵢ ∈ span({Uⱼ, Vₖ})
```

An adversary with {Uⱼ^r}, {Vₖ^r} could then compute:
```
G_G16^r = ∏ᵢ e(Aᵢ, Bᵢ)^r = ∏ᵢ e(Aᵢ^r, Bᵢ)
```

This would allow deriving M_i without a valid proof.

### 3. CRS Correlation

**Problem:** The document asserts independence (§6, lines 131-132):
> "The CRS- and input-dependent bases {Uⱼ(x), Vₖ(x)} and target G_G16(vk,x) are fixed by (CRS, vk, x) before any armer chooses randomness; armers cannot choose or influence these bases."

**Concern:** Independence is *claimed* but not *proven*. Both CRS structures (Groth16 and GS) derive from the same pairing-friendly curve setup. Structural correlations may exist regardless of setup ordering.

### 4. Comparison to Known Assumptions

**Standard assumptions this does NOT reduce to:**
- **DDH (Decisional Diffie-Hellman):** Only involves one group
- **CDH (Computational Diffie-Hellman):** Only involves computing g^(ab) from g^a, g^b in one group
- **SXDH (Symmetric External DH):** DDH hard in both 𝔾₁ and 𝔾₂, but doesn't cover 𝔾_T targets
- **DLIN (Decision Linear):** Linear problem in one group
- **q-SDH, q-BDHE:** Involve polynomial evaluations, not arbitrary targets

**Novel aspects:**
- Two base sets in *different* source groups (𝔾₁, 𝔾₂)
- Target in *output* group (𝔾_T)
- Target claimed "independent" but derived from related CRS

---

## Attack Vectors

### Attack 1.1: Algebraic Relation Exploitation

**Scenario:** Attacker discovers that G_G16 can be expressed using {Uⱼ, Vₖ}

```
Preconditions:
- Attacker analyzes Groth16 CRS structure for vk
- Attacker analyzes GS CRS structure
- Discovers relation: G_G16 = f({Uⱼ}, {Vₖ}) for some computable f

Attack steps:
1. Protocol publishes masks: D₁,ⱼ = Uⱼ^ρ, D₂,ₖ = Vₖ^ρ
2. Attacker computes: M = f({D₁,ⱼ}, {D₂,ₖ}) = f({Uⱼ^ρ}, {Vₖ^ρ}) = G_G16^ρ
3. Derives: K = HKDF(ctx_hash, ser(M), "PVUGC/KEM/v1")
4. Decrypts: ct → (s || h)
5. Sums: α = Σᵢ sᵢ
6. Finalizes adaptor: s = s' + α
7. Broadcasts Bitcoin transaction, spends without valid proof

Result: Complete break of no-proof-spend property
```

### Attack 1.2: CRS Trapdoor Exploitation

**Scenario:** Attacker with CRS trapdoor knowledge computes M directly

```
Preconditions:
- Attacker knows Groth16 CRS trapdoor (τ) OR GS CRS trapdoor
- Or attacker influences CRS generation (see PVUGC-003)

Attack steps:
1. Using trapdoor knowledge, derive relationship between:
   - G_G16 (involves e([τ·G]₁, [β]₂) components)
   - {Uⱼ, Vₖ} (derived from GS CRS with potential τ dependencies)
2. From published {D₁,ⱼ}, {D₂,ₖ}, compute M using trapdoor
3. Decrypt and finalize signature as in Attack 1.1

Result: Break if CRS setup is malicious or compromised
```

### Attack 1.3: Adaptive Target Selection

**Scenario:** Attacker adaptively chooses (vk, x) after observing CRS to create favorable algebraic structure

```
Preconditions:
- GS CRS generated first (contains {Uⱼ, Vₖ})
- Attacker can choose x (public input) adaptively
- Or attacker influences vk selection

Attack steps:
1. Observe GS CRS bases {Uⱼ, Vₖ}
2. Search for x such that G_G16(vk, x) has special form:
   - G_G16 expressible as product involving {Uⱼ, Vₖ}
   - Or G_G16 = 1 (degenerate, line 146 attempts to prevent)
   - Or G_G16 has small order
3. Propose transaction with chosen x
4. Exploit favorable structure as in Attack 1.1

Result: Partial break (requires adaptive input choice)
```

---

## Recommendations

### Immediate Actions (Before Any Further Development)

#### 1. Formal Cryptanalysis
**Owner:** External pairing-crypto experts
**Timeline:** 2-4 months
**Deliverable:** Formal analysis report

Tasks:
- [ ] Engage 2-3 independent cryptography researchers specializing in pairing-based cryptography
- [ ] Attempt reduction of Power-Target Hardness to known assumptions (DDH, SXDH, etc.)
- [ ] If reduction impossible, attempt to construct attack exploiting algebraic structure
- [ ] Analyze specific instantiation (e.g., BLS12-381) for hidden relations
- [ ] Verify independence of G_G16 from {Uⱼ, Vₖ} in the chosen pairing group

#### 2. Attack Construction Attempts
**Owner:** Research team + adversarial consultants
**Timeline:** 1-2 months
**Deliverable:** Either (a) successful attack demo, or (b) hardness evidence report

Tasks:
- [ ] Symbolic computation: Express G_G16 in terms of potential {Uⱼ, Vₖ} relations
- [ ] Computational search: Test if G_G16^ρ computable from {Uⱼ^ρ, Vₖ^ρ} for sample parameters
- [ ] Lattice-based approaches: Check if linear combinations in exponent space solve problem
- [ ] Groebner basis analysis: Algebraic system solving for M given {D₁,ⱼ}, {D₂,ₖ}

#### 3. Literature Review
**Owner:** Research team
**Timeline:** 2 weeks
**Deliverable:** Survey document

Tasks:
- [ ] Search for related assumptions in bilinear group literature
- [ ] Review witness encryption constructions (GGSW13, BL20, CVW18, BCGJM23)
- [ ] Consult with authors of related work
- [ ] Check if similar assumptions broken in past (e.g., multilinear map attacks)

### Specification Improvements

#### 4. Assumption Formalization
**Owner:** Protocol designers
**Timeline:** 1 week

Update specification with:
- [ ] Precise mathematical formulation of assumption (currently somewhat informal)
- [ ] Explicit parameter choices (curve, group orders, etc.)
- [ ] Security level claims (e.g., "128-bit security assuming Power-Target Hardness")
- [ ] Comparison table with standard assumptions
- [ ] Acknowledgment of non-standard nature and need for further analysis

#### 5. Alternative Construction Research
**Owner:** Research team
**Timeline:** 1-2 months (parallel with cryptanalysis)

Explore alternatives if assumption proves problematic:
- [ ] KZG-based witness encryption (BCGJM23) - also non-standard but different structure
- [ ] Lockable Obfuscation (BL20) - reduces to iO (also non-standard but more studied)
- [ ] Modified GS attestation layer with different KEM approach
- [ ] Hybrid approach: Combine with different security mechanisms (e.g., fraud proofs)

### Testing & Validation

#### 6. Computational Experiments
**Owner:** Implementation team
**Timeline:** 2-4 weeks

Tasks:
- [ ] Implement KEM with multiple pairing libraries (BLS12-381, BN254)
- [ ] Measure actual computational cost of attack attempts
- [ ] Fuzz testing: Random (vk, x) pairs to find edge cases where assumption might fail
- [ ] Check for small subgroup or degenerate cases where M easily computable

---

## Acceptance Criteria for Resolution

This flaw can be considered resolved when **ONE** of the following is achieved:

### Option A: Formal Reduction (Preferred)
- [ ] Peer-reviewed paper proving Power-Target Hardness reduces to a standard assumption (SXDH, DLIN, etc.)
- [ ] Reduction is tight (polynomial security loss)
- [ ] Accepted at reputable cryptography venue (CRYPTO, EUROCRYPT, TCC, etc.)

### Option B: Extensive Cryptanalysis (Acceptable)
- [ ] 3+ independent cryptography teams analyze assumption (6+ months)
- [ ] All teams fail to construct attacks
- [ ] Analysis covers multiple pairing curves and parameter choices
- [ ] Published technical report with negative results (no attack found)
- [ ] Community consensus that assumption appears sound (similar to RSA assumption acceptance process)

### Option C: Alternative Construction (Fallback)
- [ ] Protocol modified to use different, better-established assumptions
- [ ] New construction maintains same functionality (off-chain compute gating Bitcoin)
- [ ] Security proof available under standard assumptions

---

## Related Flaws
- **PVUGC-002:** GS commitment malleability (also affects KEM security)
- **PVUGC-003:** Independence violation (directly related to target vs. base independence)

---

## References

### Witness Encryption Literature
- **GGSW13:** Garg et al., "Witness Encryption and its Applications" (STOC 2013) - original WE definition
- **BL20:** Brakerski & Lombardi, "Lockable Obfuscation" (FOCS 2020) - iO-based WE
- **CVW18:** Chen et al., "Witness Encryption for Algebraic Languages" (CRYPTO 2018)
- **BCGJM23:** Boneh et al., "Witness Encryption from KZG" (2023) - also uses non-standard assumption

### Pairing-Based Cryptography
- **Groth16:** Groth, "On the Size of Pairing-Based Non-interactive Arguments" (EUROCRYPT 2016)
- **GS Proofs:** Groth & Sahai, "Efficient Non-interactive Proof Systems" (EUROCRYPT 2008)
- **Pairing assumptions:** Freeman, "Converting Pairing-Based Cryptosystems" (PKC 2010)

### Historical Assumption Breaks
- **Multilinear maps:** Multiple breaks (CHL+15, HJ16, etc.) - cautionary tale for novel assumptions
- **GGH15:** Graph-induced multilinear maps broken (CRYPTO 2016)

---

## Notes

- This is the **highest priority** finding in the entire report
- Production deployment absolutely **MUST NOT** proceed without addressing this issue
- Even testnet deployment should include clear warnings about unvalidated assumption
- Consider public challenge: Offer bounty for attack construction or reduction proof

---

**Last Updated:** 2025-10-07
**Next Review:** After cryptanalysis results available
