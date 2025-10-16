# PVUGC-005: Context Binding - Incomplete Binding and Replay

**Flaw Code:** PVUGC-005
**Severity:** 🟠 **HIGH**
**Status:** 🔓 Open
**Date Identified:** 2025-10-07

---

## Component
Layered hash structure (ctx_hash)

## Location
- **Section:** §3 (Context binding)
- **Lines:** 54-82 (ctx_hash definition and domain tags)

---

## Description

The protocol uses layered hashing to bind artifacts to context:

```
ctx_core        = H("PVUGC/CTX_CORE" || vk_hash || H(x) || tapleaf_hash ||
                     tapleaf_version || txid_template || path_tag)
arming_pkg_hash = H("PVUGC/ARM" || {D₁} || {D₂} || header_meta)
presig_pkg_hash = H("PVUGC/PRESIG" || m || T || R || signer_set || musig_coeffs)
ctx_hash        = H("PVUGC/CTX" || ctx_core || arming_pkg_hash || presig_pkg_hash)
```

**The gap:** Critical parameters missing from bindings, package hashes not tied to ctx_core, enabling cross-context replay and substitution attacks.

---

## Security Impact

**Cross-context replay:** Reuse arming artifacts across different transactions
**CRS substitution:** Replace GS/Groth16 CRS with malicious version
**Nonce reuse:** Same ctx_hash for multiple transactions with same (vk, x)

---

## Specific Concerns

### 1. CRS Not Bound in ctx_core

**Missing:**
- Groth16_CRS_hash: Partially included via vk_hash, but vk ≠ full CRS
- GS_CRS_hash: **Completely missing**

**Risk:**
```
Attack: CRS Substitution
1. Honest arming for (GS_CRS_1, vk, x)
2. Attacker substitutes GS_CRS_2 (malicious, with trapdoor)
3. Bases {U'ⱼ, V'ₖ} now from GS_CRS_2
4. Published {D₁,ⱼ, D₂,ₖ} reinterpreted under new CRS
5. Attacker uses trapdoor to compute M
```

### 2. GS_instance_digest Placement

Appears in AD_core (line 213) but **not in ctx_core**

**Should be in ctx_core** because:
- GS instance includes CRS, {Uⱼ, Vₖ}, public inputs
- Critical to security, should be in top-level binding

### 3. arming_pkg_hash Independence

Defined as: `H("PVUGC/ARM" || {D₁} || {D₂} || header_meta)`

**Doesn't include:**
- ctx_core
- txid_template
- vk or x

**Risk:**
```
Attack: Cross-Context Arming Replay
Context A: (vk, x, tx_A)
Context B: (vk, x, tx_B)  // same (vk,x), different tx

1. Armers publish {D₁}, {D₂} for context A
2. Attacker creates context B (different txid_template)
3. ctx_core_B differs (different txid_template)
4. But arming_pkg_hash_A can be reused (no binding to ctx_core!)
5. ctx_hash_B = H("..." || ctx_core_B || arming_pkg_hash_A || presig_pkg_hash_B)
6. KEM uses ctx_hash_B as salt, but arming artifacts from A
7. Depending on KEM design: may succeed or fail
```

### 4. No Explicit Epoch/Nonce

**Missing:** Unique counter or nonce in ctx_hash

**Risk:**
- Multiple transactions with same (vk, x, tapleaf, path) share ctx_core
- If txid_template similar (e.g., only output amounts differ): near-identical ctx_hash
- Nonce reuse in DEM (line 208: deterministic nonce from ctx_hash)

NUMS key generation (line 46) includes epoch, but not propagated to ctx_hash.

---

## Attack Vectors

### Attack 5.1: Cross-Context Arming Replay

```
Setup:
- Context A: (vk, x, tx_template_A) with arming artifacts {D₁}, {D₂}, {ct_i}
- Context B: (vk, x, tx_template_B) with different transaction

Attack:
1. Compute arming_pkg_hash_A (includes only {D₁}, {D₂}, header_meta)
2. Create context B with ctx_core_B (different txid_template_B)
3. Reuse arming_pkg_hash_A in context B:
   ctx_hash_B = H("..." || ctx_core_B || arming_pkg_hash_A || presig_pkg_hash_B)
4. If KEM derivation tolerates this: unintended key reuse
5. If proof exists for (vk,x): decapsulation may succeed in both contexts

Impact: Arming artifacts leak across contexts
```

### Attack 5.2: Malicious GS CRS Substitution

```
Attack:
1. Protocol uses (GS_CRS_1, vk, x) for arming
2. GS_CRS_1_hash not in ctx_core
3. Attacker proposes using GS_CRS_2 (malicious, with known trapdoor)
4. Bases change: {Uⱼ, Vₖ} → {U'ⱼ, V'ₖ}
5. Published masks {D₁,ⱼ} interpreted as U'ⱼ^ρ instead of Uⱼ^ρ
6. Attacker computes M using trapdoor knowledge
7. Decapsulates, finalizes signature without proof

Impact: Complete break if CRS substitutable
```

