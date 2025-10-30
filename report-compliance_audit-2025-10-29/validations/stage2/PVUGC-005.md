# PVUGC-005: Context Binding - Stage 2 Standards Validation

**Issue:** PVUGC-005
**Stage:** Stage 2 (Standards Validation)
**Date:** 2025-10-29
**Decision Method:** Solo
**Verdict:** RESOLVED

---

## Evidence from v2.7

**Spec Location:**
- PVUGC-2025-10-27.md §2, line 52 (NUMS derivation with epoch_nonce)
- PVUGC-2025-10-27.md §3, line 66 (epoch_nonce generation: CSPRNG(32 bytes))
- PVUGC-2025-10-27.md §3, line 67 (ctx_core with epoch_nonce)
- PVUGC-2025-10-27.md §3, lines 68-70 (layered hash structure)
- PVUGC-2025-10-27.md §3, line 87 (normative epoch_nonce requirements)

**v3.0 Status:**
- Enhanced (Low severity)

**v3.0 Baseline (report-peer_review-2025-10-26/PVUGC-005.md):**

The v3.0 peer review validated that v2.0 closed all critical v1.0 binding gaps through a sophisticated three-layer hash structure:

1. **ctx_core:** Binds (vk, x, tapleaf, txid_template, path_tag)
2. **arming_pkg_hash:** Binds {D_j}, D_δ, header_meta (includes GS_instance_digest)
3. **presig_pkg_hash:** Binds (m, T, R, signer_set, musig_coeffs)
4. **ctx_hash:** Final aggregation of all three layers

The peer review identified an **enhancement opportunity**: adding explicit `epoch_nonce` to ctx_core for:
- Provable uniqueness (formal security under Random Oracle Model)
- Independence from transaction structure assumptions
- Defense-in-depth (explicit + implicit uniqueness)

v2.0 was assessed as "deployment-ready" with practical security (2^{-256} collision probability via txid_template). v3.0 enhancement (epoch_nonce) was "RECOMMENDED" for formal provability and conservative deployments.

---

## Assessment

**epoch_nonce Implementation Status (v2.7):**

### 1. Generation Specification

**Line 66:**
```
epoch_nonce = CSPRNG(32 bytes)  // unique per instance
```

**Status:** ✅ FULLY SPECIFIED
- Size: 32 bytes (256 bits)
- Source: CSPRNG (cryptographically secure pseudorandom number generator)
- Uniqueness: "unique per instance" (explicit requirement)

### 2. Inclusion in ctx_core

**Line 67:**
```
ctx_core = H_bytes("PVUGC/CTX_CORE" || vk_hash || H_bytes(x) || tapleaf_hash ||
                   tapleaf_version || txid_template || path_tag ||
                   y_cols_digest || epoch_nonce)
```

**Status:** ✅ INCLUDED
- Position: Final field in ctx_core hash input
- Binding: Cryptographically bound via H_bytes (SHA-256)
- Impact: Different epoch_nonce → different ctx_core → different ctx_hash

### 3. Normative Requirements

**Line 87:**
```
epoch_nonce (normative). A 256-bit value sampled from OS CSPRNG (getrandom,
getentropy, or BCryptGenRandom) that MUST be unique per protocol instance.
Implementations MUST reject nonce reuse and verify all participants agree on
the same value before arming. Include in ctx_core, NUMS derivation (line 46),
and transaction metadata for verification.
```

**Status:** ✅ COMPREHENSIVE NORMATIVE SPECIFICATION
- MUST requirements:
  - Unique per protocol instance
  - Reject nonce reuse
  - Verify participant agreement before arming
- Included in:
  - ctx_core (line 67)
  - NUMS derivation (line 52)
  - Transaction metadata (for verification)
- Entropy sources specified:
  - Linux: getrandom
  - BSD/macOS: getentropy
  - Windows: BCryptGenRandom

### 4. NUMS Internal Key Derivation

**Line 52:**
```
Q_nums <- hash_to_curve("PVUGC/NUMS" || vk_hash || H(x) || tapleaf_hash ||
                        tapleaf_version || epoch_nonce)
internal_key = xonly_even_y(Q_nums)
```

**Status:** ✅ INCLUDED
- NUMS key binds epoch_nonce (prevents key reuse across instances)
- Domain tag: "PVUGC/NUMS"
- Method: hash_to_curve with simplified-SWU (IETF standard)

### 5. Layered Hash Structure Maintained

