# PVUGC-001: GT-XPDH Assumption Non-Standard

**Flaw Code:** PVUGC-001
**Severity:** 🔴 **CRITICAL**
**Status:** 🔓 Open (Mitigated by Multi-CRS AND-ing)
**Date Identified:** 2025-10-07 (v1.0)
**Last Updated:** 2025-10-07 (v2.0)

---

## Component
Core security foundation (KEM hardness assumption)

## Location
- **Section:** §7 (Security Analysis - Updated)
- **Lines:** 184-223 (v2.0 spec)
- **Related Sections:** §6 (Product-Key KEM), §89-91 (Multi-CRS AND-ing)

---

## Description

The PVUGC protocol's core security property ("no-proof-spend") relies on a novel cryptographic assumption that has not been peer-reviewed or reduced to known hardness problems. The assumption, now formalized as **GT-XPDH (External Power in 𝔾_T)** in v2.0:

> **Assumption (GT-XPDH):** Given random base sets {Uⱼ} ⊂ 𝔾₂, {Vₖ} ⊂ 𝔾₁, their r-powers {Uⱼ^r}, {Vₖ^r} for unknown r ← ℤ_r*, and an independent target T ∈ 𝔾_T, it is computationally hard to compute T^r.

This assumption is the foundation of the "No-Proof-Spend" property, which claims that without a valid attestation, computing M = G_G16^ρ from published masks {D₁,ⱼ = Uⱼ^ρ} and {D₂,ₖ = Vₖ^ρ} is infeasible.

---

## What Changed in v2.0

### ✅ Major Improvements

**1. Multi-CRS AND-ing (MANDATORY)**
- **v1.0:** Single CRS (single point of failure)
- **v2.0:** Minimum 2 independent CRS transcripts required (§89-91)
- **Impact:** Attacker must break GT-XPDH for **ALL** CRS simultaneously (exponential hardening)

**2. Formal Multi-Instance Analysis**
- **v1.0:** No analysis of multi-instance security
- **v2.0:** q-GT-XPDH analysis (§7, lines 218-223)
- **Impact:** Security degrades gracefully: breaking q instances is q times harder (in generic group model)

**3. Generic Group Security Argument**
- **v1.0:** Only informal hardness claim
- **v2.0:** Generic group heuristic argument provided (§7, lines 192-205)
- **Impact:** Provides theoretical foundation (though not a formal proof)

**4. Production Profile Normalization**
- **v1.0:** No specific curve mandated
- **v2.0:** BLS12-381 mandatory (§89)
- **Impact:** Allows focused cryptanalysis on specific instantiation

### 🔧 Remaining Gaps

- Still no reduction to standard assumptions (SXDH, DLIN, CDH)
- Generic group argument is heuristic, not a proof
- No peer-reviewed cryptanalysis yet
- Independence of G_G16 from {Uⱼ, Vₖ} still unproven (see PVUGC-003)

---

## Security Impact

**Consequence if assumption is false:** An attacker could spend Bitcoin without satisfying the computation predicate, completely breaking the protocol's security.

### Why This Remains Critical (Despite Mitigation)

1. **Foundation of entire protocol:** The ability to gate Bitcoin spends on proof existence depends entirely on this assumption
2. **No fallback security:** If this assumption breaks for all CRS, there is no secondary defense
3. **Unvalidated in literature:** Not reducible to DDH, CDH, SXDH, DLIN, or any standard pairing assumption
4. **Novel problem structure:** Adversary has exponentiations of *two* base sets (across different groups 𝔾₁, 𝔾₂) and must compute target in 𝔾_T

### Multi-CRS Mitigation Strength

**Before v2.0 (Single CRS):** Attack requires breaking GT-XPDH once.

**After v2.0 (Multi-CRS AND-ing):** Attack requires:
- Breaking GT-XPDH for CRS #1 **AND**
- Breaking GT-XPDH for CRS #2 **AND**
- ... (for all CRS transcripts)

**Security amplification:** If GT-XPDH has security level λ, multi-CRS provides security ≈ n·λ where n = number of CRS (under independence assumption).

**Example:**
- 2 CRS: 256-bit security (if GT-XPDH provides 128-bit per instance)
- 3 CRS: 384-bit security

**Critical assumption:** CRS transcripts must be **independently generated** and **binding** (see PVUGC-010, now resolved).

---

## Specific Concerns

### 1. Hidden Algebraic Structure (Unchanged)

**Problem:** The protocol operates in a bilinear pairing setting where algebraic relations may exist between:
- **G_G16(vk, x)** (the target) - defined as:
  ```
  G_G16 = e([α]₁, [β]₂) · e(Σᵢ xᵢ[lᵢ]₁, [γ]₂)
  ```
  Where [α]₁, [β]₂, [lᵢ]₁, [γ]₂ are Groth16 CRS elements

- **{Uⱼ, Vₖ}** (the bases) - derived from the Groth-Sahai CRS

**Concern:** Both G_G16 and {Uⱼ, Vₖ} involve elements from pairing-friendly curve CRS structures. There may be computable relations that allow deriving G_G16^r from {Uⱼ^r, Vₖ^r} without breaking discrete log.

