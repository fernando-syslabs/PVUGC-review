# PVUGC-003: Independence Claim Violation

**Flaw Code:** PVUGC-003
**Severity:** 🔴 **CRITICAL**
**Status:** 🔓 Open
**Date Identified:** 2025-10-07

---

## Component
CRS independence property (setup and structural dependencies)

## Location
- **Section:** §6 (WE via Product-Key KEM)
- **Lines:** 131-132 (Independence MUST NOT clause)
- **Related:** Lines 186-187 (Power-Target Hardness independence claim)

---

## Description

The protocol states (§6, lines 131-132):

> **Independence (MUST).** The CRS- and input-dependent bases {Uⱼ(x), Vₖ(x)} and the target G_G16(vk,x) are fixed by (CRS, vk, x) before any armer chooses randomness; armers cannot choose or influence these bases.

And further:

> **Implementation MUST NOT** allow armers to influence CRS selection, vk, or x once arming begins; these are fixed before any ρᵢ is chosen.

**The problem:** Independence is *asserted* but not *proven* or *enforced* cryptographically. Both G_G16 and {Uⱼ, Vₖ} derive from related pairing-friendly curve structures, and the protocol doesn't specify how independence is guaranteed in practice.

---

## Security Impact

**If independence fails:**
1. Armers could exploit correlations between G_G16 and {Uⱼ, Vₖ}
2. Could enable computing M without valid proof (breaks PVUGC-001 assumption)
3. Could break determinism (different proofs yield different keys)
4. Could allow adaptive attacks where armers choose parameters to their advantage

---

## Specific Concerns

### 1. Structural Correlation

**Problem:** G_G16 and {Uⱼ, Vₖ} both derive from pairing-friendly curve elements:

**G_G16 construction:**
```
G_G16(vk, x) = e([α]₁, [β]₂) · e(Σᵢ xᵢ[lᵢ]₁, [γ]₂)
```
Where [α]₁, [β]₂, [lᵢ]₁, [γ]₂ are Groth16 CRS elements.

**{Uⱼ, Vₖ} construction:**
Derived from GS CRS, which is built over the same pairing-friendly curve.

**Question:** Even if setup ordering is correct, are there *inherent algebraic relations* between:
- Elements of Groth16 CRS (which define G_G16)
- Elements of GS CRS (which define {Uⱼ, Vₖ})

Both CRS structures use the same base groups (𝔾₁, 𝔾₂, 𝔾_T) and pairing. Structural correlations may exist regardless of setup ordering.

### 2. Setup Ceremony Ordering

**Current specification:** "Fixed by (CRS, vk, x) before any armer chooses randomness"

**Unclear:**
- What is the exact ordering of CRS generation, parameter selection, and armer participation?
- Who generates which CRS (Groth16 vs. GS)?
- When is x (public input) chosen relative to CRS generation?
- When can armers first observe these values?

**Risks by setup phase:**

**Phase 1: Groth16 CRS generation**
- Generates vk (verification key)
- Defines [α]₁, [β]₂, [lᵢ]₁, [γ]₂

**Phase 2: GS CRS generation**
- Defines {Uⱼ, Vₖ} bases
- **Risk:** If GS CRS generator knows Groth16 CRS, could they create correlated structure?

**Phase 3: Public input x selection**
- x determines part of G_G16 (via Σᵢ xᵢ[lᵢ]₁)
- **Risk:** If armers observe CRS before x is fixed, could they influence x choice?

**Phase 4: Armer participation begins**
- Armers choose ρᵢ
- **Risk:** If armers observe (CRS, vk, x) early, could they choose ρᵢ to exploit correlations?

### 3. CRS Relationship

**Groth16 CRS and GS CRS:**
- Are they generated independently?
- Do they share randomness or structure?
- Is the same trusted setup used for both?

**If yes (shared setup):**
- High correlation risk
- Trapdoor knowledge for one might affect the other

**If no (independent):**
- Must be proven independent
- Coordination needed to prevent sequential influence

### 4. Adaptive Parameter Choice

**Scenario 1: Adaptive x**
```
1. GS CRS generated → {Uⱼ, Vₖ} fixed
2. Attacker observes {Uⱼ, Vₖ}
3. Attacker chooses x such that G_G16(vk, x) has favorable properties:
   - Expressible using {Uⱼ, Vₖ}
   - Small order
   - Degenerate (=1, checked at line 146 but could be circumvented)
4. Exploitation as in PVUGC-001
```

