# PVUGC-005: Context Binding Strengthened

**Flaw Code:** PVUGC-005
**Severity:** 🟡 **MEDIUM**
**Status:** 🔧 Improved (Layered hash structure, cross-context testing needed)
**Date Identified:** 2025-10-07 (v1.0)
**Last Updated:** 2025-10-07 (v2.0)

---

## Component
Layered hash structure (ctx_hash) for binding artifacts to context

## Location
- **v1.0:** §3 (context binding, incomplete)
- **v2.0:** §3, Lines 55-77 (enhanced layered hash structure with explicit domain tags)

---

## Description

The protocol uses a layered hashing scheme to bind all protocol artifacts (arming, pre-signature, proofs) to a specific context. This prevents replay attacks where artifacts from one transaction could be reused in another.

**Layered hash structure (v2.0):**
```
ctx_core        = H_bytes("PVUGC/CTX_CORE" || vk_hash || H(x) || tapleaf_hash ||
                          tapleaf_version || txid_template || path_tag)

arming_pkg_hash = H_bytes("PVUGC/ARM" || {D₁} || {D₂} || header_meta)

presig_pkg_hash = H_bytes("PVUGC/PRESIG" || m || T || R || signer_set || musig_coeffs)

ctx_hash        = H_bytes("PVUGC/CTX" || ctx_core || arming_pkg_hash || presig_pkg_hash)
```

**Purpose:** Ensure artifacts from context A cannot be used in context B.

---

## What Changed in v2.0

### ✅ Major Improvements

**1. Explicit Domain Tags (§3, lines 55-77)**
- **v1.0:** Domain tags mentioned but not fully specified
- **v2.0:** Normative domain tags for all hash layers:
  - `"PVUGC/CTX_CORE"` - core context hash
  - `"PVUGC/ARM"` - arming package hash
  - `"PVUGC/PRESIG"` - pre-signature package hash
  - `"PVUGC/CTX"` - final context hash
- **Impact:** Prevents cross-domain attacks, ensures hash independence

**2. Normative Hash Functions (§89)**
- **v1.0:** Hash function not specified ("use secure hash")
- **v2.0:**
  - `H_bytes = SHA-256` for byte-level context hashes
  - `H_p2 = Poseidon2` for KDF/DEM and in-circuit PoCE
- **Impact:** Eliminates implementation divergence, enables interoperability

**3. Enhanced header_meta Structure (§3, line 64)**
- **v1.0:** header_meta content underspecified
- **v2.0:** Explicit inclusion of:
  - GS_instance_digest (includes CRS hashes)
  - Both CRS digests (for Multi-CRS AND-ing)
  - Binding verification tags
- **Impact:** CRS now bound to context (prevents CRS substitution)

**4. Layered Hash Structure Clarified**
- **v1.0:** Single-level ctx_hash (unclear dependencies)
- **v2.0:** Three-layer structure with clear dependency graph:
  ```
  ctx_core (fixed parameters)
      ↓
  arming_pkg_hash (includes header_meta with CRS)
      ↓
  presig_pkg_hash (includes MuSig2 data)
      ↓
  ctx_hash (final binding)
  ```
- **Impact:** Clearer security analysis, explicit binding order

### 🔧 Remaining Gaps

**What's still missing:**
- ❌ Cross-context replay testing not mandated
- ❌ arming_pkg_hash doesn't directly include ctx_core (implicit via header_meta)
- ❌ Epoch/nonce not explicitly in ctx_core (multiple txs with same vk,x)
- ❌ No explicit verification that all implementations compute identical ctx_hash

---

## Security Impact

**Consequence if binding fails:** Artifacts from one context could be replayed in another, potentially allowing:
- Same arming shares used for different transactions
- Pre-signatures reused across contexts
- CRS substitution attacks

**Impact severity:** Medium (requires specific conditions, mitigated by Multi-CRS)

### Why This Is Medium Severity (Not Critical)