**v2.0 Mitigation:** Multi-CRS means attacker must find such relations for **all** CRS simultaneously. If even one CRS lacks exploitable structure, the AND construction holds.

### 2. Generic Group Argument Limitations

**v2.0 Addition:** Spec now includes generic group argument (§7, lines 192-205):
- In the generic group model, no polynomial-time algorithm can distinguish GT-XPDH instance from random
- Provides heuristic evidence of hardness

**Limitations:**
- Generic group model is idealized (real groups may have exploitable structure)
- Not a formal proof (similar arguments have failed for multilinear maps - GGH15, CLT13)
- Specific curves (BLS12-381) may have properties not captured by generic model

### 3. CRS Independence Requirements

**v2.0 Requirement (§89):**
> "Production MUST use at least 2 independently generated binding GS-CRS transcripts."

**Security depends on:**
- CRS generated via different random tapes (no shared randomness)
- CRS generators don't collude
- Both CRS are truly binding (not witness-indistinguishable)

**Risk:** If CRS share hidden structure or are maliciously correlated, multi-instance security may not hold.

### 4. Comparison to Known Assumptions (Unchanged)

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

## Attack Vectors (Updated for v2.0)

### Attack 1.1: Algebraic Relation Exploitation (Hardened)

**Scenario:** Attacker discovers that G_G16 can be expressed using {Uⱼ, Vₖ}

```
Preconditions:
- Attacker analyzes Groth16 CRS structure for vk
- Attacker analyzes **all** GS CRS structures (CRS #1, CRS #2, ...)
- Discovers relation: G_G16 = f({Uⱼ}, {Vₖ}) for **every CRS**

Attack steps:
1. For CRS #1: Compute M₁ = f({D₁,ⱼ⁽¹⁾}, {D₂,ₖ⁽¹⁾})
2. For CRS #2: Compute M₂ = f({D₁,ⱼ⁽²⁾}, {D₂,ₖ⁽²⁾})
3. Combined key material: K_i = HKDF(ctx, ser(M₁) || ser(M₂), ...)
4. Decrypt ct_i, extract s_i
5. Sum: α = Σᵢ s_i
6. Finalize adaptor: s = s' + α
7. Broadcast Bitcoin transaction

Result: Complete break (but requires breaking ALL CRS)
```

**v2.0 Defense:** Exponentially harder. If finding relation has probability ε per CRS:
- Single CRS: Success probability = ε
- Multi-CRS (n=2): Success probability = ε²
- Multi-CRS (n=3): Success probability = ε³

### Attack 1.2: CRS Trapdoor Exploitation (Partially Mitigated)

**Scenario:** Attacker with CRS trapdoor knowledge computes M directly

```
Preconditions (v2.0):
- Attacker knows trapdoors for **all** GS CRS transcripts
- Or attacker compromises **all** CRS ceremonies

Attack steps:
1. Using trapdoors, derive M₁, M₂ for all CRS
2. Compute combined KDF as in Attack 1.1
3. Decrypt and finalize signature

Result: Break if **all** CRS ceremonies compromised
```

**v2.0 Defense:** Requires compromising multiple independent ceremonies. If CRS generated by different parties:
- Single malicious ceremony participant: Attack fails (honest CRS blocks it)
- All participants collude: Attack succeeds (but detectable via ceremony transcripts)

### Attack 1.3: Adaptive Target Selection (Unchanged)

**Scenario:** Attacker adaptively chooses (vk, x) after observing CRS to create favorable algebraic structure

```
Preconditions:
- All GS CRS generated first
- Attacker can choose x (public input) adaptively

Attack steps:
1. Observe all GS CRS bases
2. Search for x such that G_G16(vk, x) has special form for **all CRS**
3. Exploit favorable structure

Result: Partial break (requires adaptive input choice and finding favorable x for all CRS)
```

**v2.0 Defense:** Finding favorable x becomes exponentially harder with multiple CRS (must satisfy constraints for all).

---

## Recommendations (Updated for v2.0)

### Phase 1: Critical Path to Mainnet (2-3 months)

#### 1. Formal Cryptanalysis (HIGHEST PRIORITY)
**Owner:** External pairing-crypto experts
**Timeline:** 2-4 months
**Deliverable:** Formal analysis report

Tasks:
- [ ] Engage 3+ independent cryptography researchers specializing in pairing-based cryptography
- [ ] Attempt reduction of GT-XPDH to known assumptions (SXDH, DLIN, CDH)
- [ ] If reduction impossible, attempt to construct attack exploiting algebraic structure
- [ ] Analyze BLS12-381 specific instantiation for hidden relations
- [ ] Verify multi-instance security (q-GT-XPDH) holds as claimed
- [ ] Analyze whether G_G16 can be expressed in span of {e(Vₖ, Uⱼ)}

#### 2. Multi-CRS Security Validation
**Owner:** Cryptography team
**Timeline:** 1 month
**Deliverable:** Security proof or counterexample

Tasks:
- [ ] Formalize multi-instance security game
- [ ] Prove (or disprove) that breaking n-CRS AND-ing requires breaking all n instances
- [ ] Verify independence assumption: CRS correlation doesn't weaken security
- [ ] Analyze attack complexity reduction (if any) from multi-instance structure

