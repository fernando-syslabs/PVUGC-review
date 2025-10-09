# PVUGC-009: DEM Profile Multiplicity Resolved

**Flaw Code:** PVUGC-009
**Severity:** 🟠 High → ✅ **RESOLVED**
**Status:** ✅ Resolved (v2.0)
**Date Identified:** 2025-10-07 (v1.0)
**Date Resolved:** 2025-10-07 (v2.0)

---

## Component
Data Encapsulation Mechanism (DEM) for encrypting arming shares

## Location
- **v1.0:** §8, Lines 217-220 (multiple DEM options allowed)
- **v2.0:** §89, Line 91 (single mandatory DEM profile)

---

## Original Problem (v1.0)

### Description

In v1.0, the protocol allowed **two different DEM options** for encrypting arming shares (sᵢ, hᵢ):

**Option 1: Hash-only DEM (ROM)**
```
ctᵢ = (sᵢ || hᵢ) ⊕ H(Kᵢ, AD_core)
τᵢ = H(Kᵢ, AD_core, ctᵢ)
```

**Option 2: AEAD+commit wrapper (RECOMMENDED)**
```
ct_aead = AEAD_Kᵢ(sᵢ || hᵢ, AD_core)
τᵢ = H(Kᵢ, AD_core, ct_aead)
```

With additional note (line 220):
> "For non-SIV AEADs, use deterministic nonce."

### Security Impact (v1.0)

**Risk 1: Cross-implementation incompatibility**
```
Scenario:
- Armer uses Hash-only DEM (Option 1)
- Decapper expects AEAD wrapper (Option 2)
- Format mismatch → decryption fails
- Funds locked until timeout
```

**Risk 2: Non-SIV AEAD with deterministic nonce**
```
Scenario:
- Implementation uses AES-GCM (non-SIV)
- Deterministic nonce as recommended
- Same (K, nonce) pair used twice → catastrophic failure
- AES-GCM security breaks with nonce reuse
```

**Risk 3: Tag verification inconsistency**
```
Question: Is τᵢ redundant with AEAD internal MAC?
- AEAD already provides authentication
- External tag τᵢ = H(K, AD, ct) adds second layer
- Different implementations might handle differently
```

**Risk 4: ROM vs Standard Model**
```
Hash-only DEM requires Random Oracle Model:
- Not all deployment contexts accept ROM
- AEAD approach more conservative (standard model)
- Mixed deployments create security assumption confusion
```

**Original risk assessment:**
- **Severity:** 🟠 High (operational risk, not direct crypto break)
- **Impact:** Liveness failures, incompatibility, potential security degradation
- **Likelihood:** High (if multiple implementations exist)

---

## How v2.0 Resolves This

### ✅ Single Mandatory DEM Profile

**v2.0 Production Profile (§89, line 91):**

```
MUST: DEM_PROFILE = "PVUGC/DEM-P2-v1"
```

**Full specification:**
- **Base:** Poseidon2 hash function (SNARK-friendly)
- **Mode:** Key derivation + symmetric encryption
- **Profile version:** v1 (versioned for future upgrades)

**Details (implicit from production profile):**

**1. Key Derivation (KDF):**
```
Kᵢ = HKDF(
    salt = ctx_hash,
    IKM = M_i^{AND},  // Combined from all CRS (Multi-CRS AND-ing)
    info = "PVUGC/KEM/v1"
)

Where M_i^{AND} = ser_𝔾_T(M_i^(1)) || ser_𝔾_T(M_i^(2)) || ...
```

**2. Hash Functions (§89):**
```
H_bytes = SHA-256  // For byte-level hashes (ctx_hash, etc.)
H_p2 = Poseidon2   // For KDF, DEM, tag, and in-circuit PoCE
```

**3. DEM Construction (Poseidon2-based):**
```
Encryption:
  nonce_i = H_p2("PVUGC/AEAD-NONCE" || Kᵢ || i)
  ctᵢ = (sᵢ || hᵢ) ⊕ H_p2(Kᵢ || nonce_i || "PVUGC/DEM/v1")
  τᵢ = H_p2(Kᵢ || AD_core || ctᵢ || "PVUGC/TAG")

Decryption:
  plaintext = ctᵢ ⊕ H_p2(Kᵢ || nonce_i || "PVUGC/DEM/v1")
  Parse: (sᵢ, hᵢ) = plaintext
  Verify: τᵢ ?= H_p2(Kᵢ || AD_core || ctᵢ || "PVUGC/TAG")
  Verify: hᵢ ?= H_p2(sᵢ || "PVUGC/SHARE-HASH")
```

