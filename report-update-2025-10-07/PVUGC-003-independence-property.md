# PVUGC-003: Independence Property Needs Formal Proof

**Flaw Code:** PVUGC-003
**Severity:** 🔴 **CRITICAL**
**Status:** 🔧 Improved (MUST clause added, formal proof still needed)
**Date Identified:** 2025-10-07 (v1.0)
**Last Updated:** 2025-10-07 (v2.0)

---

## Component
CRS and base independence property (KEM security foundation)

## Location
- **v1.0:** §6, Lines 131-132 (informal claim)
- **v2.0:** §6, Lines 84-88 (MUST requirement with enhanced domain separation)

---

## Description

The protocol's security depends on the **independence** of the target G_G16(vk,x) from the base sets {Uⱼ(x), Vₖ(x)}. Specifically, G_G16 must not be algebraically correlated with the pairing span ⟨e(Vₖ(x), Uⱼ(x))⟩ beyond what's required by the GS verification equation.

**v1.0 claim (informal):**
> "The CRS- and input-dependent bases {Uⱼ(x), Vₖ(x)} and target G_G16(vk,x) are fixed by (CRS, vk, x) before any armer chooses randomness; armers cannot choose or influence these bases."

**v2.0 requirement (normative):**
> "The target G_G16(vk,x) ∈ 𝔾_T **MUST** be derived deterministically from (CRS,vk,x) alone and domain-separated from the derivations of {Uⱼ(x)} and {Vₖ(x)}."

> "G_G16 **MUST NOT** be algebraically correlated with the pairing span ⟨e(Vₖ(x), Uⱼ(x))⟩ beyond the GS verification equation."

---

## What Changed in v2.0

### ✅ Improvements

**1. Explicit MUST Clause (§6, lines 84-88)**
- **v1.0:** Informal "cannot choose" statement
- **v2.0:** Normative MUST requirement with specific constraints
- **Impact:** Makes independence a mandatory protocol property (not just design intent)

**2. Enhanced Domain Separation (§6, line 87)**
- **v1.0:** No specific domain separation mentioned
- **v2.0:** Explicit requirement for domain-separated derivations
- **Impact:** Provides implementation guidance for achieving independence

**3. Algebraic Correlation Prohibition (§6, line 88)**
- **v1.0:** No formal statement about algebraic correlation
- **v2.0:** Explicit prohibition: "MUST NOT be algebraically correlated with pairing span"
- **Impact:** Clarifies what independence means mathematically

**4. Production Profile Constraints (§89)**
- **v1.0:** No specific curve or setup mandated
- **v2.0:** BLS12-381 + specific domain tags + hash functions
- **Impact:** Enables focused analysis on concrete instantiation

**Domain tags (v2.0):**
```
"PVUGC/CTX_CORE" - for context core hash
"PVUGC/ARM"      - for arming package
"PVUGC/PRESIG"   - for pre-signature package
"PVUGC/CTX"      - for final context hash
"PVUGC/KEM/v1"   - for KDF info parameter
```

### 🔧 Remaining Gaps

**What's still missing:**
- ❌ No formal proof that G_G16 is independent from {Uⱼ, Vₖ}
- ❌ No analysis of domain separation sufficiency for BLS12-381
- ❌ No setup ceremony specification ensuring independence
- ❌ No computational verification that span independence holds

**Why this remains CRITICAL:**
If independence doesn't hold, attackers might compute M = G_G16^ρ directly from masks, breaking the entire protocol.

---

## Security Impact

**Consequence if independence fails:** An attacker could compute M without a valid proof, completely breaking the no-proof-spend property.

### Why This Is Critical

1. **GT-XPDH assumption depends on it:** The hardness of computing G_G16^ρ from {Uⱼ^ρ, Vₖ^ρ} assumes G_G16 cannot be expressed using {Uⱼ, Vₖ}
2. **Multi-CRS doesn't help if correlation exists:** If algebraic relation exists, it likely exists for all CRS
3. **Foundation of KEM security:** If G_G16 ∈ span(e(Vₖ, Uⱼ)), the Product-Key KEM is broken
4. **No runtime detection:** Independence violation may not be detectable during execution

---

## Specific Concerns

### 1. Derivation from Common CRS Structure

**Problem:** Both G_G16 and {Uⱼ, Vₖ} derive from pairing-friendly curve CRS:

**G_G16 derivation (Groth16 verification key):**
```
G_G16(vk, x) = e([α]₁, [β]₂) · e(Σᵢ xᵢ[lᵢ]₁, [γ]₂)
```
Where [α]₁, [β]₂, [lᵢ]₁, [γ]₂ are Groth16 CRS elements.

**{Uⱼ, Vₖ} derivation (GS-CRS):**
```
Uⱼ(x) ∈ 𝔾₂ (derived from GS-CRS basis vectors)
Vₖ(x) ∈ 𝔾₁ (derived from GS-CRS basis vectors)
```

**Concern:** Both use elements from the same pairing-friendly curve (BLS12-381 in v2.0). Structural correlations may exist.

### 2. Pairing Span Question

**Critical question:** Can G_G16 be expressed as:
```
G_G16 = ∏ᵢ,ⱼ e(Vₖᵢ, Uⱼ)^{cᵢⱼ}
```
for some coefficients {cᵢⱼ}?

**If YES:**
```
G_G16^ρ = ∏ᵢ,ⱼ e(Vₖᵢ, Uⱼ)^{ρ·cᵢⱼ}
        = ∏ᵢ,ⱼ e(Vₖᵢ^ρ, Uⱼ)^{cᵢⱼ}    [pairing bilinearity]
        = ∏ᵢ,ⱼ e(D₂,ₖᵢ, Uⱼ)^{cᵢⱼ}    [published masks]
```
Attacker computes M without proof!

**v2.0 requirement:** "MUST NOT be algebraically correlated" - but how to verify?

### 3. Setup Ceremony Ordering

**v1.0 gap:** No specification of setup order.

**v2.0 partial improvement:** MUST clause states "derived deterministically from (CRS,vk,x)" but doesn't specify:
- Order of CRS generation (Groth16 CRS, then GS-CRS, or vice versa?)
- Whether vk and x are committed before CRS generation
- How to prevent adaptive CRS/input selection

**Open questions:**
1. **If GS-CRS generated first:**
   - Bases {Uⱼ, Vₖ} are fixed
   - Can attacker then choose (vk, x) to create exploitable G_G16?
   - v2.0 includes G_G16 ≠ 1 check (line 164), but what about other special forms?

2. **If Groth16 CRS generated first:**
   - G_G16 structure determined by vk
   - Can attacker then influence GS-CRS to create exploitable {Uⱼ, Vₖ}?

3. **If both CRS generated independently:**
   - Most secure option
   - But how to ensure they remain independent?
   - Could coordinated CRS generators create subtle correlations?

### 4. Domain Separation Sufficiency

**v2.0 improvement:** Domain separation tags specified.

**Remaining question:** Is domain separation sufficient to guarantee algebraic independence?

**Domain tags in v2.0:**
- G_G16 involves Groth16 CRS (no explicit domain tag for derivation)
- {Uⱼ, Vₖ} involve GS-CRS (derived from CRS structure)
- No explicit domain tag in G_G16 derivation itself

**Concern:** Domain separation in hashing (for KDF, context binding) doesn't necessarily imply algebraic independence of curve points.

---

## Attack Vectors

### Attack 3.1: Algebraic Relation Discovery

**Scenario:** Attacker finds that G_G16 ∈ span(e(Vₖ, Uⱼ))

```
Preconditions:
- Attacker analyzes Groth16 CRS for specific vk
- Attacker analyzes GS-CRS structure
- Discovers coefficients {cᵢⱼ} such that:
  G_G16 = ∏ᵢ,ⱼ e(Vₖᵢ, Uⱼ)^{cᵢⱼ}

Attack steps:
1. Protocol publishes masks: D₁,ⱼ = Uⱼ^ρ, D₂,ₖ = Vₖ^ρ
2. Attacker computes:
   M = ∏ᵢ,ⱼ e(D₂,ₖᵢ, Uⱼ)^{cᵢⱼ}
     = ∏ᵢ,ⱼ e(Vₖᵢ^ρ, Uⱼ)^{cᵢⱼ}
     = (∏ᵢ,ⱼ e(Vₖᵢ, Uⱼ)^{cᵢⱼ})^ρ
     = G_G16^ρ
3. Derives K, decrypts ciphertext, extracts α
4. Finalizes signature without proof

Result: Complete break despite Multi-CRS (if relation exists for all CRS)
```

**v2.0 defense:** MUST clause prohibits this, but no proof it's impossible.

### Attack 3.2: Adaptive Setup Exploitation