#### 3. Generic Group Model Formalization
**Owner:** Research team + external reviewers
**Timeline:** 1-2 months
**Deliverable:** Formal proof or limitations analysis

Tasks:
- [ ] Formalize generic group argument from spec (§7, lines 192-205)
- [ ] Identify gap between generic model and BLS12-381 reality
- [ ] Compare to historical failures (GGH15, CLT13 multilinear maps)
- [ ] Establish security level estimates (e.g., "128-bit security in generic group model")

### Phase 2: Alternative Construction Research (Parallel)

#### 4. Explore Standard-Assumption Alternatives
**Owner:** Research team
**Timeline:** 2-3 months (parallel with cryptanalysis)

If GT-XPDH proves problematic, explore:
- [ ] KZG-based witness encryption (BCGJM23) - different non-standard assumption
- [ ] Lockable Obfuscation (BL20) - reduces to iO (also non-standard but more studied)
- [ ] Hybrid approach: Combine WE with fraud proofs or optimistic verification
- [ ] Time-lock puzzles + MPC for conditional signature release

### Phase 3: CRS Ceremony Design (If Proceeding)

#### 5. Multi-Party CRS Generation Ceremony
**Owner:** Protocol team + ceremony coordinators
**Timeline:** 1-2 months
**Deliverable:** Publicly auditable CRS transcripts

Requirements:
- [ ] Minimum 2 independent ceremonies (for 2-CRS minimum)
- [ ] Different participant sets for each ceremony (no overlap preferred)
- [ ] Public ceremony transcripts with verifiable randomness
- [ ] Binding verification tools (verify CRS is binding, not WI)
- [ ] Digest pinning in protocol (both CRS digests in GS_instance_digest)

---

## Acceptance Criteria for Resolution

This flaw can be considered resolved when **ONE** of the following is achieved:

### Option A: Formal Reduction (Preferred)
- [ ] Peer-reviewed paper proving GT-XPDH reduces to a standard assumption (SXDH, DLIN, etc.)
- [ ] Reduction is tight (polynomial security loss)
- [ ] Multi-instance security (q-GT-XPDH) formally proven
- [ ] Accepted at reputable cryptography venue (CRYPTO, EUROCRYPT, TCC, etc.)

### Option B: Extensive Cryptanalysis (Acceptable with Multi-CRS)
- [ ] 3+ independent cryptography teams analyze assumption (6+ months)
- [ ] All teams analyze both single-instance and multi-instance security
- [ ] All teams fail to construct attacks (for BLS12-381 and other curves)
- [ ] Published technical report with negative results
- [ ] Community consensus that assumption appears sound
- [ ] **Additional requirement for v2.0:** Verification that Multi-CRS AND-ing provides claimed security amplification

### Option C: Alternative Construction (Fallback)
- [ ] Protocol modified to use different, better-established assumptions
- [ ] New construction maintains same functionality (off-chain compute gating Bitcoin)
- [ ] Security proof available under standard assumptions

---

## Related Flaws
- **PVUGC-003:** Independence property (directly related - G_G16 independence from {Uⱼ, Vₖ})
- **PVUGC-010:** CRS validation (✅ resolved in v2.0 - binding CRS requirement added)

---

## References

### Witness Encryption Literature
- **GGSW13:** Garg et al., "Witness Encryption and its Applications" (STOC 2013)
- **BL20:** Brakerski & Lombardi, "Lockable Obfuscation" (FOCS 2020)
- **CVW18:** Chen et al., "Witness Encryption for Algebraic Languages" (CRYPTO 2018)
- **BCGJM23:** Boneh et al., "Witness Encryption from KZG" (2023)

### Pairing-Based Cryptography
- **Groth16:** Groth, "On the Size of Pairing-Based Non-interactive Arguments" (EUROCRYPT 2016)
- **GS Proofs:** Groth & Sahai, "Efficient Non-interactive Proof Systems" (EUROCRYPT 2008)
- **Pairing assumptions:** Freeman, "Converting Pairing-Based Cryptosystems" (PKC 2010)

### Cautionary Examples (Non-Standard Assumptions)
- **GGH15:** Graph-induced multilinear maps (broken - CRYPTO 2016)
- **CLT13:** Coron-Lepoint-Tibouchi multilinear maps (multiple breaks)

---

## Notes

- **v2.0 Status:** Substantially mitigated by Multi-CRS AND-ing, but still requires formal validation
- **Priority:** Highest priority for external cryptanalysis
- **Mainnet readiness:** NOT READY until Option A or B achieved
- **Testnet:** Acceptable with clear warnings about unvalidated assumption
- **Recommended:** Public challenge/bounty for attack construction or reduction proof

---

**Version History:**
- **v1.0 (2025-10-07):** Initial identification (single CRS, critical unmitigated)
- **v2.0 (2025-10-07):** Updated with Multi-CRS mitigation analysis (critical but mitigated)

**Last Updated:** 2025-10-07
**Next Review:** After cryptanalysis results available (target: 3-6 months)