1. **Multi-layered binding:** Multiple hash layers provide defense-in-depth
2. **Transaction-specific:** txid_template in ctx_core prevents simple replay
3. **MuSig2 protection:** Signature commits to message m (transaction)
4. **Domain separation:** Different tags prevent cross-domain collisions

**But still concerning because:**
- Subtle replay scenarios might exist (untested)
- Implementation bugs could skip checks
- Edge cases with same (vk,x) across transactions

---

## Specific Concerns

### 1. CRS Binding (✅ Improved in v2.0)

**v1.0 gap:** CRS not bound in ctx_core.

**v2.0 improvement:**
- GS_instance_digest included in header_meta (line 64)
- header_meta included in arming_pkg_hash
- arming_pkg_hash included in ctx_hash
- **Result:** CRS indirectly bound via hash chain

**Remaining concern:**
- Indirect binding (3 levels deep) vs direct binding
- GS_instance_digest format not fully specified
- Could malicious implementation omit CRS from digest?

**Attack scenario (v1.0, mitigated in v2.0):**
```
v1.0 vulnerability:
1. Armers publish {D₁, D₂, ct} for CRS_1
2. Attacker substitutes CRS_2 (malicious, with trapdoor)
3. Bases {U'ⱼ, V'ₖ} now from CRS_2
4. Published masks interpreted as U'ⱼ^ρᵢ instead of Uⱼ^ρᵢ
5. Attacker with CRS_2 trapdoor computes M
6. Decrypts without proof

v2.0 mitigation:
- GS_instance_digest includes Hash(CRS_1)
- header_meta includes GS_instance_digest
- arming_pkg_hash binds header_meta
- CRS substitution changes arming_pkg_hash → ctx_hash changes
- KDF uses ctx_hash as salt → different K → decryption fails ✅
```

### 2. Arming Package Independence (🔧 Partially Addressed)

**v1.0 concern:** arming_pkg_hash defined as:
```
H("PVUGC/ARM" || {D₁} || {D₂} || header_meta)
```
Does **not** directly include ctx_core or txid_template.

**Question:** Can arming artifacts from context A be reused in context B?

**Analysis:**
- Arming depends on (vk, x) → determines {Uⱼ, Vₖ} → determines {D₁,ⱼ, D₂,ₖ}
- If context A and B have same (vk, x): masks {D₁, D₂} are **identical**
- arming_pkg_hash would be identical if header_meta is same
- header_meta includes GS_instance_digest (context-specific? unclear)

**Scenario:**
```
Context A: (vk, x, tx_template_A)
Context B: (vk, x, tx_template_B)  // same vk,x, different tx

Arming phase:
- Same (vk, x) → same {Uⱼ, Vₖ}
- Same ρᵢ → same {D₁,ⱼ, D₂,ₖ}
- arming_pkg_hash_A = H("..." || {D₁} || {D₂} || header_meta_A)
- arming_pkg_hash_B = H("..." || {D₁} || {D₂} || header_meta_B)

Question: Does header_meta differ between A and B?
- If header_meta includes ctx_core: YES (different txid_template)
- If header_meta doesn't include ctx_core: NO (replay possible)

v2.0 spec (line 64): header_meta includes GS_instance_digest
- GS_instance_digest includes: CRS, vk, x, public inputs
- Does NOT explicitly include txid_template

Concern: header_meta_A might equal header_meta_B if (vk,x) are same
```

**v2.0 partial mitigation:**
- ctx_hash includes presig_pkg_hash
- presig_pkg_hash includes m (message = transaction hash)
- KDF uses ctx_hash as salt
- Different tx → different m → different presig_pkg_hash → different ctx_hash → different K

**So replay attack fails at decryption step** (different K for different tx).

**Remaining question:** Should arming_pkg_hash directly bind to ctx_core for clarity?

### 3. Epoch/Nonce Missing (❌ Unchanged)

**v1.0 gap:** No explicit epoch or nonce in ctx_core.

**v2.0:** Still no explicit epoch/nonce field.

