# PVUGC-006: Degenerate Values - Stage 2 Standards Validation

**Issue:** PVUGC-006
**Stage:** Stage 2 (Standards Validation)
**Date:** 2025-10-29
**Decision Method:** Solo
**Verdict:** PARTIAL

---

## Evidence from v2.7

**Spec Location:**
- PVUGC-2025-10-27.md §6, line 191 (GS size bounds: m_1 + m_2 ≤ 96)
- PVUGC-2025-10-27.md §6, line 212 (Degenerate & subgroup guards for R(vk,x))
- PVUGC-2025-10-27.md §6, line 213 (Canonical serialization: ser_G_T)
- PVUGC-2025-10-27.md §6, line 214 (Group model and subgroup checks)
- PVUGC-2025-10-27.md §8, line 283 (Per-share T_i ≠ O validation)
- PVUGC-2025-10-27.md §8, line 284 (Aggregated T ≠ O validation)
- PVUGC-2025-10-27.md §12, line 306 (Mandatory hygiene: subgroup checks)
- PVUGC-2025-10-27.md §12, line 374 (KEM/DEM checklist with explicit details)

**v3.0 Status:**
- Enhanced (Medium severity)

**v3.0 Baseline (report-peer_review-2025-10-26/PVUGC-006.md):**

The v3.0 peer review validated that v2.0 introduced comprehensive first-order guards addressing all six originally identified degenerate value vulnerabilities:

1. G_G16 identity/subgroup checks (prevents M = 1)
2. GS attestation size bounds (DoS prevention: m_1 + m_2 ≤ 96)
3. Per-share T_i ≠ O validation (early detection of s_i = 0)
4. Aggregated T ≠ O validation (prevents Σs_i = 0 collusion)
5. Canonical ser_G_T encoding (interoperability)
6. Subgroup membership tests (small-order element prevention)

The peer review also identified a second-order vulnerability: collusive randomness cancellation (∏ρ_i ≡ 1 mod r), which is tracked separately as PVUGC-011.

---

## Assessment

**First-Order Guards Status (v2.7):**

All seven layers of the v2.0 defense system are **maintained** in v2.7:

### Layer 1: R(vk,x) Identity and Subgroup Checks

**Line 212:**
```
Abort arming if R(vk, x) = 1 (identity in G_T) or if it lies in any proper subgroup
of G_T. While negligible for honest setups, these checks prevent trivial keys. PoCE
MUST also assert R(vk,x) ≠ 1 via a public input bit tied to GS_instance_digest.
```

**Status:** ✅ MAINTAINED
- Timing: During arming phase setup (before ρ_i chosen)
- Checks: Identity test + subgroup membership + PoCE assertion
- Security: Prevents M = R(vk,x)^ρ = 1^ρ = 1 (trivial KEM key)

### Layer 2: GS Size Bounds (DoS Prevention)

**Line 191:**
```
Implementations MUST reject GS attestations where the total number of pairings exceeds
96. Specifically, if m_1 is the number of G₁ commitments and m_2 is the number of G₂
commitments, then MUST enforce: m_1 + m_2 ≤ 96.
```

**Status:** ✅ MAINTAINED
- Normative: MUST enforcement (not SHOULD)
- Bound: 96 pairings maximum
- Rationale: ~50-100ms verification time on BLS12-381 (acceptable for one-time decap)
- Security: Prevents oversized attestation DoS attacks

### Layer 3: Per-Share T_i Validation

**Line 283:**
```
Per-share validation (MUST): For each arming package i, verify T_i ≠ O (point at
infinity). This detects s_i = 0 early (before aggregation) and prevents griefing
via invalid null shares.
```

**Status:** ✅ MAINTAINED
- Timing: During arming phase (before pre-signing)
- Check: T_i ≠ O for each individual share
- Security: Early detection of null shares (prevents late-stage failures)

### Layer 4: Aggregated T Validation

**Line 284:**
```
Aggregated validation (MUST): Compute T = Σ T_i and reject if T = O (point at
infinity). This prevents collusion where Σ s_i = 0 mod n (which would make the
pre-signature immediately valid without decryption).
```