### Attack 5.3: Epoch/Nonce Reuse

```
Setup:
- Transaction 1: (vk, x, template₁) → ctx_hash₁
- Transaction 2: (vk, x, template₂) → ctx_hash₂
- If template₁ ≈ template₂ and no epoch nonce: ctx_hash₁ ≈ ctx_hash₂

Attack:
1. DEM nonce (line 208): nonce_i = H("PVUGC/AEAD-NONCE" || ser(M_i) || H(AD_core))
2. If ctx_hash repeated and same share i: nonce_i repeats
3. For non-SIV AEAD (AES-GCM, ChaCha20-Poly1305): nonce reuse breaks security
4. Attacker observes two ciphertexts with same nonce, different plaintexts
5. XOR attack: ct₁ ⊕ ct₂ = pt₁ ⊕ pt₂
6. May leak s_i

Impact: Ciphertext confidentiality break
```

---

## Recommendations

### Immediate Actions

#### 1. Bind CRS in ctx_core
**Priority:** P0
**Timeline:** 1 week

Update ctx_core definition:
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

Add:
- `Groth16_CRS_hash`: Full CRS hash (not just vk)
- `GS_CRS_hash`: Complete GS CRS binding
- `epoch_nonce`: Unique per setup/transaction

#### 2. Bind arming_pkg_hash to ctx_core
**Priority:** P0
**Timeline:** 1 week

Update arming_pkg_hash:
```
arming_pkg_hash = H("PVUGC/ARM" ||
                     ctx_core ||           // NEW: bind to context
                     {D₁} ||
                     {D₂} ||
                     header_meta)
```

This prevents cross-context arming replay.

#### 3. Elevate GS_instance_digest
**Priority:** P1
**Timeline:** 1 week

Move GS_instance_digest from AD_core to ctx_core:
```
ctx_core = H("..." || GS_instance_digest || ...)
```

#### 4. Add Epoch Counter
**Priority:** P1
**Timeline:** 1 week

Specification:
```
epoch_nonce: Unique 256-bit value per protocol instance
- Generation: Random (entropy from multiple sources)
- OR: Counter (global, monotonic)
- Binding: Included in ctx_core, NUMS derivation (line 46), AD_core
```

Prevents:
- ctx_hash reuse across transactions
- Nonce reuse in DEM
- Cross-epoch attacks

### Specification Improvements

#### 5. Update §3 with Complete Bindings
**Timeline:** 1 week

Rewrite §3 context binding section:
```markdown
## §3: Context Binding (Revised)

### Core Context
ctx_core = H("PVUGC/CTX_CORE" ||
             Groth16_CRS_hash ||
             GS_CRS_hash ||
             vk_hash ||
             H(x) ||
             GS_instance_digest ||
             tapleaf_hash ||
             tapleaf_version ||
             txid_template ||
             path_tag ||
             epoch_nonce)

### Package Hashes
arming_pkg_hash = H("PVUGC/ARM" ||
                     ctx_core ||         // Binds arming to context
                     {D₁} || {D₂} ||
                     header_meta)

presig_pkg_hash = H("PVUGC/PRESIG" ||
                     ctx_hash_partial || // Binds to ctx_core + arming
                     m || T || R ||
                     signer_set ||
                     musig_coeffs)

### Full Context
ctx_hash = H("PVUGC/CTX" ||
             ctx_core ||
             arming_pkg_hash ||
             presig_pkg_hash)
```

#### 6. Binding Verification Checklist
**Timeline:** 1 week

Add to specification:
```
Verifier MUST check:
☐ Groth16_CRS_hash in ctx_core matches actual CRS
☐ GS_CRS_hash in ctx_core matches actual CRS
☐ epoch_nonce unique (not reused from previous protocol instance)
☐ arming_pkg_hash includes ctx_core (recompute and verify)
☐ ctx_hash correctly chains all components
```

### Testing

#### 7. Cross-Context Replay Tests
**Timeline:** 2 weeks

Tests:
- [ ] Same (vk,x) different tx → different ctx_hash
- [ ] Reuse arming artifacts → must be rejected (hash mismatch)
- [ ] CRS substitution → must be detected (hash in ctx_core)
- [ ] Epoch reuse → must be rejected
- [ ] Nonce uniqueness → verify no DEM nonce repeats

---

## Acceptance Criteria

- [ ] CRS hashes bound in ctx_core
- [ ] arming_pkg_hash includes ctx_core binding
- [ ] Epoch nonce added and enforced unique
- [ ] GS_instance_digest moved to ctx_core
- [ ] Specification updated with complete bindings
- [ ] Test suite verifies cross-context isolation
- [ ] External review validates binding completeness

---

## Related Flaws
- **PVUGC-003:** Independence violation (CRS binding directly related)
- **PVUGC-009:** DEM interoperability (nonce reuse affects this)

---

**Last Updated:** 2025-10-07