**Concern:** Multiple transactions with identical parameters:
```
Transaction 1: (vk, x, tx_template_1) at time t=0
Transaction 2: (vk, x, tx_template_2) at time t=1
```

If tx_template_1 ≈ tx_template_2 (similar structure):
- ctx_core_1 ≈ ctx_core_2
- Could there be edge cases where ctx_hash collides?

**Mitigation in v2.0:**
- txid_template in ctx_core is unique per transaction (includes tx inputs/outputs)
- Even similar transactions have different txids
- Collision probability: negligible (SHA-256 collision resistance)

**But best practice:** Include explicit nonce for uniqueness guarantee.

**Recommendation:**
```diff
ctx_core = H_bytes("PVUGC/CTX_CORE" || vk_hash || H(x) || tapleaf_hash ||
-                  tapleaf_version || txid_template || path_tag)
+                  tapleaf_version || txid_template || path_tag || nonce)
```

### 4. Serialization Consistency (🔧 Partially Addressed)

**v1.0 gap:** No specification of how to serialize parameters for hashing.

**v2.0 improvement:**
- Normative hash function specified (SHA-256 for H_bytes)
- Canonical serialization mentioned (line 147)
- Multi-CRS: ser_𝔾_T(·) for KDF IKM (line 91)

**Remaining gap:**
- No test vectors for ctx_hash computation
- Different implementations might serialize differently
- Could lead to ctx_hash divergence → KDF key mismatch → decrypt failure

**Recommendation:**
- Provide reference implementation for ctx_hash computation
- Test vectors: (vk, x, tx_template, ...) → ctx_hash (bit-exact)
- Specify byte order (big-endian vs little-endian)
- Specify concatenation delimiters (if any)

### 5. Signer Set Binding (✅ Adequate)

**v1.0 concern:** Are signer identities bound to context?

**v2.0 verification:**
```
presig_pkg_hash = H_bytes("PVUGC/PRESIG" || m || T || R ||
                          signer_set || musig_coeffs)
```

- signer_set: List of public keys {X₁, X₂, ..., Xₙ}
- musig_coeffs: {a₁, a₂, ..., aₙ} (for aggregate key P = Σᵢ aᵢ·Xᵢ)

**Result:** ✅ Signers are bound in presig_pkg_hash → ctx_hash

**No issue here.**

---

## Attack Vectors (Updated for v2.0)

### Attack 5.1: Cross-Context Arming Replay (Mitigated)

**Severity:** Medium (v1.0), Low (v2.0)
**Likelihood:** Low (v2.0 mitigation via layered hash)

```
Scenario: Reuse arming artifacts across transactions with same (vk,x)

Context A: (vk, x, tx_template_A)
Context B: (vk, x, tx_template_B)

v1.0 Attack:
1. Armers publish {D₁, D₂, ct} for context A
2. Attacker attempts to use same artifacts in context B
3. If arming_pkg_hash_A = arming_pkg_hash_B: replay succeeds
4. KDF derives K using ctx_hash (which includes arming_pkg_hash)
5. If successful: decrypt α, finalize signature for tx B using arming from A

v2.0 Mitigation:
1. ctx_hash_A = H("PVUGC/CTX" || ctx_core_A || arming_pkg_hash_A || presig_pkg_hash_A)
2. ctx_hash_B = H("PVUGC/CTX" || ctx_core_B || arming_pkg_hash_B || presig_pkg_hash_B)
3. ctx_core_A ≠ ctx_core_B (different txid_template_A vs txid_template_B)
4. ctx_hash_A ≠ ctx_hash_B (even if arming artifacts same)
5. KDF: K_A = HKDF(salt=ctx_hash_A, IKM=M, ...)
6. KDF: K_B = HKDF(salt=ctx_hash_B, IKM=M, ...)
7. K_A ≠ K_B → decrypt fails if using ct from context A in context B ✅

Result: Attack fails due to layered hash binding
```

