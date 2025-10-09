# PVUGC-008: MuSig2 Compartmentalization Clarified

**Flaw Code:** PVUGC-008
**Severity:** 🟡 **MEDIUM**
**Status:** 🔧 Improved (MUST clause added, enforcement mechanism TBD)
**Date Identified:** 2025-10-07 (v1.0)
**Last Updated:** 2025-10-07 (v2.0)

---

## Component
MuSig2 adaptor signature nonce uniqueness and compartmentalization

## Location
- **v1.0:** §4, §10 (informal "never reuse" statement)
- **v2.0:** §4, §10, Line 102 (explicit MUST clause for compartmentalization)

---

## Description

The protocol uses MuSig2 adaptor signatures to create Bitcoin spends gated on proof existence. Critical to security: **nonces (R) and adaptor points (T) must be unique per context**.

**Why uniqueness is critical:**
- **Nonce reuse (R):** Classic Schnorr vulnerability - reusing R across two messages allows private key extraction
- **Adaptor reuse (T):** Cross-protocol attacks if same T used in different contexts

**v1.0 statement:**
> "Never reuse across paths/epochs or templates."

**v2.0 requirement:**
> "**MUST:** Compartmentalization: one adaptor ⇒ one unique T and one unique R. Never reuse across paths/epochs or templates."

---

## What Changed in v2.0

### ✅ Improvements

**1. Explicit MUST Clause (§4, line 102)**
- **v1.0:** Informal "never reuse" guidance
- **v2.0:** Normative MUST requirement
- **Impact:** Makes uniqueness mandatory (not just best practice)

**2. Context Binding Enhanced (§3, lines 55-77)**
- **v1.0:** Unclear how T, R relate to context
- **v2.0:** presig_pkg_hash includes T, R (line 70)
  ```
  presig_pkg_hash = H_bytes("PVUGC/PRESIG" || m || T || R ||
                            signer_set || musig_coeffs)
  ```
- **Impact:** T, R bound to specific context via ctx_hash

**3. Domain Tags Specified (§3)**
- **v1.0:** No domain separation for nonce/adaptor derivation
- **v2.0:** Normative domain tags:
  - `"BIP0340/challenge"` - for Schnorr challenge hash
  - `"PVUGC/PRESIG"` - for pre-signature package binding
- **Impact:** Prevents cross-domain collisions

**4. Production Profile (§89)**
- **v1.0:** No specific curve mandated
- **v2.0:** BLS12-381 for pairing, secp256k1 for signatures (implicit)
- **Impact:** Enables focused analysis on specific curve

### 🔧 Remaining Gaps

**What's still missing:**
- ❌ No cryptographic enforcement of T, R uniqueness
- ❌ No specification of how to derive T, R deterministically
- ❌ No nonce commitment proof (that R is fresh)
- ❌ No blacklist mechanism (prevent reuse detection)
- ❌ Enforcement relies on honest implementation

**Why enforcement matters:**
- Buggy or malicious signer could reuse nonces
- No runtime detection of reuse until after-the-fact
- Private key compromise if R reused

---

## Security Impact

**Consequence if nonce reused:** Private key extraction, complete break of signature security.

**Consequence if adaptor reused:** Cross-protocol attacks, potential information leakage.

### Why This Is Medium Severity (Not Critical)