**Status:** ✅ MAINTAINED
- Timing: After aggregation T = Σ_{i=1}^k T_i
- Check: Aggregate T ≠ O
- Security: Prevents collusion attack (Σs_i = 0 → α = 0 → pre-sig valid without decap)

### Layer 5: Canonical ser_G_T Encoding

**Line 213:**
```
Serialization (MUST): Use a canonical, subgroup-checked encoding ser_G_T(·) for
KDF input; reject non-canonical encodings.
```

**Line 374:**
```
canonical ser_G_T (fixed-length 576 bytes = 12×48B Fp limbs, little-endian for BLS12-381)
```

**Status:** ✅ MAINTAINED with ENHANCEMENT
- v2.7 improvement: Explicit format specification (576 bytes, little-endian, 12×48B Fp limbs)
- v2.0: "Use canonical encoding" (no format details)
- Security: Eliminates serialization divergence → ensures interoperability

### Layer 6: Subgroup Membership Tests

**Line 214:**
```
Group model. G_1, G_2, G_T are prime-order groups of order r. Implementations MUST
perform subgroup checks using library-provided tests (and cofactor clearing where
applicable) and reject non-canonical encodings.
```

**Line 306:**
```
Mandatory hygiene: subgroup checks (G_1, G_2, G_T), cofactor clearing, ...
```

**Status:** ✅ MAINTAINED
- Scope: All three groups (G_1, G_2, G_T)
- Method: Library-provided tests + cofactor clearing
- Security: Prevents small-order element attacks

### Layer 7: PoCE Assertion of R(vk,x) ≠ 1

**Line 212 (continued):**
```
PoCE MUST also assert R(vk,x) ≠ 1 via a public input bit tied to GS_instance_digest.
```

**Status:** ✅ MAINTAINED
- Method: Public input constraint in PoCE-A circuit
- Binding: Tied to GS_instance_digest (prevents substitution)
- Security: Cryptographic enforcement (not just implementation check)

**Remaining Gaps:**

### Second-Order Issue: Collusive Randomness Cancellation

**Status:** NOT ADDRESSED (see PVUGC-011)

The v2.7 spec maintains only per-share ρ_i ≠ 0 validation without enforcing independence across armers. Coalitions can still coordinate {ρ_i} to achieve ∏ρ_i ≡ 1 (mod r).

**Evidence:**
- Line 200: "Choose ρ_i ∈ Z_r* (non-zero)" ← individual check only
- Line 278 (PoCE-A): "ρ_i ≠ 0 (via auxiliary ρ_i · u_i = 1)" ← per-share
- No commit-reveal protocol for temporal isolation

This gap is tracked separately as PVUGC-011 (PERSISTS).

---

## Verdict: PARTIAL

**Justification:**

The v2.7 specification successfully maintains all seven first-order degenerate value guards introduced in v2.0, with enhancements to serialization specificity (576-byte format for BLS12-381). All critical attack vectors identified in v1.0 remain mitigated:
- R(vk,x) = 1 (blocked by line 212)
- Small-order R(vk,x) (blocked by line 214)
- Oversized GS attestations (blocked by line 191)
- Null shares s_i = 0 (blocked by lines 283-284)
- Serialization divergence (blocked by lines 213, 374)

However, the second-order collusive randomness vulnerability (PVUGC-011) is unaddressed. This prevents a RESOLVED verdict. First-order security is **deployment-ready**; second-order hardening is **recommended** for future-proofing.

**Flag for Gap-Remediation:** No (first-order guards complete)

**Related Issue Flagged:** Yes (PVUGC-011 requires gap remediation for second-order hardening)

---

## Cross-References

- v3.0 Peer Review: `report-peer_review-2025-10-26/PVUGC-006.md`
- Related Issue: `validations/stage2/PVUGC-011.md` (second-order collusive randomness - PERSISTS)
- v2.0 Baseline: PVUGC-2025-10-20.md (first-order guards introduced)

---

**Date:** 2025-10-29
**Auditor:** Standards Compliance Auditor (Lead)
**Verdict:** PARTIAL