### Attack 5.2: CRS Substitution (Mitigated in v2.0)

**Severity:** High (v1.0), Low (v2.0)
**Likelihood:** Low (v2.0 requires breaking binding)

```
v1.0 Attack:
1. Armers publish {D₁, D₂} for CRS_1 (honest)
2. Attacker substitutes CRS_2 (malicious, with trapdoor)
3. CRS_2 not bound in ctx_hash (v1.0 gap)
4. Decapper uses CRS_2 to interpret {D₁, D₂}
5. Attacker with CRS_2 trapdoor computes M
6. Breaks no-proof-spend property

v2.0 Mitigation:
1. header_meta includes GS_instance_digest
2. GS_instance_digest includes Hash(CRS_1)
3. arming_pkg_hash = H("..." || {D₁} || {D₂} || header_meta)
4. ctx_hash includes arming_pkg_hash
5. Attacker substitutes CRS_2:
   - GS_instance_digest' includes Hash(CRS_2)
   - header_meta' ≠ header_meta
   - arming_pkg_hash' ≠ arming_pkg_hash
   - ctx_hash' ≠ ctx_hash
   - K' = HKDF(ctx_hash', M, ...) ≠ K
   - Decryption fails ✅

Result: Attack fails due to CRS binding in header_meta
```

### Attack 5.3: Epoch Collision (Low Risk)

**Severity:** Low
**Likelihood:** Very Low (relies on collision)

```
Scenario: Two transactions with "similar enough" parameters

Transaction 1: (vk, x, tx_template_1)
Transaction 2: (vk, x, tx_template_2)

If attacker can craft tx_template_2 such that:
- txid_template_1 ≈ txid_template_2 (SHA-256 near-collision)
- ctx_core_1 ≈ ctx_core_2
- All other parameters match

Then: ctx_hash_1 might equal ctx_hash_2 (replay succeeds)

Reality:
- SHA-256 collision resistance: 2^128 operations
- txid includes tx inputs/outputs (unique per transaction)
- Computationally infeasible

Recommendation: Add explicit nonce for guaranteed uniqueness
```

---

## Recommendations

### Phase 1: Testing & Verification (1-2 months)

#### 1. Cross-Context Replay Testing (HIGHEST PRIORITY)
**Owner:** Security testing team
**Timeline:** 1 month
**Deliverable:** Test suite with negative results

Test scenarios:
- [ ] **Test 1:** Same (vk,x), different tx → verify arming not reusable
- [ ] **Test 2:** Same CRS, different (vk,x) → verify arming not reusable
- [ ] **Test 3:** Different CRS, same (vk,x) → verify masks not reusable
- [ ] **Test 4:** Attempt CRS substitution → verify rejection
- [ ] **Test 5:** Reuse pre-signature across contexts → verify failure
- [ ] **Test 6:** Multiple transactions with same parameters → verify uniqueness

**Success criteria:** All replay attempts fail (decrypt fails or verification fails).

#### 2. Test Vectors for ctx_hash (HIGH PRIORITY)
**Owner:** Protocol designers + implementation team
**Timeline:** 2 weeks
**Deliverable:** Normative test vectors

Provide reference:
```
Input:
  vk_hash = 0x1234...
  x = [1, 2, 3, ...]
  tapleaf_hash = 0x5678...
  tapleaf_version = 0xc0
  txid_template = 0xabcd...
  path_tag = 0xef01...
  {D₁,ⱼ}, {D₂,ₖ} = [...]
  header_meta = [...]
  m, T, R, signer_set, musig_coeffs = [...]

Expected Output:
  ctx_core = 0x...
  arming_pkg_hash = 0x...
  presig_pkg_hash = 0x...
  ctx_hash = 0x...
```

**Must be bit-exact** across all implementations.

#### 3. Serialization Specification
**Owner:** Protocol designers
**Timeline:** 1 week
**Deliverable:** Normative serialization rules