**Scenario 2: Adaptive CRS**
```
1. Attacker participates in or observes CRS generation
2. Learns structural properties
3. During arming, chooses ρᵢ to exploit known structure
4. Creates masks with special algebraic properties
```

### 5. Enforcement Mechanism

**Problem:** Protocol says "Implementation MUST NOT allow armers to influence CRS selection"

**Questions:**
- How is this enforced? Social convention? Cryptographic commitment?
- What prevents armers from observing CRS before committing to participation?
- Can armers decline participation after seeing CRS (selection bias)?

**No cryptographic enforcement specified:** Unlike other protocol properties (e.g., PoCE proves mask correctness), independence relies on procedural guarantees, not cryptographic ones.

---

## Attack Vectors

### Attack 3.1: Correlated CRS Exploitation

```
Preconditions:
- Groth16 CRS and GS CRS generated non-independently
- Or shared randomness/trapdoor

Attack steps:
1. Malicious participant in CRS generation phase
2. Introduces correlation between Groth16 elements and GS bases:
   - G_G16 expressible using {Uⱼ, Vₖ}
   - Or ρᵢ chosen to exploit hidden structure
3. During arming: publishes masks D₁,ⱼ, D₂,ₖ with special properties
4. Either:
   a) Enables computing M without proof (insider), OR
   b) Shares correlation knowledge with external attacker
5. Attack proceeds as in PVUGC-001

Result: Break if CRS setup is malicious
```

### Attack 3.2: Adaptive Public Input

```
Preconditions:
- Attacker can influence x choice after observing CRS
- Or propose multiple x values and select based on properties

Attack steps:
1. Observe GS CRS → know {Uⱼ, Vₖ}
2. For various x candidates, compute G_G16(vk, x)
3. Search for x where G_G16 has favorable structure:
   - Low algebraic complexity
   - Expressible as product involving {Uⱼ, Vₖ}
4. Propose transaction with chosen x
5. Exploit structure to compute M without proof

Result: Adaptive break (requires input manipulation)
```

### Attack 3.3: Observation-Based Selection

```
Preconditions:
- Armers can observe (CRS, vk, x) before committing to participation
- Selective participation allowed

Attack steps:
1. Protocol announces (GS_CRS, Groth16_CRS, vk, x)
2. Potential armers analyze for exploitable structure
3. If favorable: Collude to participate and exploit
4. If unfavorable: Decline participation (DoS or force re-setup)
5. If participate and exploit: Spend without proof

Result: Selection bias attack
```

---

## Recommendations

### Immediate Actions

#### 1. Formalize Setup Ceremony
**Priority:** P0 (Critical)
**Timeline:** 1-2 weeks

**Deliverable:** Precise setup ceremony specification

Required ordering:
```
Step 1: Groth16 CRS generation
  - Trusted setup ceremony (MPC recommended)
  - Produces vk, proving key (pk discarded), CRS elements
  - Publish: vk, Groth16_CRS_hash

Step 2: Public input commitment
  - Transaction proposer commits to x: c_x = Commit(x, r_x)
  - Publish: c_x

Step 3: GS CRS generation
  - Independent trusted setup (different participants from Step 1)
  - Produces GS CRS with bases {Uⱼ, Vₖ}
  - Publish: GS_CRS, GS_CRS_hash

Step 4: Armer commitment phase
  - Armers commit to participation: c_armer = Commit(armer_id, r_armer)
  - No CRS or x knowledge at this point
  - Publish: {c_armer_i}

Step 5: Parameter revelation
  - Reveal x (open c_x)
  - Compute G_G16(vk, x)
  - Verify G_G16 ≠ 1 and proper order

Step 6: Arming phase
  - Committed armers (from Step 4) now execute arming protocol
  - Choose ρᵢ, publish masks, etc.
```

**Key properties:**
- Armers commit before seeing CRS or x (no adaptive choice)
- CRS generated independently
- x committed before GS CRS generation (no adaptive target)

#### 2. Prove CRS Independence
**Priority:** P0 (Critical)
**Timeline:** 2-3 months (research)

Tasks:
- [ ] Formal proof: G_G16(vk, x) and {Uⱼ, Vₖ} are independent
  - Define "independence" precisely (information-theoretic, computational, etc.)
  - Prove: Knowledge of one doesn't help compute related quantities of the other