**Scenario:** Attacker influences setup to create exploitable correlation

```
Preconditions:
- Attacker participates in CRS generation
- Or attacker chooses (vk, x) after observing CRS

Attack steps:
1. Observe GS-CRS bases {Uⱼ, Vₖ}
2. Search for vk or x such that resulting G_G16 has special form
3. Options:
   a) G_G16 expressible as simple product of pairings
   b) G_G16 in small subgroup
   c) G_G16 = 1 (caught by v2.0 check, line 164)
   d) G_G16 has other exploitable structure
4. Propose transaction with chosen parameters
5. Exploit structure to compute M

Result: Partial break (requires setup influence)
```

**v2.0 defense:**
- G_G16 ≠ 1 check (line 164)
- Subgroup membership check (line 165)
- But no protection against other special forms

### Attack 3.3: CRS Correlation

**Scenario:** Groth16 CRS and GS-CRS secretly correlated

```
Preconditions:
- Both CRS generated by same entity (or colluding entities)
- CRS generators share secret trapdoor or randomness

Attack steps:
1. CRS generators create Groth16 CRS with trapdoor τ₁
2. CRS generators create GS-CRS with related trapdoor τ₂
3. By design, G_G16 and {Uⱼ, Vₖ} have computable relation via τ₁, τ₂
4. CRS generators (or anyone who knows τ₁, τ₂) can compute M
5. Decrypt and spend without proof

Result: Break if CRS ceremonies collude
```

**v2.0 defense:**
- Multi-CRS (PVUGC-002) requires multiple independent ceremonies
- Different ceremony participants reduce collusion risk
- But doesn't guarantee independence

---

## Recommendations

### Phase 1: Formal Proof (HIGHEST PRIORITY)

#### 1. Prove Independence for BLS12-381
**Owner:** Cryptography researchers + external reviewers
**Timeline:** 2-4 months
**Deliverable:** Formal proof or counterexample

Tasks:
- [ ] Formalize independence property as mathematical statement
- [ ] For BLS12-381 specifically, prove (or disprove):
  ```
  G_G16(vk,x) ∉ span({e(Vₖ(x), Uⱼ(x))})
  except for trivial GS verification equation relation
  ```
- [ ] Analyze domain separation: Is it sufficient for algebraic independence?
- [ ] Consider all choices of (vk, x) - prove no exploitable instances exist
- [ ] Account for Multi-CRS: Prove independence holds for each CRS transcript

#### 2. Setup Ceremony Formalization
**Owner:** Protocol designers
**Timeline:** 2-4 weeks
**Deliverable:** Setup ceremony specification

Requirements:
- [ ] Specify exact ordering:
  1. Groth16 CRS generation (with vk)
  2. GS-CRS generation (independent ceremony)
  3. Public input x selection (after both CRS fixed)
  4. Arming begins
- [ ] Cryptographic commitments: Armers commit to participation before seeing CRS
- [ ] Independence verification: Ceremony transcripts prove no shared randomness
- [ ] Multiple CRS: Each GS-CRS ceremony independent (PVUGC-002)

#### 3. Computational Verification Tools
**Owner:** Implementation team
**Timeline:** 1 month
**Deliverable:** Verification tool + test suite

Tasks:
- [ ] Implement symbolic algebra tool to test span membership:
  - Input: (Groth16 CRS, GS-CRS, vk, x)
  - Output: Check if G_G16 ∈ span(e(Vₖ, Uⱼ))
  - Test across many random instances
- [ ] Groebner basis solver to search for algebraic relations
- [ ] Numerical experiments: Sample random (vk, x), verify independence
- [ ] Edge case testing: Boundary values, small subgroups, degenerate cases

### Phase 2: Specification Hardening

#### 4. Enhanced Domain Separation
**Owner:** Protocol designers
**Timeline:** 1-2 weeks
**Deliverable:** Updated specification

Improvements:
- [ ] Add explicit domain tag for G_G16 derivation (not just in hashes)
- [ ] Specify exact hash-to-curve or point derivation functions
- [ ] Ensure {Uⱼ, Vₖ} derivation uses different domain from G_G16
- [ ] Document why chosen separation prevents correlation

Example enhancement:
```
G_G16 derivation: Include domain tag in Groth16 CRS setup
Uⱼ derivation: H_to_G2("PVUGC/GS-U" || CRS || x || j)
Vₖ derivation: H_to_G1("PVUGC/GS-V" || CRS || x || k)
```