Specify:
- [ ] Byte order (big-endian for all integers)
- [ ] Array concatenation (no delimiters, direct concatenation)
- [ ] Point serialization (compressed format for curve points)
- [ ] Hash input format (tag || data, no length prefixes)

**Reference implementation:** Provide canonical code for ctx_hash computation.

### Phase 2: Specification Enhancements (2-4 weeks)

#### 4. Add Explicit Nonce to ctx_core
**Owner:** Protocol designers
**Timeline:** 1 week
**Deliverable:** Spec update

Update ctx_core definition:
```diff
ctx_core = H_bytes("PVUGC/CTX_CORE" || vk_hash || H(x) || tapleaf_hash ||
-                  tapleaf_version || txid_template || path_tag)
+                  tapleaf_version || txid_template || path_tag || nonce)

Where nonce is:
- 32-byte random value
- Generated at context creation
- Included in transaction metadata
- Ensures absolute uniqueness across contexts
```

**Impact:** Eliminates theoretical collision concerns.

#### 5. Direct CRS Binding in arming_pkg_hash (Optional)
**Owner:** Protocol designers
**Timeline:** 1 week
**Deliverable:** Spec clarification

Consider making CRS binding more explicit:
```diff
arming_pkg_hash = H_bytes("PVUGC/ARM" || {D₁} || {D₂} || header_meta)

Clarify that header_meta MUST include:
+ - GS_CRS_digest_1 (for Multi-CRS transcript 1)
+ - GS_CRS_digest_2 (for Multi-CRS transcript 2)
+ - ctx_core (optional, for direct binding)
```

**Benefit:** Makes CRS binding more explicit and easier to audit.

### Phase 3: Long-Term Improvements (3-6 months)

#### 6. Formal Context Binding Proof
**Owner:** Research team
**Timeline:** 3-4 months
**Deliverable:** Security proof

Prove (or provide strong evidence):
- [ ] **Theorem:** If ctx_hash_A ≠ ctx_hash_B, then no artifact from A is valid in B
- [ ] **Corollary:** Given SHA-256 collision resistance, ctx_hash uniqueness guarantees context isolation
- [ ] **Verification:** Formal methods tool (e.g., Tamarin, ProVerif) to verify binding properties

---

## Acceptance Criteria for Resolution

This flaw can be considered **RESOLVED** when:

### Option A: Testing + Formal Verification (Preferred)
- [ ] Comprehensive cross-context replay test suite (100+ test cases)
- [ ] All tests pass (no successful replay)
- [ ] Test vectors provided for ctx_hash (bit-exact)
- [ ] Formal proof of context binding property
- [ ] 2+ independent implementations produce identical ctx_hash

### Option B: Enhanced Specification (Acceptable)
- [ ] Explicit nonce added to ctx_core
- [ ] Serialization fully specified (reference implementation)
- [ ] CRS binding made explicit in arming_pkg_hash
- [ ] Test suite with negative results (no replay succeeded)
- [ ] Security review confirms no binding gaps

---

## Related Flaws

- **PVUGC-002:** Multi-CRS AND-ing (✅ resolved - CRS digests now in header_meta, bound via arming_pkg_hash)
- **PVUGC-010:** CRS validation (✅ resolved - binding CRS requirement ensures correct CRS used)

---

## Notes

- **v2.0 Status:** Significantly improved with layered hash structure and domain tags
- **Priority:** Medium (defense-in-depth provided by multiple layers, Multi-CRS mitigation)
- **Mainnet readiness:** Acceptable after cross-context testing completes
- **Testing critical:** Must verify no replay scenarios exist in practice
- **Best practice:** Add explicit nonce even though txid_template provides uniqueness

---

**Version History:**
- **v1.0 (2025-10-07):** Initial identification (incomplete binding, CRS not bound)
- **v2.0 (2025-10-07):** Improved with layered hash structure, domain tags, CRS binding via header_meta

**Last Updated:** 2025-10-07
**Next Review:** After cross-context replay testing (target: 1-2 months)