**Lines 68-70:**
```
arming_pkg_hash = H_bytes("PVUGC/ARM"    || {D_j} || D_δ || header_meta)
presig_pkg_hash = H_bytes("PVUGC/PRESIG" || m || T || R || signer_set || musig_coeffs)
ctx_hash        = H_bytes("PVUGC/CTX"    || ctx_core || arming_pkg_hash ||
                                             presig_pkg_hash || dlrep_transcripts)
```

**Status:** ✅ MAINTAINED
- Three-layer structure preserved from v2.0
- Domain separation via tags ("PVUGC/CTX_CORE", "PVUGC/ARM", "PVUGC/PRESIG", "PVUGC/CTX")
- epoch_nonce propagates: ctx_core → ctx_hash → K_i (KDF)

---

## Formal Security Impact

**Theorem 4 from v3.0 Peer Review (Now Applicable):**

With epoch_nonce in ctx_core, the Context Uniqueness property achieves provable security:

```
For any two distinct protocol instances I₁, I₂ with epoch_nonce values n₁, n₂:

If n₁ ≠ n₂ (enforced by uniqueness requirement):
  → ctx_core(I₁) ≠ ctx_core(I₂) with overwhelming probability
  → ctx_hash(I₁) ≠ ctx_hash(I₂) with overwhelming probability
  → K_i^{(1)} ≠ K_i^{(2)} (different KDF keys)
  → Ciphertext from I₁ cannot decrypt in I₂
```

**Collision Probability:**
- Single-instance: P(collision) = 0 (unique by MUST requirement)
- Multi-instance (N instances): P(any collision) ≈ N²/2^{257} (birthday bound)
  - For N = 2^{64}: P ≈ 2^{-129} (negligible)
  - For N = 2^{80}: P ≈ 2^{-97} (negligible)

**Security Model:**
- v2.0: Relied on implicit uniqueness (txid_template structure)
- v2.7: Explicit cryptographic uniqueness guarantee (epoch_nonce)
- Formal proof: Clean proof under Random Oracle Model (no external assumptions)

---

## Comparison: v2.0 vs v2.7

| Property | v2.0 (PVUGC-2025-10-20.md) | v2.7 (PVUGC-2025-10-27.md) |
|----------|---------------------------|---------------------------|
| **Layered hash structure** | ✅ Present | ✅ Maintained |
| **CRS binding** | ✅ Via header_meta chain | ✅ Maintained |
| **Domain separation** | ✅ Tags present | ✅ Maintained |
| **Explicit epoch_nonce** | ❌ Missing | ✅ **ADDED** |
| **Normative nonce spec** | ❌ N/A | ✅ **ADDED** (line 87) |
| **NUMS binding** | Partial | ✅ **ENHANCED** (line 52) |
| **Deployment readiness** | ACCEPTABLE (practical security) | **RECOMMENDED** (provable security) |
| **Formal proof** | Requires assumption | **CLEAN** (ROM only) |

---

## Verdict: RESOLVED

**Justification:**

The v2.7 specification fully implements the v3.0-recommended epoch_nonce enhancement with comprehensive normative requirements. All components are present:

1. ✅ Explicit 256-bit epoch_nonce field (line 66)
2. ✅ Included in ctx_core hash (line 67)
3. ✅ Normative MUST requirements (line 87)
4. ✅ NUMS key derivation binding (line 52)
5. ✅ Entropy source specifications (getrandom, getentropy, BCryptGenRandom)
6. ✅ Uniqueness enforcement (reject reuse, verify agreement)
7. ✅ Transaction metadata inclusion (for verification)
8. ✅ Layered hash structure maintained

The implementation elevates the protocol from "secure under reasonable assumptions" (v2.0) to "provably secure under standard assumptions" (v2.7). This addresses the v3.0 peer review's enhancement recommendation and achieves the "gold standard for cryptographic protocols handling high-value transactions."

**Flag for Gap-Remediation:** No

**Deployment Status:** RECOMMENDED for production mainnet (v3.0 target achieved)

---

## Cross-References

- v3.0 Peer Review: `report-peer_review-2025-10-26/PVUGC-005.md`
- v2.0 Baseline: PVUGC-2025-10-20.md §3 (layered hash introduced)
- v2.7 Enhancement: PVUGC-2025-10-27.md §3, lines 52, 66-67, 87

---

**Date:** 2025-10-29
**Auditor:** Standards Compliance Auditor (Lead)
**Verdict:** RESOLVED