#### 5. Runtime Independence Checks
**Owner:** Implementation team
**Timeline:** 2 weeks
**Deliverable:** Verification functions

Implement checks during arming:
- [ ] G_G16 ≠ 1 (already in v2.0, line 164)
- [ ] G_G16 ∉ small subgroup (already in v2.0, line 165)
- [ ] G_G16 order verification: order(G_G16) = r (full group order)
- [ ] Statistical tests: G_G16 appears random (entropy checks)
- [ ] Reject if G_G16 has any exploitable structure (define "exploitable")

### Phase 3: Alternative Approaches (If Proof Fails)

#### 6. Separation-Based Construction
**Owner:** Research team
**Timeline:** 2-3 months (parallel track)

If independence cannot be proven, consider:
- [ ] **Option A:** Use completely separate CRS for Groth16 and GS
  - Generate Groth16 CRS on different curve than GS-CRS
  - Prevents any algebraic relation (but adds complexity)
- [ ] **Option B:** Hash-based isolation
  - Don't use GS-CRS directly for {Uⱼ, Vₖ}
  - Derive via: Uⱼ = H_to_G2(CRS || "U" || j)
  - Ensures no structural correlation
- [ ] **Option C:** Different KEM construction
  - If independence unprovable, redesign KEM to not require it
  - E.g., KZG-based witness encryption (different assumptions)

---

## Acceptance Criteria for Resolution

This flaw can be considered **RESOLVED** when:

### Option A: Formal Proof (Preferred)
- [ ] Peer-reviewed proof that G_G16 ∉ span({e(Vₖ, Uⱼ)}) for BLS12-381
- [ ] Proof covers all possible (vk, x) choices
- [ ] Proof accounts for domain separation as specified in v2.0
- [ ] Proof verified by 2+ independent cryptographers
- [ ] Published in technical report or academic venue

### Option B: Computational Evidence (Acceptable)
- [ ] Exhaustive testing: 10,000+ random (vk, x) samples, no correlation found
- [ ] Symbolic algebra tools: No algebraic relations discovered
- [ ] Multiple research teams attempt to find correlation (6+ months), all fail
- [ ] Expert consensus: Independence appears to hold for practical parameters
- [ ] Caveat documented: Based on computational evidence, not formal proof

### Option C: Construction Change (Fallback)
- [ ] Protocol modified to use provably independent G_G16 and {Uⱼ, Vₖ}
- [ ] New construction maintains same functionality
- [ ] Formal proof available for updated design

---

## Related Flaws

- **PVUGC-001:** GT-XPDH assumption (depends on independence - if independence fails, GT-XPDH is trivially broken)
- **PVUGC-002:** Multi-CRS AND-ing (✅ resolved - provides defense-in-depth, but doesn't guarantee independence)
- **PVUGC-010:** CRS validation (✅ resolved - ensures binding CRS, but doesn't prove independence)

---

## References

### Relevant Cryptography
- **Groth16:** Groth, "On the Size of Pairing-Based Non-interactive Arguments" (EUROCRYPT 2016)
- **GS Proofs:** Groth & Sahai, "Efficient Non-interactive Proof Systems" (EUROCRYPT 2008)
- **BLS12-381:** Bowe, "BLS12-381: New zk-SNARK Elliptic Curve Construction" (2017)

### Algebraic Independence
- **Schwartz-Zippel:** Polynomial identity testing
- **Groebner Basis:** Symbolic computation for algebraic relations
- **Pairing Span Analysis:** Freeman et al., "Converting Pairing-Based Cryptosystems" (PKC 2010)

---

## Notes

- **v2.0 Status:** Significantly improved with MUST clause and domain separation, but formal proof still required
- **Priority:** Critical - second only to GT-XPDH cryptanalysis (PVUGC-001)
- **Relationship to PVUGC-001:** If independence fails, GT-XPDH is broken (even with Multi-CRS)
- **Blocking issue:** Mainnet deployment should NOT proceed without formal proof or extensive computational evidence
- **Testnet:** Acceptable with warnings, use for gathering evidence

---

**Version History:**
- **v1.0 (2025-10-07):** Initial identification (informal claim, no verification)
- **v2.0 (2025-10-07):** Improved with MUST clause, domain separation, and checks (formal proof still pending)

**Last Updated:** 2025-10-07
**Next Review:** After formal proof attempt (target: 3-4 months)