- [ ] Alternatively: Use separate, provably independent CRS
  - Different curve parameters?
  - Different pairing types?
- [ ] Engage experts in pairing-based crypto for validation

#### 3. Cryptographic Commitment Enforcement
**Priority:** P1 (High)
**Timeline:** 2-3 weeks

Replace procedural guarantees with cryptographic commitments:
- [ ] Armer participation: Commit before seeing parameters
- [ ] Public input: Commit before GS CRS generation
- [ ] CRS binding: Add CRS hashes to ctx_core (see PVUGC-005)

#### 4. Multi-Party CRS Generation
**Priority:** P1 (High)
**Timeline:** 4-6 weeks (implementation)

Tasks:
- [ ] Use MPC for both Groth16 and GS CRS generation
- [ ] Different participant sets for each CRS (reduce correlation risk)
- [ ] Public verifiability: Transcripts published and auditable
- [ ] Reference: Zcash Powers of Tau ceremony (Groth16 example)

### Specification Improvements

#### 5. Add Setup Section to Protocol
**Timeline:** 1 week

Add new section to PVUGC-breakthrough.md:
```markdown
## § 2.5: Setup Ceremony and Independence Guarantees

### Ordering Requirements
[Detailed steps as above]

### Independence Property
[Formal statement and proof sketch]

### CRS Generation
[Specific algorithms for Groth16 and GS CRS]

### Commitment Schemes
[Armer and input commitment protocols]
```

#### 6. Bind CRS to Context
**Timeline:** 1 week (specification)

Update ctx_core definition (see PVUGC-005):
```
ctx_core = H("PVUGC/CTX_CORE" ||
             Groth16_CRS_hash ||
             GS_CRS_hash ||
             vk_hash ||
             H(x) ||
             tapleaf_hash ||
             tapleaf_version ||
             txid_template ||
             path_tag ||
             epoch_nonce)
```

This prevents CRS substitution attacks.

### Testing & Validation

#### 7. Security Testing
**Timeline:** 2-3 weeks

Tests:
- [ ] **Setup ordering:** Violate ordering, verify protocol aborts
- [ ] **Adaptive x:** Try choosing x after CRS, verify rejection
- [ ] **CRS correlation:** Test for exploitable correlations (computational search)
- [ ] **Selection bias:** Simulate armers declining participation, measure impact

#### 8. Formal Verification
**Timeline:** 2-3 months (advanced)

Tasks:
- [ ] Model setup ceremony in formal verification tool (e.g., Tamarin, ProVerif)
- [ ] Prove: Independence property holds under ceremony protocol
- [ ] Verify: Timing constraints prevent adaptive attacks

---

## Acceptance Criteria for Resolution

This flaw can be considered resolved when **ALL** of the following are achieved:

- [ ] **Setup ceremony formalized** with precise ordering and commit-reveal phases
- [ ] **Independence formally proven** or separate provably independent CRS used
- [ ] **Cryptographic commitments** enforce ordering (not just procedural)
- [ ] **CRS hashes bound** in ctx_core
- [ ] **MPC-based CRS generation** specified and reference implementation available
- [ ] **External review:** Pairing-crypto experts validate independence claim
- [ ] **Test suite** verifies ordering enforcement and adaptive attack resistance

---

## Related Flaws
- **PVUGC-001:** Power-Target Hardness (independence is assumption in that hardness claim)
- **PVUGC-002:** GS commitment malleability (CRS trust also affects this)
- **PVUGC-005:** Context binding (CRS binding directly related)

---

## References

### Trusted Setup Ceremonies
- **Zcash Powers of Tau:** Multi-party Groth16 CRS generation (public transcript)
- **Aztec Ignition:** Another example of MPC-based trusted setup
- **BCTV14:** "Scalable Zero Knowledge via Cycles" - discusses CRS generation

### Independence in Cryptography
- **Canetti:** Universally Composable Security (UC framework) - defining independence
- **BR93:** "Random Oracles are Practical" - independence via hashing

---

## Notes

- This finding is **critical** because it affects PVUGC-001's hardness assumption
- If independence doesn't hold, Power-Target Hardness may not be hard even if the assumption is theoretically sound
- Unlike PVUGC-001 (novel assumption), this can be addressed with proper ceremony design
- Requires both cryptographic analysis AND protocol/procedural improvements

---

**Last Updated:** 2025-10-07
**Next Review:** After setup ceremony formalization and independence proof
