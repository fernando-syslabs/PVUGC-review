# PVUGC-009: Key-Committing DEM - Interoperability

**Flaw Code:** PVUGC-009
**Severity:** 🟢 **LOW**
**Status:** 🔓 Open
**Date Identified:** 2025-10-07

---

## Component
Data Encapsulation Mechanism (DEM) for encrypting sᵢ

## Location
- **Section:** §8 (PoCE and DEM details)
- **Lines:** 217-220 (DEM specification)

---

## Description

Protocol allows two DEM options:
1. **Hash-only DEM (ROM):** ctᵢ = (sᵢ || hᵢ) ⊕ H(Kᵢ, AD_core), τᵢ = H(Kᵢ, AD_core, ctᵢ)
2. **AEAD+commit wrapper (RECOMMENDED SIV-mode):** ct_aead = AEAD_Kᵢ(·), τᵢ = H(Kᵢ, AD_core, ct_aead)

**Concern:** Different implementations might choose different DEMs, causing interoperability failures. Not a security vulnerability per se, but operational risk.

---

## Specific Issues

### 1. Cross-Implementation Compatibility
- Armer uses Hash-only DEM
- Decapper expects SIV-mode AEAD
- Decryption fails (wrong format)

### 2. Non-SIV AEAD Danger
Line 220: "For non-SIV AEADs, use deterministic nonce"

**Problem:** Deterministic nonce with CTR or GCM mode is dangerous
- Nonce reuse: catastrophic security failure
- Line 208: `nonce_i = H("PVUGC/AEAD-NONCE" || ser(M_i) || H(AD_core))`
- If ctx_hash repeats (see PVUGC-005): nonce repeats
- For GCM/ChaCha20: nonce reuse breaks confidentiality

### 3. Tag Redundancy
- τᵢ = H(Kᵢ, AD_core, ctᵢ)
- For AEAD schemes, ct already includes internal MAC
- Is τᵢ redundant? Or additional security layer?
- Inconsistent interpretation across implementations

---

## Recommendations

### Specification

#### 1. Single Mandatory DEM for Mainnet
**Priority:** P3, **Timeline:** 1 week

Specify:
```markdown
### DEM (Normative for Mainnet)

MANDATORY: AES-256-SIV per RFC 5297

Parameters:
- K: 256-bit key from HKDF
- Plaintext: sᵢ || hᵢ (where hᵢ = SHA256(sᵢ || Tᵢ || i))
- AD: AD_core (as defined in §8, line 213)
- Nonce handling: Include nonce_i in AD (SIV mode doesn't use separate nonce)

Output: ct_aead (includes embedded authentication tag)

Key-commit tag: τᵢ = SHA256(Kᵢ || AD_core || ct_aead)

Verification:
1. Decrypt: (sᵢ || hᵢ) = SIV-Decrypt(Kᵢ, ct_aead, AD_core)
2. Check: hᵢ ?= SHA256(sᵢ || Tᵢ || i)
3. Check: Tᵢ ?= sᵢ · G
4. Verify tag: τᵢ ?= SHA256(Kᵢ || AD_core || ct_aead)
```

**Rationale:**
- SIV mode: Nonce-misuse resistant (safe even if nonce repeats)
- Standard: RFC 5297, widely implemented
- Single scheme: Ensures interoperability

**For testing/research:** Hash-only DEM allowed, but not for mainnet

#### 2. Test Vectors
**Priority:** P3, **Timeline:** 1 week

Provide reference test vectors:
```
Input:
- Kᵢ: [256-bit key, hex]
- sᵢ: [scalar, hex]
- hᵢ: [hash, hex]
- AD_core: [associated data, hex]

Output:
- ct_aead: [ciphertext, hex]
- τᵢ: [tag, hex]
```

Multiple implementations must produce bit-identical outputs.

#### 3. Deprecate Non-SIV AEADs
**Priority:** P3, **Timeline:** Immediate (documentation)

Update line 220:
```
DEPRECATED: Non-SIV AEADs with deterministic nonce
Reason: Nonce reuse risk (see PVUGC-005)

If non-SIV used (testing only):
- MUST include random nonce (defeats determinism)
- NOT RECOMMENDED
```

### Testing

#### 4. Interoperability Tests
**Timeline:** 1 week

- [ ] Multiple implementations (e.g., Rust, Go, Python) encrypt same plaintext
- [ ] Verify all produce identical ct_aead
- [ ] Cross-decrypt: Impl A encrypts, Impl B decrypts successfully
- [ ] Tag verification: All implementations compute same τᵢ

---

## Acceptance Criteria

- [ ] Single mandatory DEM (AES-SIV) specified for mainnet
- [ ] Test vectors published
- [ ] Non-SIV AEADs deprecated with clear warnings
- [ ] Interoperability tests pass across implementations

---

## Related Flaws
- **PVUGC-005:** Context binding (nonce reuse if ctx_hash repeats)
- **PVUGC-006:** Degenerate values (serialization affects both DEM and KEM)

---

**Last Updated:** 2025-10-07