1. **Context binding provides defense:** T, R bound via presig_pkg_hash → ctx_hash
2. **Reuse detectable post-facto:** After two signatures with same R, extraction possible but observable
3. **Requires implementation bug:** Honest implementation won't reuse (if MUST clause followed)
4. **Social/procedural layer:** Multiple signers provide redundancy (one reuses, others don't)

**But still concerning because:**
- Implementation bugs are common (especially RNG failures)
- No cryptographic proof of freshness
- Retrospective detection doesn't prevent damage

---

## Specific Concerns

### 1. Nonce Reuse Attack (Classical Schnorr Vulnerability)

**Problem:** If R reused across two messages, private key recoverable.

**Attack math:**
```
Setup: Two signatures with same nonce R but different messages m₁, m₂

Signature 1 (message m₁):
- Nonce: r (private)
- Public: R = r·G
- Challenge: e₁ = H("BIP0340/challenge" || R_x || P_x || m₁)
- Signature: s₁ = r + e₁·x  (where x is private key)

Signature 2 (message m₂, SAME nonce r):
- Nonce: r (REUSED)
- Public: R (same as above)
- Challenge: e₂ = H("BIP0340/challenge" || R_x || P_x || m₂)
- Signature: s₂ = r + e₂·x

Private key extraction:
s₁ - s₂ = (r + e₁·x) - (r + e₂·x)
        = (e₁ - e₂)·x

x = (s₁ - s₂) / (e₁ - e₂)  [if e₁ ≠ e₂, which holds if m₁ ≠ m₂]

Result: Attacker recovers private key x from two signatures
```

**In PVUGC context:**
```
Scenario: Signer reuses R for two different transactions

Transaction A: (vk, x_A, tx_template_A)
- Pre-signature: s'_A with nonce R
- After decap: s_A = s'_A + α_A

Transaction B: (vk, x_B, tx_template_B)
- Pre-signature: s'_B with SAME nonce R (bug/malicious)
- After decap: s_B = s'_B + α_B

Both signatures broadcast:
- Attacker observes (R, s_A, m_A, P) and (R, s_B, m_B, P)
- Extracts: x = (s_A - s_B) / (e_A - e_B)
- Recovers aggregate private key for signer set

Impact: Can spend from any address using public key P
```

### 2. Adaptor Reuse (Cross-Protocol Attack)

**Problem:** If same T used in different contexts, revealing α in one context leaks it in all.

**Attack scenario:**
```
Setup: Same adaptor point T used for two contexts

Context A: Bitcoin spend with adaptor T
- Pre-signature: s'_A + T
- After decap: α revealed (from s_A = s'_A + α)

Context B: Different Bitcoin script with SAME adaptor T
- Pre-signature: s'_B + T
- Attacker knows α (from context A)
- Computes: s_B = s'_B + α
- Spends without generating proof for context B

Impact: Proof requirement bypassed in context B
```

**v2.0 mitigation (context binding):**
```
presig_pkg_hash includes T:
- presig_pkg_hash_A = H("..." || m_A || T_A || R_A || ...)
- presig_pkg_hash_B = H("..." || m_B || T_B || R_B || ...)

If T_A = T_B (reused):
- presig_pkg_hash_A = presig_pkg_hash_B only if all other params match
- ctx_hash_A = ctx_hash_B only if contexts truly identical
- If contexts differ: KDF derives different keys K_A ≠ K_B
- Decryption fails (α_A doesn't decrypt ciphertext for context B)

Result: Attack partially mitigated by context binding
```

**Remaining risk:** Same T reused within same context (multiple transactions with same vk, x).

### 3. Deterministic vs Random Nonce Generation

**Problem:** No specification of how to generate R, T.

**Options:**

**Option A: Random generation (v1.0 implied)**
```
r ← random(ℤᵣ)  // Cryptographic RNG
R = r·G

Pros: Simple, standard approach
Cons: RNG failures → reuse (historical Bitcoin exploits)
      No proof of uniqueness
```

**Option B: Deterministic derivation (recommended)**
```
r_seed = HKDF(secret_key, ctx_hash || "PVUGC/NONCE")
r = H_scalar(r_seed)  // Hash to scalar
R = r·G

Pros: No RNG dependency
      Unique per ctx_hash (if ctx_hash unique)
      Reproducible (for auditing)
Cons: Implementation complexity
      Requires careful key management
```

**v2.0 status:** No specification (implementation choice).

**Recommendation:** Mandate deterministic derivation for production.

### 4. MuSig2 State Machine Violations

**Problem:** BIP-327 (MuSig2) specifies careful state progression. Violating sequence can leak secrets.

**MuSig2 phases:**
```
1. Key aggregation: P = Σᵢ aᵢ·Xᵢ
2. Nonce commitment: Each signer commits to Rᵢ
3. Nonce reveal: Signers reveal Rᵢ after all commitments received
4. Nonce aggregation: R = Σᵢ Rᵢ
5. Challenge: e = H("BIP0340/challenge" || R_x || P_x || m)
6. Partial signing: Each signer computes sᵢ = rᵢ + e·aᵢ·xᵢ
7. Signature aggregation: s = Σᵢ sᵢ
```

**Critical:** Steps must occur in order (no skipping, no regressing).

**Violations that leak secrets:**
```
Violation 1: Sign before seeing all nonce commitments
- Attacker can adaptively choose their nonce
- Wagner's attack (if multiple signatures with related nonces)

Violation 2: Reveal nonce before receiving commitment
- Attacker can choose nonce based on honest party's nonce
- Cancellation attacks

Violation 3: Sign same message twice (with different nonce)
- Classic nonce reuse if any overlap
```

**v2.0 status:** presig_pkg_hash includes signer_set and musig_coeffs (line 70), but no state machine enforcement specification.

**Recommendation:** Explicitly require BIP-327 state machine compliance.

### 5. R Normalization (BIP-340 Requirement)

**Problem:** BIP-340 requires R to have even y-coordinate. Normalization timing matters.

**BIP-340 rule:**
```
If R.y is odd: negate R (use -R instead)
Result: R with even y-coordinate
```

**Issue:** If normalization happens inconsistently:
```
Scenario 1: Pre-signature computed with R_even
            Verification uses R_odd
            → Signature fails

Scenario 2: Different signers normalize at different stages
            → Aggregated R incorrect
            → Signature fails
```

**v2.0 mention (§10, lines 246, 251):**
```
R must be normalized to even-y per BIP-340
```

**What's specified:** That normalization must occur.

**What's missing:** **When** normalization occurs (timing).

**Recommendation:**
```
Specify: Normalize R immediately after MuSig2 nonce aggregation
Before: Use for adaptor verification and signing
```

### 6. Cross-Protocol T Reuse (Partially Mitigated)

**Problem:** T used for different Bitcoin scripts (different tapleafs).

**v2.0 mitigation:**
```
ctx_core includes tapleaf_hash (line 55):
  ctx_core = H_bytes("PVUGC/CTX_CORE" || vk_hash || H(x) || tapleaf_hash || ...)

Different tapleaf → different ctx_core → different ctx_hash
presig_pkg_hash includes T (line 70)
ctx_hash includes presig_pkg_hash

Result: T should differ for different tapleafs (if properly derived)
```

**Question:** Is T derivation cryptographically bound to ctx_hash?

**Current state:** MUST clause says "one adaptor ⇒ one unique T" but no derivation specified.

**If T derived deterministically from ctx_hash:** ✅ Unique per context
**If T random:** ❌ Relies on implementation to not reuse

---

## Attack Vectors (Updated for v2.0)

### Attack 8.1: Nonce Reuse Key Extraction

**Severity:** Critical (if occurs)
**Likelihood:** Low (requires implementation bug or malicious signer)

```
Preconditions:
- Buggy or malicious signer reuses R for two transactions
- Both transactions signed and broadcast

Attack steps:
1. Signer uses same R for tx₁ and tx₂ (bug/malicious)
2. Pre-signatures: s'₁ with R, s'₂ with R
3. After decapsulation: s₁ = s'₁ + α₁, s₂ = s'₂ + α₂
4. Both signatures broadcast on Bitcoin
5. Attacker observes: (R, s₁, m₁, P) and (R, s₂, m₂, P)
6. Computes challenges:
   e₁ = H("BIP0340/challenge" || R_x || P_x || m₁)
   e₂ = H("BIP0340/challenge" || R_x || P_x || m₂)
7. Extracts private key:
   x = (s₁ - s₂) / (e₁ - e₂)
8. Can now spend from any address with public key P

Impact:
- Complete compromise of aggregate key
- All funds controlled by P at risk
```

**v2.0 mitigation:** MUST clause prohibits reuse (but no enforcement).

**Detection:** Retrospective (after signatures published).

**Prevention:** Requires correct implementation + testing.

### Attack 8.2: Wagner's Algorithm (Multi-Instance)

**Severity:** High (if many instances with nonce reuse)
**Likelihood:** Very Low (requires systematic reuse)

```
Preconditions:
- Many protocol instances with reused R values
- Attacker collects k signatures with related nonces

Attack steps:
1. Attacker collects k adaptors with reused or related R values
2. Applies Wagner's generalized birthday attack:
   - Find subset where Σ eᵢ has special form
   - Reduces private key extraction complexity
3. For k ≥ 20-30: Attack becomes practical

Complexity:
- Classical nonce reuse (2 sigs): O(1)
- Wagner's attack (k sigs): O(k · 2^(λ/log k))
- For λ=128, k=20: ~2^30 operations (practical)

Impact:
- Private key extraction with reduced complexity
- Requires collecting many related instances
```

**v2.0 mitigation:** Context binding makes collecting related instances harder.

**Prevention:** Ensure R unique per context (deterministic derivation).

### Attack 8.3: Adaptor Reuse Information Leakage

**Severity:** Medium (bypass proof requirement)
**Likelihood:** Low (v2.0 context binding mitigates)

```
Preconditions:
- Same T used for two different contexts
- α revealed in one context

Attack steps:
1. Context A: Attacker observes s_A = s'_A + α
2. Recovers: α = s_A - s'_A
3. Context B: Same T reused (different transaction)
4. Pre-signature s'_B known (public)
5. Computes: s_B = s'_B + α
6. Broadcasts transaction with signature s_B
7. Spends without generating proof for context B

Impact:
- Proof requirement bypassed
- Security relies on T uniqueness
```

**v2.0 mitigation:**
- presig_pkg_hash binds T to context
- ctx_hash includes presig_pkg_hash
- KDF: K = HKDF(ctx_hash, M, ...)
- Different contexts → different K → attack fails at decryption

**Remaining risk:** Same T reused within same context (edge case).

---

## Recommendations

### Phase 1: Specification Enhancements (1-2 months)

#### 1. Deterministic T, R Derivation (HIGHEST PRIORITY)
**Owner:** Protocol designers
**Timeline:** 2 weeks
**Deliverable:** Normative derivation spec

**Specification:**
```
Deterministic nonce derivation (MUST):

r_seed = HKDF(
    salt = signer_secret_key,
    IKM = ctx_hash || "PVUGC/NONCE",
    info = signer_id || session_id
)
r = H_to_scalar(r_seed)  // Hash to scalar in [1, n-1]
R = r·G

Normalize R to even-y per BIP-340:
if R.y is odd:
    R = -R
    r = -r

Use R for MuSig2 adaptor construction


Deterministic adaptor derivation (MUST):

T_seed = HKDF(
    salt = aggregate_key_material,
    IKM = ctx_hash || "PVUGC/ADAPTOR",
    info = epoch || tapleaf_hash
)
α = H_to_scalar(T_seed)  // Adaptor secret
T = α·G  // Adaptor point (used in pre-signature)
```

**Benefits:**
- No RNG dependency (eliminates reuse from RNG failure)
- Unique per ctx_hash (cryptographically enforced)
- Reproducible (auditing, debugging)
- Prevents accidental reuse

#### 2. Nonce Commitment Proof (Optional, Advanced)
**Owner:** Research team
**Timeline:** 1-2 months
**Deliverable:** ZK proof for nonce freshness

**Concept:**
```
At pre-signature phase, each signer proves (in ZK):
"I know r such that R = r·G AND r was derived from ctx_hash"

Proof system:
- Public inputs: R, ctx_hash
- Private inputs: r, secret_key
- Statement: R = r·G ∧ r = DeriveNonce(secret_key, ctx_hash)

Benefits:
- Cryptographic proof of uniqueness
- Detects reuse at signing time (not post-facto)

Challenges:
- Overhead (additional ZK proof)
- Complexity (requires proving HKDF in circuit)
```

**Recommendation:** Optional extension for high-security deployments.

#### 3. R/T Blacklist Mechanism
**Owner:** Implementation team
**Timeline:** 2 weeks
**Deliverable:** State tracking system

**Mechanism:**
```
Maintain set of previously used (R, T) pairs:
- blacklist_R = {R₁, R₂, ..., Rₖ}
- blacklist_T = {T₁, T₂, ..., Tₘ}

Before signing:
- Check: R ∉ blacklist_R (reject if duplicate)
- Check: T ∉ blacklist_T (reject if duplicate)
- Add to blacklist: blacklist_R ∪= {R}, blacklist_T ∪= {T}

Persistence:
- Store blacklist in database
- Sync across restarts
- Optionally: Distributed blacklist (cross-participant)
```

**Benefits:**
- Runtime detection of reuse
- Defense against buggy deterministic derivation

**Challenges:**
- State management (persistence)
- Synchronization (multi-signer)
- Storage growth (bounded by protocol lifetime)

#### 4. MuSig2 State Machine Enforcement
**Owner:** Protocol designers + implementation team
**Timeline:** 2 weeks
**Deliverable:** State machine spec + reference impl

**Specification:**
```
Enforce BIP-327 state machine (MUST):

State 0: INIT
  - Collect public keys {X₁, ..., Xₙ}
  - Compute aggregate key: P = Σᵢ aᵢ·Xᵢ
  - Transition: → NONCE_COMMIT

State 1: NONCE_COMMIT
  - Each signer commits: commit_i = H(Rᵢ || nonce_i)
  - Collect all commitments
  - Transition: → NONCE_REVEAL (when all commits received)

State 2: NONCE_REVEAL
  - Each signer reveals: (Rᵢ, nonce_i)
  - Verify: H(Rᵢ || nonce_i) = commit_i
  - Transition: → NONCE_AGG (when all reveals verified)

State 3: NONCE_AGG
  - Aggregate: R = Σᵢ Rᵢ
  - Normalize: R to even-y per BIP-340
  - Transition: → SIGNING

State 4: SIGNING
  - Compute challenge: e = H("BIP0340/challenge" || R_x || P_x || m)
  - Each signer: sᵢ = rᵢ + e·aᵢ·xᵢ
  - Aggregate: s = Σᵢ sᵢ
  - Transition: → COMPLETE

Invariants:
- Cannot skip states
- Cannot regress
- Cannot sign twice in same session
```

**Testing:**
- State transition tests
- Violation detection (attempt to skip)
- BIP-327 test vectors

#### 5. R Normalization Timing Specification
**Owner:** Protocol designers
**Timeline:** 1 week
**Deliverable:** Spec clarification

**Normative requirement:**
```
R normalization (MUST):

Timing: Immediately after MuSig2 nonce aggregation (State 3)
Before: Any use of R (adaptor verification, signing, hashing)

Procedure:
1. Aggregate nonces: R_agg = Σᵢ Rᵢ
2. Check y-coordinate: if R_agg.y is odd
3. Normalize: R = -R_agg, r = -r_agg (all signers)
4. Use normalized R for all subsequent operations

Verification:
- presig_pkg_hash uses R_x (x-coordinate only)
- Adaptor verification uses normalized R
- Challenge hash uses normalized R
```

**Test vectors:**
- Sample R with odd y → normalized R
- Verify signature with normalized R passes

### Phase 2: Implementation & Testing (2-4 months)

#### 6. Reference Implementation
**Owner:** Implementation team
**Timeline:** 1 month
**Deliverable:** Reference MuSig2 adaptor code

**Requirements:**
```
1. Deterministic r, T derivation (per Phase 1 spec)
2. R/T blacklist mechanism
3. BIP-327 state machine enforcement
4. R normalization at correct stage
5. Comprehensive test suite (see below)
```

#### 7. Test Suite for Nonce Uniqueness
**Owner:** Testing team
**Timeline:** 1 month
**Deliverable:** Comprehensive tests

**Test categories:**
```
1. Uniqueness tests:
   - Test: 1000 signatures → 1000 unique R values
   - Test: Same ctx_hash twice → same R (deterministic)
   - Test: Different ctx_hash → different R

2. Reuse detection:
   - Test: Attempt R reuse → rejected by blacklist
   - Test: Attempt T reuse → rejected by blacklist

3. State machine:
   - Test: BIP-327 happy path
   - Test: Attempt to skip state → rejected
   - Test: Attempt to regress → rejected

4. Normalization:
   - Test: R with odd y → normalized correctly
   - Test: Signature with normalized R passes BIP-340 verification

5. Attack simulation:
   - Test: Two signatures with same R → key extractable (negative test)
   - Test: Adaptor reuse attack → fails (positive test)
```

#### 8. Formal Verification (Optional)
**Owner:** Research team
**Timeline:** 3-6 months
**Deliverable:** Formal proof

**Approach:**
```
Use formal methods tool (e.g., Tamarin, ProVerif):
- Model: MuSig2 adaptor protocol
- Property: Nonce uniqueness ⇒ key security
- Prove: If R unique per context, no key extraction
- Verify: Deterministic derivation ensures uniqueness
```

---

## Acceptance Criteria for Resolution

This flaw can be considered **RESOLVED** when:

### Option A: Deterministic + Blacklist (Preferred)
- [ ] Deterministic r, T derivation specified (normative MUST)
- [ ] Derivation bound to ctx_hash (cryptographic uniqueness)
- [ ] R/T blacklist mechanism implemented
- [ ] BIP-327 state machine enforced
- [ ] R normalization timing specified
- [ ] Test suite (1000+ tests) demonstrates uniqueness
- [ ] No R or T reuse observed in testing

### Option B: Best Practices + Testing (Acceptable)
- [ ] Deterministic derivation recommended (SHOULD, not MUST)
- [ ] Best practices guide for implementation
- [ ] Test suite demonstrates uniqueness for reference impl
- [ ] Documentation warns about reuse risks
- [ ] Blacklist recommended but not required

---

## Related Flaws

- **PVUGC-007:** Timing/race conditions (related - MuSig2 coordination timing)
- **PVUGC-005:** Context binding (related - presig_pkg_hash includes T, R)

---

## References

### MuSig2 Specifications
- **BIP-327:** MuSig2 for BIP340-compatible Multi-Signatures (Nick, Ruffing, Seurin)
- **MuSig2 paper:** "MuSig2: Simple Two-Round Schnorr Multi-Signatures" (CRYPTO 2021)

### Schnorr Signatures
- **BIP-340:** Schnorr Signatures for secp256k1
- **Nonce reuse attacks:** Historical Bitcoin exploits (blockchain.info, 2013)

### Deterministic Nonces
- **RFC 6979:** Deterministic Usage of DSA and ECDSA
- **BIP-340 recommendations:** Nonce derivation best practices

---

## Notes

- **v2.0 Status:** Improved with MUST clause, but enforcement mechanism still unspecified
- **Priority:** Medium (requires implementation bug to exploit, but consequences severe)
- **Mainnet considerations:** MUST implement deterministic derivation before mainnet
- **Testing critical:** Uniqueness must be verified across thousands of test cases
- **Defense-in-depth:** Multiple layers (deterministic + blacklist + state machine)

---

**Version History:**
- **v1.0 (2025-10-07):** Initial identification (informal guidance, no enforcement)
- **v2.0 (2025-10-07):** Improved with MUST clause and context binding (enforcement TBD)

**Last Updated:** 2025-10-07
**Next Review:** After reference implementation with deterministic derivation (target: 2-3 months)