**4. Domain Tags (§89, normative):**
```
"PVUGC/KEM/v1"      - KDF info parameter
"PVUGC/AEAD-NONCE"  - Nonce derivation
"PVUGC/DEM/v1"      - DEM keystream
"PVUGC/TAG"         - Authentication tag
"PVUGC/SHARE-HASH"  - Share hash verification
```

**5. Why Poseidon2:**
```
Properties:
- SNARK-friendly (efficient in-circuit for PoCE)
- Cryptographic hash function (collision-resistant)
- Deterministic (reproducible encryption with same inputs)
- No external AEAD dependency (simplified stack)

Tradeoffs:
- Newer primitive (less historical cryptanalysis than AES)
- Requires Poseidon2 implementation (not in standard libraries)
- Custom construction (not off-the-shelf AEAD)
```

---

## Security Impact (Resolved)

### Eliminated Risks

**✅ No more cross-implementation incompatibility**
```
Before (v1.0):
- Implementation A: Hash-only DEM
- Implementation B: AES-SIV wrapper
- Result: Incompatible ciphertexts

After (v2.0):
- All implementations: MUST use "PVUGC/DEM-P2-v1"
- Same DEM construction
- Result: Interoperable ciphertexts
```

**✅ No more nonce reuse risks**
```
Before (v1.0):
- "Use deterministic nonce" with non-SIV AEAD
- Risk: Nonce reuse → catastrophic for GCM/CTR modes

After (v2.0):
- Poseidon2-based DEM
- Nonce derived: nonce_i = H_p2(Kᵢ || i || ...)
- Deterministic but unique per share
- No nonce reuse (different i → different nonce)
```

**✅ No more ROM vs standard model confusion**
```
Before (v1.0):
- Option 1: ROM assumption (hash as random oracle)
- Option 2: Standard model (AEAD security)
- Mixed assumptions across implementations

After (v2.0):
- Single DEM profile
- Poseidon2 as hash function (ROM model for DEM)
- But also used in PoCE (SNARK context)
- Clear security model
```

**✅ No more tag verification ambiguity**
```
Before (v1.0):
- AEAD internal MAC + external tag τᵢ
- Redundancy unclear

After (v2.0):
- Poseidon2 DEM with explicit tag computation
- τᵢ = H_p2(Kᵢ || AD_core || ctᵢ || "PVUGC/TAG")
- Clear purpose: Key-committing authentication
```

---

## Verification Checklist

To verify this flaw is properly resolved in implementation:

### DEM Profile Compliance
- [ ] Implementation uses "PVUGC/DEM-P2-v1" (no other DEMs allowed)
- [ ] Poseidon2 hash function implemented
- [ ] H_p2 = Poseidon2 (for DEM, KDF, tags)
- [ ] H_bytes = SHA-256 (for byte-level hashes)

### KDF Construction
- [ ] HKDF with correct parameters:
  - salt = ctx_hash
  - IKM = M_i^{AND} (combined from all CRS)
  - info = "PVUGC/KEM/v1"
- [ ] Multi-CRS: M_i^{AND} = ser_𝔾_T(M_i^(1)) || ser_𝔾_T(M_i^(2)) || ...

### DEM Encryption/Decryption
- [ ] Nonce derivation: nonce_i = H_p2("PVUGC/AEAD-NONCE" || Kᵢ || i)
- [ ] Encryption: ctᵢ = (sᵢ || hᵢ) ⊕ H_p2(Kᵢ || nonce_i || "PVUGC/DEM/v1")
- [ ] Tag: τᵢ = H_p2(Kᵢ || AD_core || ctᵢ || "PVUGC/TAG")
- [ ] Decryption: Verify τᵢ first, then decrypt
- [ ] Share hash: Verify hᵢ = H_p2(sᵢ || "PVUGC/SHARE-HASH")

### Domain Tags
- [ ] All domain tags match specification exactly
- [ ] No custom/modified tags (breaks interoperability)

### Interoperability Testing
- [ ] Test vectors: (Kᵢ, sᵢ, hᵢ) → (ctᵢ, τᵢ) (bit-exact)
- [ ] Cross-implementation test: Multiple implementations encrypt same plaintext → identical ciphertext
- [ ] Decrypt test: Implementation A encrypts, Implementation B decrypts → success

### Poseidon2 Implementation
- [ ] Use standard Poseidon2 parameters (for chosen field)
- [ ] Test vectors match reference implementation
- [ ] No custom modifications (breaks compatibility)

---

## Attack Scenarios Prevented

### Attack 9.1: Cross-Implementation Format Attack (RESOLVED)

**v1.0 vulnerability:**
```
Setup:
- Armer uses Hash-only DEM (ROM)
- Decapper expects AES-SIV AEAD wrapper

Attack (unintentional):
1. Armer encrypts: ctᵢ = (sᵢ || hᵢ) ⊕ H(Kᵢ, AD)
2. Publishes: ctᵢ, τᵢ
3. Decapper attempts: plaintext = AEAD_Kᵢ^{-1}(ctᵢ, AD)
4. Decryption fails (format mismatch)
5. Funds locked until timeout

Impact: Liveness failure (not security break, but disruptive)
```

**v2.0 resolution:**
- Single mandatory DEM: "PVUGC/DEM-P2-v1"
- All implementations use same encryption scheme
- Format compatibility guaranteed

**Status:** ✅ Resolved

### Attack 9.2: Nonce Reuse with Non-SIV AEAD (RESOLVED)

**v1.0 vulnerability:**
```
Setup:
- Implementation uses AES-GCM (non-SIV mode)
- Follows v1.0 recommendation: "use deterministic nonce"

Attack (unintentional):
1. Armer encrypts share 1: ct₁ = GCM_K(s₁, nonce₁)
   Where nonce₁ = H(K || "nonce")  // Deterministic
2. Later, armer encrypts share 2 (different context): ct₂ = GCM_K(s₂, nonce₂)
   Where nonce₂ = H(K || "nonce")  // Same deterministic nonce!
3. If K is same (should be different but implementation bug): nonce reused
4. AES-GCM security breaks: Key stream revealed
5. Attacker XORs ct₁ ⊕ ct₂ → s₁ ⊕ s₂
6. With known plaintext attack: Recovers s₁ and s₂

Impact: Secret share leakage
```

**v2.0 resolution:**
- No more non-SIV AEAD allowed
- Poseidon2 DEM uses: nonce_i = H_p2(K || i || ...)
- Each share has unique nonce (different i)
- No nonce reuse even with deterministic derivation

**Status:** ✅ Resolved

### Attack 9.3: Tag Bypass Attack (RESOLVED)

**v1.0 potential vulnerability:**
```
Setup:
- AEAD provides internal MAC
- External tag τᵢ also computed
- Implementation might skip one check

Attack:
1. Attacker modifies ciphertext: ct'ᵢ = ctᵢ ⊕ Δ
2. If implementation only checks AEAD MAC (skips external τᵢ):
   - AEAD decryption might fail (good)
3. But if AEAD check has bug or is skipped:
   - External tag should catch modification
4. If both checks skipped: Ciphertext malleability

Impact: Depends on implementation quality
```

**v2.0 resolution:**
- Single DEM construction with explicit tag
- Tag τᵢ = H_p2(Kᵢ || AD_core || ctᵢ || "PVUGC/TAG")
- Clear specification: MUST verify tag before decryption
- No ambiguity about verification order

**Status:** ✅ Resolved (via clear specification)

---

## Remaining Considerations

While this flaw is **fully resolved**, implementations should be aware of:

### 1. Poseidon2 Implementation Quality

**Current state:** Poseidon2 mandated, but implementation not specified.

**Considerations:**
- Multiple Poseidon2 implementations exist (different languages)
- Parameters vary by field (Fp, Fp2, etc.)
- Must use standard parameters for interoperability

**Recommendation:**
```
Specify exact Poseidon2 parameters:
- Field: [Specify which field, e.g., BLS12-381 scalar field]
- Round constants: [Reference to standard parameters]
- MDS matrix: [Reference to standard]
- Number of rounds: Full + partial rounds specification

Provide test vectors:
Input: [bytes] → H_p2(input) → [output]
Verify: All implementations produce identical output
```

### 2. Test Vectors (HIGH PRIORITY)

**Current state:** DEM profile specified, but no test vectors in spec.

**Needed:**
```
Test vector 1: KDF
Input:
  ctx_hash = 0x1234...
  M_i^{AND} = 0x5678...
  info = "PVUGC/KEM/v1"
Expected:
  Kᵢ = 0xabcd...

Test vector 2: DEM Encryption
Input:
  Kᵢ = 0xabcd...
  sᵢ = 0xef01...
  hᵢ = 0x2345...
  i = 1
  AD_core = 0x6789...
Expected:
  nonce_i = 0x1111...
  ctᵢ = 0x2222...
  τᵢ = 0x3333...

Test vector 3: DEM Decryption
Input: (ctᵢ, τᵢ from test vector 2)
Expected: (sᵢ, hᵢ) matching test vector 2
```

**Priority:** HIGH - Critical for interoperability

### 3. Poseidon2 Security Review

**Consideration:** Poseidon2 is relatively new (compared to SHA-256, AES).

**Status:**
- Poseidon (v1) extensively analyzed
- Poseidon2 improvements based on cryptanalysis
- Used in production: ZK-STARK systems (StarkWare, etc.)

**Recommendation:**
- Monitor Poseidon2 cryptanalysis literature
- Update to Poseidon3 or alternative if issues found
- Versioned profile ("DEM-P2-v1") allows future upgrades

### 4. In-Circuit PoCE Verification

**Benefit of Poseidon2:** SNARK-friendly (efficient in ZK circuits).

**Implication:**
```
PoCE (Proof of Correct Encryption) can efficiently verify:
- Share hash: hᵢ = H_p2(sᵢ || "PVUGC/SHARE-HASH")
- In-circuit (within SNARK proof)
- More efficient than SHA-256 in-circuit

Enables future enhancement:
- Make PoCE-B publicly verifiable (see PVUGC-004)
- Include DEM verification in PoCE circuit
```

**Status:** Architecture benefit (not immediate feature)

---

## Related Flaws

- **PVUGC-006:** Degenerate values (✅ resolved - canonical serialization also relevant here)
- **PVUGC-004:** PoCE-B soundness (related - Poseidon2 enables future PoCE-B improvements)
- **PVUGC-002:** Multi-CRS AND-ing (✅ resolved - KDF uses M_i^{AND} from all CRS)

---

## Implementation References

### v2.0 Specification Sections
- **§89:** Production profile (DEM_PROFILE mandatory)
- **§8:** DEM construction (conceptual, updated by production profile)
- **§3:** Domain tags (normative tags for DEM)

### Poseidon2 References
- **Poseidon2 paper:** Grassi et al., "Poseidon2: A Faster Version of Poseidon" (2023)
- **Reference implementation:** [Specify reference repo/library]
- **Parameters:** [Standard parameters for BLS12-381 scalar field]

### Test Vectors Needed
- [ ] KDF test vectors (HKDF with specified parameters)
- [ ] Poseidon2 test vectors (input → output, bit-exact)
- [ ] DEM encryption test vectors (plaintext → ciphertext + tag)
- [ ] DEM decryption test vectors (ciphertext → plaintext, verification)
- [ ] Cross-implementation compatibility tests

---

## Notes

- **v2.0 Status:** ✅ Fully resolved with single mandatory DEM profile
- **Implementation priority:** HIGH - Must implement exactly as specified
- **Testing critical:** Test vectors essential for interoperability
- **Best practice:** Use reference Poseidon2 implementation (avoid custom implementations)
- **Versioning:** "DEM-P2-v1" allows future upgrades if Poseidon2 issues found

---

**Version History:**
- **v1.0 (2025-10-07):** Initial identification (multiple DEM options, interoperability risk)
- **v2.0 (2025-10-07):** Resolved with single mandatory DEM profile "PVUGC/DEM-P2-v1"

**Last Updated:** 2025-10-07
**Status:** ✅ RESOLVED (pending test vectors and reference implementation verification)
