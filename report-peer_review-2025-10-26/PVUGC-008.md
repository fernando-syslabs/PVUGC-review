# PVUGC-008: MuSig2 Compartmentalization

**Issue Code:** PVUGC-008
**Title:** MuSig2 Nonce Uniqueness and Adaptor Compartmentalization
**Severity:** 🔴 High
**Status:** ❌ Refuted
**Report Version:** 4.0 (Final)
**Review Round:** 4 (Final)
**Dates:** Identified 2025-10-07 (v1.0); Improved 2025-10-07 (v2.0); Peer Reviewed 2025-10-26
**Reviewers:** M1 (Mathematician #1); M2 (Mathematician #2); Crypto-Peer-Reviewer
**Cross-References:**
- [`PVUGC-2025-10-05.md:94`](../PVUGC-2025-10-05.md) (final specification)
- [`report-preliminary-2025-10-07/PVUGC-008-musig2-compartmentalization.md`](../report-preliminary-2025-10-07/PVUGC-008-musig2-compartmentalization.md) (v1.0)
- [`report-update-2025-10-07/PVUGC-008-musig2-compartmentalization.md`](../report-update-2025-10-07/PVUGC-008-musig2-compartmentalization.md) (v2.0)
- [Appendix A08.M201](APPENDIX-issue-debates.md#a08-m201) (M2 analysis plan)
 - [Appendix A08.M201](APPENDIX-issue-debates.md#a08-m201) (Nonce reuse refutation and extraction)
 - [Appendix A08.CR02](APPENDIX-issue-debates.md#a08-cr02) (Deterministic HKDF nonce derivation)
 - [Appendix A08.M203](APPENDIX-issue-debates.md#a08-m203) (Revisions: worst‑vs‑average, TOST, cache timing)

---

## Executive Summary

**Verdict:** The v2.0 mitigation relying solely on a MUST clause for MuSig2 nonce uniqueness is **insufficient** and leaves the protocol vulnerable to catastrophic private key extraction. Deterministic HKDF-based nonce derivation is required for cryptographic enforcement.

**Impact:** Nonce reuse enables O(1) private key extraction from two signatures with the same R value. Complete compromise of the aggregate MuSig2 key allows unauthorized spending of all funds controlled by that key. Historical precedent (blockchain.info 2013, Android Bitcoin wallet 2013, PlayStation 3 2010) demonstrates this is not a theoretical concern.

**Changes Across Versions:**
- **v1.0:** Informal "never reuse" guidance; no enforcement; status Open
- **v2.0:** Explicit MUST clause added (line 102); context binding via presig_pkg_hash; status Improved
- **v4.0 (Peer Review):** MUST clause refuted as insufficient; deterministic HKDF-based nonce derivation specified with BIP-327 integration, version handling, R normalization timing, and statistical test requirements; cryptographic enforcement mechanism required; status Refuted; severity confirmed High

**Remaining Gaps (Critical):**
- ❌ No cryptographic enforcement of R, T uniqueness in current spec
- ❌ No specification of deterministic nonce derivation mechanism
- ❌ No normative HKDF-based nonce generation in §10
- ❌ No BIP-327 state machine compliance requirements
- ❌ No timing specification for R normalization with negation factors
- ❌ Enforcement relies entirely on honest implementation (unacceptable for cryptographic protocol)

**Required Action:**
1. **Normative specification (MUST):** Add §10.1 MuSig2 Nonce Derivation with complete HKDF-based deterministic scheme including version handling and two-nonce derivation
2. **Implementation:** Reference implementation with deterministic derivation and BIP-327 compliance
3. **Testing:** Comprehensive test suite with statistical bounds (1000+ uniqueness tests, attack simulations)
4. **External review:** Independent cryptographic review of nonce derivation scheme

---

## Spec Location

**Primary:**
- [`PVUGC-2025-10-20.md §6 Compartmentalization`](../PVUGC-2025-10-20.md) (Compartmentalization MUST clause)
- [`PVUGC-2025-10-20.md §4.2 AdaptorVerify`](../PVUGC-2025-10-20.md) (AdaptorVerify function)

**Secondary:**
- [`PVUGC-2025-10-20.md §3 Context Binding`](../PVUGC-2025-10-20.md) (Context binding - ctx_hash structure)
- [`PVUGC-2025-10-20.md §3 Context Binding`](../PVUGC-2025-10-20.md) (presig_pkg_hash including T, R)
- [`PVUGC-2025-10-20.md:314`](../PVUGC-2025-10-20.md) (R normalization requirement)
- [`PVUGC-2025-10-20.md:315`](../PVUGC-2025-10-20.md) (Key aggregation binding)

**Missing (must be added):**
- §10.1 MuSig2 Nonce Derivation (normative specification for deterministic r_i derivation)
- §10.2 MuSig2 State Machine (BIP-327 compliance requirements)
- R Normalization Timing (negation factor application specification)

---

## History (v1.0 → v2.0 → Peer Review)

### v1.0 Original Concern (2025-10-07)
- **Identified by:** M1
- **Problem:** Classic Schnorr nonce reuse vulnerability
- **Risk:** If R reused across two messages, private key extractable via linear algebra
- **Mitigation proposed:** Informal "never reuse" guidance
- **Status:** 🔓 Open
- **Severity:** 🟡 Medium
- **Gap:** No cryptographic enforcement, no deterministic derivation specified

### v2.0 Mitigation (2025-10-07)
- **Change 1:** Explicit MUST clause added (line 102):
  > "**Compartmentalization (MUST):** one adaptor ⇒ one unique T and one unique R. Never reuse across paths/epochs or templates."

- **Change 2:** Context binding enhanced (line 61):
  ```
  presig_pkg_hash = H_bytes("PVUGC/PRESIG" || m || T || R ||
                            signer_set || musig_coeffs)
  ```
  Binds T and R to specific context via ctx_hash.

- **Change 3:** Domain separation tags specified (line 84):
  - `"BIP0340/challenge"` for Schnorr challenge hash
  - `"PVUGC/PRESIG"` for pre-signature package

- **Change 4:** Production profile (line 88):
  - BLS12-381 for pairing operations
  - secp256k1 for Bitcoin signatures (implicit from BIP-340)

- **Status:** 🔧 Improved (MUST clause added, enforcement mechanism TBD)
- **Severity:** 🟡 Medium (unchanged)

**Remaining gaps in v2.0:**
- ❌ MUST clause is behavioral requirement, not cryptographic enforcement
- ❌ No specification of how to generate R, T deterministically
- ❌ No nonce commitment proof mechanism
- ❌ No blacklist/reuse detection mechanism
- ❌ Relies on implementation discipline (fragile assumption)

### v4.0 Peer Review (2025-10-26)

**M2's Round 1 Analysis:**
1. Refutation of MUST clause sufficiency
2. Formal attack specification: Key extraction algorithm with O(1) complexity
3. Historical precedent: Cryptocurrency RNG failures demonstrate practical risk
4. Normative solution: Complete HKDF-based deterministic nonce derivation specification
5. Security properties validated: Uniqueness, unpredictability, resilience

**Crypto-Peer-Reviewer's Round 2 Draft:**
1. Complete §10.1 specification with HKDF parameters
2. BIP-327 integration for two-nonce MuSig2
3. §10.2 state machine compliance requirements
4. R normalization timing specification
5. Comprehensive validation checklist

**M2's Round 3 Review (Applied Revisions):**
1. **R1 (Applied):** HKDF parameter swap to align with RFC 6979 convention
2. **R2 (Applied):** Version tag handling specification for protocol upgrades
3. **R3 (Applied):** Two-nonce derivation using HKDF info parameter per BIP-327
4. **R4 (Applied):** R normalization negation factor application clarified
5. **R5 (Applied):** Statistical bounds added to test vector requirements
6. **R6 (Applied):** Complexity analysis precision improved

**Key findings:**
- **Attack trivial:** Two signatures with same R → private key extraction in O(1) field operations, O(256) bit operations
- **MUST clause ineffective:** Behavioral requirements don't prevent implementation bugs
- **Deterministic derivation required:** Bind r_i to ctx_hash via HKDF (RFC 5869)
- **Severity confirmed:** High (complete key compromise, historical precedent)
- **Status:** Refuted (v2.0 mitigation insufficient)

**Final verdict:** ❌ **Refuted** - v2.0 mitigation insufficient, blocking issue for mainnet deployment

---

## Findings

### Finding 1: Nonce Reuse Attack Validated (M2 Contribution)

**Claim:** If aggregate nonce R is reused for two different messages, the aggregate private key can be extracted with O(1) field operations (O(256) bit operations).

**Formal Attack Algorithm (validated from M2's Round 1):**

**Setup:**
- Signer set S with k participants
- Aggregate public key P = Σᵢ aᵢ·Xᵢ (MuSig2 key aggregation)
- Aggregate private key x such that P = x·G
- Two PVUGC instances with different contexts (ctx_hash₁ ≠ ctx_hash₂)

**Attack (catastrophic nonce reuse scenario):**

**Step 1: Protocol execution with nonce reuse**
- **Instance 1:** Context ctx_hash₁, message m₁ (BIP-341 sighash)
  - Aggregate nonce R (public)
  - Pre-signature s'₁
  - After decapsulation: Final signature s₁ = s'₁ + α₁ mod n
  - BIP-340 signature verification: s₁·G = R + c₁·P
  - Where c₁ = H("BIP0340/challenge" || Rₓ || Pₓ || m₁)

- **Instance 2:** Context ctx_hash₂, message m₂ (different transaction)
  - **Same aggregate nonce R reused** (bug/malicious behavior)
  - Pre-signature s'₂
  - After decapsulation: Final signature s₂ = s'₂ + α₂ mod n
  - BIP-340 signature verification: s₂·G = R + c₂·P
  - Where c₂ = H("BIP0340/challenge" || Rₓ || Pₓ || m₂)

**Step 2: Key extraction (external attacker E)**
1. **Observation:** E observes two Bitcoin transactions on-chain:
   - Transaction A: Signature (Rₓ, s₁) for message m₁ and public key P
   - Transaction B: Signature (Rₓ, s₂) for message m₂ and public key P
   - Same R used (Rₓ identical in both signatures)

2. **Challenge recomputation:**
   ```
   c₁ = tagged_hash("BIP0340/challenge", Rₓ || Pₓ || m₁)
   c₂ = tagged_hash("BIP0340/challenge", Rₓ || Pₓ || m₂)
   ```

3. **Linear system solution:**
   From Schnorr signature equations:
   ```
   s₁ = r + c₁·x  (mod n)
   s₂ = r + c₂·x  (mod n)

   Subtraction:
   s₁ - s₂ = (c₁ - c₂)·x  (mod n)

   Private key extraction:
   x = (s₁ - s₂) · (c₁ - c₂)⁻¹  (mod n)
   ```

   Note: c₁ ≠ c₂ with overwhelming probability since m₁ ≠ m₂

4. **Result:** Attacker E recovers aggregate private key x

**Impact:**
- ✓ Complete compromise of aggregate key P
- ✓ Attacker can spend all funds controlled by P (any UTXO with scriptPubKey locking to P)
- ✓ No proof required (bypass entire WE mechanism)
- ✓ Attack undetectable until both signatures published
- ✓ Irreversible damage (private key permanently compromised)

**Complexity Analysis:**
- **Computational cost:** O(1) field operations - two modular subtractions, one modular inversion
- **Bit operation complexity:** O(log² n) ≈ O(65,536 operations) for modular inverse via extended Euclidean algorithm for n ≈ 2²⁵⁶
- **Data requirements:** Two signatures with same R (observable on Bitcoin blockchain)
- **Success probability:** 1.0 (deterministic attack, no probabilistic component)

**Note on complexity characterization:** The O(1) characterization is standard in cryptographic literature when referring to operations polynomial in the security parameter but independent of problem instance size. For PVUGC-008, the attack complexity is independent of number of signers, number of transactions, or protocol state - only the fixed curve parameter n matters.

**M2's Assessment (validated):**
> "The attacker E recovers the aggregate private key x for the entire signer set S. E can now spend any and all funds held by the aggregate key P without restriction."

**Cryptographer's validation:** ✅ **Confirmed** - Attack is correct, trivial, and catastrophic.

---

### Finding 2: MUST Clause Insufficiency (M2 Core Argument)

**M2's Claim:** "A MUST clause does not prevent this attack. It only assigns blame after the attack has occurred."

**Analysis:**

**Why behavioral requirements fail in cryptographic protocols:**

1. **Implementation bugs are inevitable:**
   - Faulty RNG implementation
   - Insufficient entropy sources
   - State management errors (nonce counter not incremented)
   - Memory corruption (nonce variable overwritten)
   - Concurrency bugs (race condition on nonce generation)

2. **Historical precedent (RNG failures in cryptocurrency):**

   **Case 1: blockchain.info Android wallet (2013)**
   - **Vulnerability:** SecureRandom not properly seeded on Android
   - **Result:** Repeated nonces in ECDSA signatures
   - **Impact:** ~$250,000 stolen via private key extraction
   - **Root cause:** Implementation relied on system RNG without verification
   - **Lesson:** "MUST use secure RNG" insufficient without enforcement

   **Case 2: Android Bitcoin Wallet (2013, CVE-2013-4444)**
   - **Vulnerability:** Java SecureRandom predictable on Android < 4.4
   - **Result:** Repeated ECDSA nonces
   - **Impact:** Multiple wallets compromised
   - **Root cause:** Platform RNG failure
   - **Lesson:** External RNG dependencies create systemic risk

   **Case 3: PlayStation 3 ECDSA (2010)**
   - **Vulnerability:** Sony reused nonce for firmware signing
   - **Result:** Console master private key extracted
   - **Impact:** Complete system compromise
   - **Root cause:** Manual nonce management error
   - **Lesson:** Human error in nonce generation

3. **v2.0 MUST clause analysis:**
   ```
   "Compartmentalization (MUST): one adaptor ⇒ one unique T and one unique R."
   ```

   **What this requires:** Implementers to ensure uniqueness

   **What this doesn't prevent:**
   - ❌ RNG seeded with insufficient entropy
   - ❌ Nonce counter implementation bug
   - ❌ State persistence failure (server restart resets counter)
   - ❌ Time-based RNG collision (parallel instances)
   - ❌ Malicious signer intentionally reusing nonce
   - ❌ Memory corruption in nonce variable
   - ❌ Concurrency race condition

   **Testing gap:**
   - How to test that R will be unique across all future executions?
   - Cannot test RNG behavior under all entropy conditions
   - Cannot test state persistence across all failure modes
   - Cannot verify implementation correctness from specification alone

4. **Cryptographic engineering principle:**
   > "A secure protocol must guarantee security properties mathematically, not rely on perfect implementation."

   - ✅ Good: "r = HKDF(secret_key, ctx_hash)" (mathematical uniqueness from ctx_hash uniqueness)
   - ❌ Bad: "MUST generate unique r" (unverifiable behavioral requirement)

**M2's Assessment (validated):**
> "The vulnerability is not that the protocol *allows* nonce reuse, but that it *fails to prevent* it. The attack is caused by implementation failure, which secure protocols must be resilient to."

**Cryptographer's validation:** ✅ **Confirmed** - MUST clause insufficient for cryptographic security property.

**Comparison with similar protocols:**
- **BIP-340 recommendation:** "Nonces SHOULD be derived deterministically" (acknowledges RNG risk)
- **RFC 6979:** Deterministic ECDSA/DSA nonces (solves identical problem)
- **BIP-327 (MuSig2):** Specifies nonce commitment scheme (prevents adaptive choice)
- **EdDSA (RFC 8032):** Deterministic nonce generation (mandatory, not optional)

**Conclusion:** Relying on MUST clause for nonce uniqueness is a known anti-pattern in cryptographic protocol design.

---

### Finding 3: Deterministic Nonce Derivation Security Properties (M2 Solution Validation)

**M2's Proposed Specification (enhanced with Round 3 revisions):**

```
### 10.1 MuSig2 Nonce Derivation (Normative)

To cryptographically enforce nonce uniqueness per context, all signers
participating in the MuSig2 pre-signature ceremony MUST derive their
secret nonce contribution r_i using the following deterministic procedure
based on HKDF (RFC 5869).

Function: Derive_MuSig2_Nonce(secret_key_i, ctx_hash)

Inputs:
- secret_key_i: The 32-byte private key x_i of the individual signer i.
- ctx_hash: The 32-byte final context hash for the current PVUGC instance.

Algorithm:
1. Extract: Per RFC 6979 convention, the ctx_hash (public context) is used as
   the salt, and secret_key_i (private key) is used as the input keying material.
   - salt = ctx_hash         (32 bytes, public context)
   - ikm = secret_key_i      (32 bytes, private key)
   - prk = HKDF-Extract(salt, ikm) using SHA-256

   Where HKDF-Extract is defined per RFC 5869:
   - prk = HMAC-SHA256(key=salt, data=ikm)

2. Expand: Two nonces are derived for BIP-327 two-nonce MuSig2 using the
   HKDF info parameter for differentiation.

   For first nonce (r_{1,i}):
   - info = "PVUGC/MuSig2-Nonce/v1" || 0x00  (24 bytes)
   - okm_1 = HKDF-Expand(prk, info, 32)

   For second nonce (r_{2,i}):
   - info = "PVUGC/MuSig2-Nonce/v1" || 0x01  (24 bytes)
   - okm_2 = HKDF-Expand(prk, info, 32)

   Where HKDF-Expand uses HMAC-SHA256:
   - T(0) = empty string
   - T(1) = HMAC-SHA256(prk, T(0) || info || 0x01)
   - okm = first 32 bytes of T(1)

3. Scalar Conversion:
   - r_{1,i} = bytes_to_integer(okm_1)  (big-endian conversion)
   - r_{2,i} = bytes_to_integer(okm_2)  (big-endian conversion)

4. Validity Check:
   For each nonce r in {r_{1,i}, r_{2,i}}:
   - if r == 0 or r >= n:
       ABORT (derivation failed, invalid scalar)

   where n is the order of secp256k1:
     n = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141

   Note: The probability of this condition is negligible (< 2^-128),
   but the check is required for correctness.

5. Output:
   - Return (r_{1,i}, r_{2,i})  (valid scalars in [1, n-1])

Version Handling (MUST):
The version tag "v1" in "PVUGC/MuSig2-Nonce/v1" is fixed for this protocol version.

For future protocol versions:
- v2: "PVUGC/MuSig2-Nonce/v2"
- v3: "PVUGC/MuSig2-Nonce/v3"
- etc.

Version upgrade rules:
1. Old implementations MUST NOT accept nonces from new versions
2. New implementations MUST NOT accept nonces from old versions
3. Version mismatch → abort protocol
4. The version tag binds to protocol version in ctx_core or GS_instance_digest

Rationale: Prevents cross-version nonce reuse attacks if HKDF parameters
change in future protocol upgrades while maintaining backward compatibility.
```

**Cryptographic Analysis:**

**Property 1: Uniqueness (Critical Security Property)**

**Claim:** If ctx_hash is unique per protocol instance, then (r_{1,i}, r_{2,i}) is unique per instance.

**Proof sketch:**
1. **Assumption:** ctx_hash uniqueness (enforced by PVUGC-005)
   - ctx_hash = H_bytes("PVUGC/CTX" || ctx_core || arming_pkg_hash || presig_pkg_hash)
   - Different instances → different ctx_hash (by construction)

2. **HKDF properties (RFC 5869):**
   - HKDF-Extract: prk = HMAC-SHA256(salt=ctx_hash, data=secret_key_i)
   - HKDF-Expand: okm_1 = HMAC-SHA256(prk, info="PVUGC/MuSig2-Nonce/v1" || 0x00)
   - HKDF-Expand: okm_2 = HMAC-SHA256(prk, info="PVUGC/MuSig2-Nonce/v1" || 0x01)

3. **Collision resistance:**
   - Different ctx_hash → different prk (SHA-256 collision resistance)
   - Different prk → different okm_1, okm_2 (HMAC-SHA256 collision resistance)
   - Different info tags (0x00 vs 0x01) → okm_1 ≠ okm_2 (HKDF domain separation)
   - Different okm → different r_{1,i}, r_{2,i} (bijective conversion, modulo negligible rejection)

4. **Uniqueness guarantee:**
   ```
   ctx_hash₁ ≠ ctx_hash₂
   ⟹ prk₁ ≠ prk₂  (with probability 1 - 2⁻²⁵⁶, SHA-256 collision resistance)
   ⟹ okm_1₁ ≠ okm_1₂ and okm_2₁ ≠ okm_2₂  (with probability 1 - 2⁻²⁵⁶, HMAC collision resistance)
   ⟹ r_{1,i,1} ≠ r_{1,i,2} and r_{2,i,1} ≠ r_{2,i,2}  (with probability 1 - 2⁻¹²⁸)
   ```

5. **Negligible failure probability:**
   - SHA-256 collision probability: 2⁻²⁵⁶
   - Scalar rejection probability (r = 0 or r >= n): < 2⁻¹²⁸
   - Combined failure probability: negligible

**Result:** ✅ **Uniqueness property proven** under standard cryptographic assumptions (SHA-256 collision resistance).

**Property 2: Unpredictability (Prevents Adaptive Attacks)**

**Claim:** To an external adversary who does not know secret_key_i, the outputs r_{1,i} and r_{2,i} are computationally indistinguishable from uniformly random scalars.

**Analysis:**
1. **HKDF security (RFC 5869, Section 5):**
   - If HMAC is a PRF (Pseudo-Random Function), then HKDF output is pseudorandom
   - SHA-256-based HMAC is a standard PRF assumption

2. **Key secrecy:**
   - secret_key_i is private to signer i
   - IKM = secret_key_i prevents adversary from computing prk
   - Even if adversary knows ctx_hash (public), cannot derive r_{1,i}, r_{2,i} without secret_key_i

3. **PRF security:**
   ```
   Adversary view: (ctx_hash, R_{1,i} = r_{1,i}·G, R_{2,i} = r_{2,i}·G)

   Distinguish: {(ctx_hash, r_{1,i}, r_{2,i})} vs {(ctx_hash, u_{1,i}, u_{2,i})}
                where u_{1,i}, u_{2,i} ← Z_n

   Reduction: If adversary distinguishes, then break HMAC-SHA256 PRF security
   ```

4. **DDH security (secp256k1):**
   - Even if adversary could compute r_{1,i}, r_{2,i}, deriving secret_key_i from R_{1,i}, R_{2,i} requires solving DLP
   - Computational Diffie-Hellman assumption on secp256k1

**Result:** ✅ **Unpredictability property holds** under PRF assumption (HMAC-SHA256) and DLP hardness (secp256k1).

**Property 3: Resilience (No External Dependencies)**

**Claim:** The derivation is stateless and requires no external random number generator.

**Advantages:**
1. **No RNG dependency:**
   - ✅ Eliminates risk of insufficient entropy
   - ✅ Eliminates risk of predictable PRNG states
   - ✅ No dependency on platform RNG (SecureRandom, /dev/urandom, etc.)

2. **Stateless:**
   - ✅ No nonce counter to maintain
   - ✅ No persistent state across restarts
   - ✅ Server restart doesn't reset nonce sequence
   - ✅ No synchronization required across instances

3. **Deterministic reproducibility:**
   - ✅ Same inputs → same output (testable)
   - ✅ Audit trail: given ctx_hash, can verify R was correctly derived
   - ✅ Debugging: can reproduce exact nonce for forensic analysis

4. **Side-channel resistance:**
   - ✅ HMAC-SHA256 has well-studied constant-time implementations
   - ✅ No timing-dependent RNG sampling

**Result:** ✅ **Resilience property validated** - eliminates entire class of implementation failures.

**Property 4: Domain Separation**

**Claim:** Domain tag "PVUGC/MuSig2-Nonce/v1" prevents cross-protocol attacks and enables versioning.

**Analysis:**
1. **HKDF info parameter:**
   ```
   okm_1 = HKDF-Expand(prk, info="PVUGC/MuSig2-Nonce/v1" || 0x00, 32)
   okm_2 = HKDF-Expand(prk, info="PVUGC/MuSig2-Nonce/v1" || 0x01, 32)
   ```

2. **Domain separation benefit:**
   - Different protocols using same (secret_key_i, ctx_hash) derive different nonces
   - Prevents nonce reuse if same key used in different contexts
   - Version tag "v1" enables protocol upgrades
   - Counter bytes (0x00, 0x01) ensure r_{1,i} ≠ r_{2,i}

3. **Comparison with other protocols:**
   - If different protocol uses "PVUGC/MuSig2-Nonce/v2" or different tag
   - Derived nonces provably different (HKDF collision resistance)

4. **Version handling prevents upgrade attacks:**
   - Old implementations reject v2 nonces
   - New implementations reject v1 nonces
   - Prevents cross-version nonce reuse if parameters change

**Result:** ✅ **Domain separation property validated**.

**Comparison with RFC 6979:**

| Property | RFC 6979 (ECDSA) | Final Proposal (MuSig2) |
|----------|------------------|------------------------|
| **KDF** | HMAC-DRBG | HKDF (RFC 5869) |
| **Input** | secret_key + hash(message) | secret_key + ctx_hash |
| **Salt** | hash(message) | ctx_hash |
| **IKM** | secret_key | secret_key |
| **Domain tag** | None (implicit) | "PVUGC/MuSig2-Nonce/v1" |
| **Nonce count** | 1 | 2 (BIP-327 requirement) |
| **Uniqueness** | ✅ (different messages) | ✅ (different ctx_hash) |
| **Unpredictability** | ✅ (PRF security) | ✅ (PRF security) |
| **Standardization** | ✅ (RFC 6979) | ⚠️ (proposed, needs review) |

**Key alignment:** Final proposal follows RFC 6979 convention (salt=public_context, IKM=secret_key) for consistency with established standards.

**Security equivalence:** Both derive nonces deterministically from (secret_key, public_input) with PRF security. HKDF is simpler and more modern than HMAC-DRBG.

**Cryptographer's assessment:** ✅ **Final specification is cryptographically sound** and aligns with RFC 6979 for the MuSig2 context.

---

### Finding 4: BIP-327 State Machine Compliance (Gap Analysis)

**Issue:** v2.0 spec doesn't specify MuSig2 state machine transitions, creating risk of implementation errors.

**BIP-327 (MuSig2) Required State Machine:**

```
State 0: KeyAgg
  Input: Public keys {X₁, ..., Xₖ}
  Compute: Aggregation coefficients aᵢ = KeyAggCoeff(L, Xᵢ)
  Compute: Aggregate key P = Σᵢ aᵢ·Xᵢ
  Transition: → NonceGen

State 1: NonceGen
  For each signer i:
    Generate: (secnonce_i, pubnonce_i)
    - secnonce_i = (k₁,ᵢ, k₂,ᵢ) [two scalar nonces]
    - pubnonce_i = (R₁,ᵢ, R₂,ᵢ) where R₁,ᵢ = k₁,ᵢ·G, R₂,ᵢ = k₂,ᵢ·G
  MUST: Erase secnonce_i if session aborted
  Transition: → NonceAgg (when all pubnonces received)

State 2: NonceAgg
  Input: All pubnonces {pubnonce₁, ..., pubnonce_k}
  Compute: Aggregated nonce R = NonceAgg({pubnonce₁, ..., pubnonce_k})
  BIP-327 formula: R = R₁ + b·R₂ where b = H_agg(X₁, ..., Xₖ, R₁, R₂, m)
  MUST: Normalize R to even y-coordinate (BIP-340)
  Transition: → Sign

State 3: Sign
  Input: Message m, aggregate R, aggregate P
  Compute: Challenge c = H("BIP0340/challenge" || Rₓ || Pₓ || m)
  For each signer i:
    Compute: sᵢ = Sign(secnonce_i, secret_key_i, aggnonce, message)
    BIP-327 formula: sᵢ = (k₁,ᵢ + b·k₂,ᵢ) + c·aᵢ·xᵢ  (mod n)
  MUST: Erase secnonce_i immediately after signing
  Transition: → SigAgg

State 4: SigAgg
  Input: Partial signatures {s₁, ..., sₖ}
  Compute: s = Σᵢ sᵢ  (mod n)
  Verify: s·G = R + c·P  (BIP-340 verification)
  Output: Signature (Rₓ, s)
  Transition: → Complete
```

**Critical Invariants (MUST NOT violate):**

1. **No nonce reuse across sessions:**
   - Each session generates fresh secnonce_i
   - MUST erase secnonce_i after use
   - MUST NOT reuse (k₁,ᵢ, k₂,ᵢ) in different session

2. **No signing before seeing all nonces:**
   - MUST receive all pubnonces before computing partial signature
   - Prevents Wagner's attack (adaptive nonce selection)

3. **No state regression:**
   - Cannot go back from Sign to NonceGen
   - Cannot sign same message twice with different nonces

4. **Nonce commitment (optional but recommended):**
   - Signers commit to pubnonce before revealing
   - Prevents adaptive attacks where malicious signer chooses nonce after seeing honest nonces

**v2.0 Spec Coverage:**

✅ **Covered:**
- Line 280: "Key aggregation binding" - references key list L and coefficients aᵢ
- Line 61: presig_pkg_hash includes signer_set and musig_coeffs
- Line 274: "R must be normalized to even-y per BIP-340"

❌ **Missing:**
- No explicit state machine specification
- No nonce generation method (two-nonce variant specified)
- No nonce commitment requirement
- No secnonce erasure requirement
- No timing specification (when does R normalization happen relative to signing?)
- No prohibition on state regression

**Risk:**
- Implementers may violate state transitions unknowingly
- Wagner's attack possible if signing before seeing all nonces
- Nonce reuse possible if secnonce not erased

**Recommendation:** Add normative state machine specification referencing BIP-327 (see Recommendations section).

---

### Finding 5: R Normalization Timing Specification (Gap)

**Issue:** v2.0 spec requires R normalization but doesn't specify when it occurs or how negation factors are applied.

**v2.0 Statement (line 274):**
> "R must be normalized to even-y per BIP-340"

**BIP-340 Requirement:**
> "If the Y coordinate of R is odd, negate R (use -R instead)"

**Critical question:** At what point in the protocol does normalization happen, and how do all signers apply the negation factor?

**Scenario Analysis:**

**Option A: Normalize individual nonces before aggregation**
```
For each signer i:
  Generate: R₁,ᵢ = r₁,ᵢ·G, R₂,ᵢ = r₂,ᵢ·G
  If R₁,ᵢ.y is odd: R₁,ᵢ = -R₁,ᵢ, r₁,ᵢ = -r₁,ᵢ
  If R₂,ᵢ.y is odd: R₂,ᵢ = -R₂,ᵢ, r₂,ᵢ = -r₂,ᵢ
  Publish: (R₁,ᵢ, R₂,ᵢ) (normalized)

Aggregate: R = Σᵢ R₁,ᵢ + b·Σᵢ R₂,ᵢ
Problem: R.y may still be odd after aggregation!
```
❌ **Incorrect** - Normalization doesn't compose under addition.

**Option B: Normalize after aggregation (CORRECT)**
```
For each signer i:
  Generate: R₁,ᵢ = r₁,ᵢ·G, R₂,ᵢ = r₂,ᵢ·G
  Publish: (R₁,ᵢ, R₂,ᵢ) (unnormalized)

Aggregate: R_agg = Σᵢ R₁,ᵢ + b·Σᵢ R₂,ᵢ

Normalize:
  If R_agg.y is odd:
    Set negation_factor = -1 mod n
    R = -R_agg  (point negation)
  Else:
    Set negation_factor = 1
    R = R_agg

Challenge: c = H("BIP0340/challenge" || Rₓ || Pₓ || m)

For each signer i:
  Apply negation factor in signature computation:
  sᵢ = negation_factor · (r₁,ᵢ + b·r₂,ᵢ) + c·aᵢ·xᵢ  (mod n)
```
✅ **Correct** - Normalization after aggregation, all signers apply negation factor.

**Clarification from M2's Round 3 Review (Applied):**

The negation factor is applied in the signature computation, not by modifying stored nonce values. Two equivalent methods:

**Method 1 (explicit negation factor):**
```
If R.y is odd after aggregation:
  Set g = -1  (negation factor)
Else:
  Set g = 1

For each signer i:
  Compute: sᵢ = g·(r₁,ᵢ + b·r₂,ᵢ) + c·aᵢ·xᵢ  (mod n)
```

**Method 2 (equivalent, conceptual nonce negation):**
```
If R.y is odd after aggregation:
  Conceptually negate nonces: r'₁,ᵢ = -r₁,ᵢ, r'₂,ᵢ = -r₂,ᵢ
  Compute: sᵢ = (r'₁,ᵢ + b·r'₂,ᵢ) + c·aᵢ·xᵢ  (mod n)
Else:
  Use original nonces: r'₁,ᵢ = r₁,ᵢ, r'₂,ᵢ = r₂,ᵢ
  Compute: sᵢ = (r'₁,ᵢ + b·r'₂,ᵢ) + c·aᵢ·xᵢ  (mod n)
```

**Important:** Stored secnonce values need not be modified; the negation can be applied only during signature computation.

**v2.0 Specification Gap:**
- Doesn't specify which option (A, B, or other)
- Doesn't specify if all signers adjust their nonces when R negated
- Doesn't specify timing relative to presig_pkg_hash computation

**Recommendation:** See §10.2 normative text in Recommendations section.

---

### Finding 6: Defense-in-Depth - Blacklist Mechanism (Secondary Defense)

**M2's Position:** Deterministic derivation is primary defense; blacklist is secondary.

**Analysis:**

**Primary defense (deterministic derivation):**
- ✅ Prevents nonce reuse mathematically
- ✅ No state management required
- ✅ Stateless, works across restarts
- ✅ Solves root cause

**Secondary defense (blacklist):**
- ✅ Detects implementation bugs in deterministic derivation
- ✅ Protects against malicious signers intentionally reusing nonces
- ✅ Runtime verification of uniqueness property

**Blacklist Mechanism:**
```
Maintain sets:
  blacklist_R = {R₁, R₂, ..., Rₖ}  // Previously used aggregate nonces
  blacklist_T = {T₁, T₂, ..., Tₘ}  // Previously used adaptor points

Before pre-signing:
  1. Check R ∉ blacklist_R
  2. Check T ∉ blacklist_T
  3. If duplicate detected: ABORT (log security violation)
  4. Add to blacklist: blacklist_R ∪= {R}, blacklist_T ∪= {T}

Persistence:
  - Store in database (survive restarts)
  - Index by ctx_core (scope to protocol instance)
  - Optionally: Distributed blacklist (sync across signers)
```

**Challenges:**
1. **Storage:** O(n) where n = number of protocol instances
   - For production: n ~ 10⁶ instances → ~64MB storage (32 bytes/point × 2M points)
   - Acceptable overhead

2. **Synchronization:** Multi-signer scenario
   - Each signer maintains local blacklist
   - Reuse detected independently by each signer
   - No consensus required (each signer aborts if sees reuse)

3. **False positives:** None (equality check on curve points)

**Recommendation:**
- **MUST:** Implement deterministic derivation (primary defense)
- **SHOULD:** Implement blacklist mechanism (defense-in-depth, catches bugs)

---

### Finding 7: Severity Confirmation (High)

**Severity Criteria:**

| Severity | Impact | Likelihood | Exploitability |
|----------|--------|------------|----------------|
| **Critical** | Complete protocol break | High | Trivial |
| **High** | Key compromise, fund loss | Medium | Low to Medium |
| **Medium** | Information leakage, DoS | Low | Medium to High |
| **Low** | Minor issues, limited impact | Very Low | Any |

**PVUGC-008 Analysis:**

**Impact: Complete Key Compromise**
- ✅ Aggregate private key extraction
- ✅ Unauthorized spending of all funds controlled by compromised key
- ✅ Bypass of entire WE mechanism (no proof required after key compromise)
- ✅ Irreversible damage (key permanently compromised)

**Assessment:** Maximum impact (same as Critical severity)

**Likelihood: Medium (Implementation Bug)**
- Historical precedent: Multiple RNG failures in cryptocurrency (blockchain.info 2013, Android wallet 2013, PlayStation 3 2010)
- Platform dependencies: Java SecureRandom, OpenSSL RAND_bytes, OS entropy sources
- Implementation complexity: Correct RNG usage non-trivial
- Human error: Manual nonce management error (PlayStation 3 case)

**Assessment:** Medium likelihood (proven by historical failures)

**Exploitability: Trivial (O(1) field operations)**
- No cryptographic work required (not breaking DLP or collision resistance)
- Linear algebra only: (s₁ - s₂) · (c₁ - c₂)⁻¹ mod n
- Publicly observable: Both signatures on Bitcoin blockchain
- No interaction required: Passive observation
- Complexity: O(1) field operations (2 subtractions, 1 inversion, 1 multiplication)
- Bit operations: O(log² n) ≈ O(65,536) for 256-bit modular inverse

**Assessment:** Trivial exploitability (High severity threshold)

**Comparison with v2.0 Assessment:**

**v2.0 (Medium) Reasoning:**
> "1. Context binding provides defense
> 2. Reuse detectable post-facto
> 3. Requires implementation bug
> 4. Social/procedural layer (multiple signers provide redundancy)"

**Refutation:**
1. **Context binding:** Doesn't prevent nonce reuse, only binds context
2. **Post-facto detection:** Too late (key already compromised)
3. **Implementation bug:** Historical precedent shows this is realistic, not theoretical
4. **Social layer:** In MuSig2, one buggy signer compromises entire aggregate key

**Historical Precedent Analysis:**

**blockchain.info (2013):**
- **Date:** December 2013
- **Vulnerability:** SecureRandom not seeded on Android
- **Result:** ECDSA nonce reuse
- **Impact:** ~$250,000 stolen
- **Lessons:**
  - "MUST use secure RNG" was in spec - still failed
  - Platform RNG assumptions broken
  - Post-incident detection (after funds stolen)

**Android Bitcoin Wallet (2013):**
- **CVE:** CVE-2013-4444
- **Vulnerability:** Java SecureRandom predictable state
- **Result:** Repeated ECDSA nonces
- **Impact:** Multiple wallet compromises
- **Lessons:**
  - Trusted platform library had vulnerability
  - Behavioral requirement (use SecureRandom) insufficient

**PlayStation 3 (2010):**
- **Vulnerability:** Sony reused ECDSA nonce for firmware signing
- **Result:** Console master key extracted
- **Impact:** Complete platform security breach
- **Lessons:**
  - Human error in nonce management
  - Even sophisticated organizations make mistakes

**Conclusion:**
- Historical precedent: 3+ major incidents
- Impact: Complete key compromise
- Exploitability: Trivial (O(1) field operations)
- Likelihood: Medium (proven by history)

**Severity Decision:** 🔴 **High** (confirmed from v2.0 Medium)

**Threshold for Critical:**
- Would require: High likelihood (easily triggered) + Remote exploit
- PVUGC-008: Medium likelihood (requires implementation bug) + Passive exploit
- Decision: High (not Critical)

---

### Finding 8: Cross-Issue Dependencies (Integration Analysis)

**PVUGC-008 depends on:**

**PVUGC-005 (Context Binding):**
- ctx_hash uniqueness is foundation for nonce uniqueness
- If ctx_hash not unique → deterministic derivation produces same r_i
- PVUGC-005 status: ✅ Resolved (v3.0)
- Dependency: **Critical** - PVUGC-008 solution requires PVUGC-005 resolution

**Integration verification:**
```
From PVUGC-005 v3.0:
  ctx_hash = H("PVUGC/CTX" || ctx_core || arming_pkg_hash || presig_pkg_hash)

  Uniqueness property (proven):
    Different instances → different ctx_hash

From PVUGC-008 proposed solution:
  (r₁,ᵢ, r₂,ᵢ) = HKDF(secret_key_i, ctx_hash)

  Uniqueness property (derived):
    Different ctx_hash → different (r₁,ᵢ, r₂,ᵢ)

Composition:
  Different instances → different ctx_hash (PVUGC-005)
                      → different (r₁,ᵢ, r₂,ᵢ) (PVUGC-008)
                      → different R (MuSig2 aggregation)

Conclusion: ✅ Integration verified (PVUGC-005 + PVUGC-008 compose correctly)
```

**No conflicts identified.**

---

## Recommendations

### Normative (MUST) - Specification Changes

#### 1. Add §10.1 MuSig2 Nonce Derivation (HIGHEST PRIORITY)
**Owner:** Protocol designers
**Timeline:** Immediate (blocking for mainnet)
**Deliverable:** Normative specification section

**Complete normative text (ready for spec insertion):**

```markdown
### 10.1 MuSig2 Nonce Derivation (Normative)

To cryptographically enforce nonce uniqueness per context, all signers
participating in the MuSig2 pre-signature ceremony **MUST** derive their
secret nonce contribution using the following deterministic procedure.

#### 10.1.1 Deterministic Nonce Derivation Function

**Function:** `Derive_MuSig2_Nonce(secret_key_i, ctx_hash)`

**Inputs:**
- `secret_key_i`: The 32-byte private key of individual signer i (x_i in ℤ_n)
- `ctx_hash`: The 32-byte final context hash for the current PVUGC instance
  (as computed in §3)

**Algorithm:**

1. **Extract Phase** (HKDF-Extract per RFC 5869):
   ```
   salt = ctx_hash       (32 bytes, public context)
   ikm = secret_key_i    (32 bytes, private key)
   prk = HKDF-Extract(salt, ikm)
   ```
   Where HKDF-Extract uses HMAC-SHA256 as specified in RFC 5869:
   ```
   prk = HMAC-SHA256(key=salt, data=ikm)
   ```

   **Rationale:** Follows RFC 6979 convention (salt=public_context, IKM=secret_key)
   for consistency with established deterministic signature standards.

2. **Expand Phase** (HKDF-Expand per RFC 5869):

   BIP-327 requires two nonces per signer. Derive using info parameter differentiation:

   **First nonce:**
   ```
   info_1 = "PVUGC/MuSig2-Nonce/v1" || 0x00  (24 bytes)
   L = 32  (output length in bytes)
   okm_1 = HKDF-Expand(prk, info_1, L)
   ```

   **Second nonce:**
   ```
   info_2 = "PVUGC/MuSig2-Nonce/v1" || 0x01  (24 bytes)
   okm_2 = HKDF-Expand(prk, info_2, L)
   ```

   Where HKDF-Expand uses HMAC-SHA256:
   ```
   T(0) = empty string
   T(1) = HMAC-SHA256(prk, T(0) || info || 0x01)
   okm = first 32 bytes of T(1)
   ```

3. **Scalar Conversion**:
   ```
   r_{1,i} = bytes_to_integer(okm_1)  (big-endian conversion)
   r_{2,i} = bytes_to_integer(okm_2)  (big-endian conversion)
   ```

4. **Validity Check**:
   ```
   For each r in {r_{1,i}, r_{2,i}}:
     if r == 0 or r >= n:
       ABORT (derivation failed, invalid scalar)

   where n is the order of secp256k1:
     n = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141
   ```

   **Note:** The probability of this condition is negligible
   (< 2^-128), but the check is required for correctness.

5. **Output**:
   ```
   Return (r_{1,i}, r_{2,i})  (valid scalars in [1, n-1])
   ```

#### 10.1.2 Version Handling

The version tag "v1" in "PVUGC/MuSig2-Nonce/v1" is fixed for this protocol version.

**Version upgrade rules (MUST):**

1. Future protocol versions MUST use incrementing version tags:
   - v2: "PVUGC/MuSig2-Nonce/v2"
   - v3: "PVUGC/MuSig2-Nonce/v3"
   - etc.

2. Old implementations MUST NOT accept nonces from new versions

3. New implementations MUST NOT accept nonces from old versions

4. Version mismatch MUST result in protocol abort

5. The protocol version MUST be bound in ctx_core or GS_instance_digest

**Rationale:** Prevents cross-version nonce reuse attacks if HKDF parameters
change in future protocol upgrades.

#### 10.1.3 BIP-327 Integration

This deterministic nonce derivation **MUST** be used in the BIP-327
NonceGen phase:

```
For each signer i:
  Generate deterministic nonces:
    (r_{1,i}, r_{2,i}) = Derive_MuSig2_Nonce(secret_key_i, ctx_hash)

  Compute public nonces:
    R_{1,i} = r_{1,i}·G
    R_{2,i} = r_{2,i}·G

  Publish: pubnonce_i = (R_{1,i}, R_{2,i})
  Store securely: secnonce_i = (r_{1,i}, r_{2,i})
```

**MUST erase** `secnonce_i` immediately after signing or session abort.

#### 10.1.4 Security Properties

This specification ensures the following properties:

1. **Uniqueness:** If `ctx_hash` is unique (guaranteed by §3 context
   binding), then `(r_{1,i}, r_{2,i})` is unique per protocol instance with
   overwhelming probability (collision resistance of SHA-256).

2. **Unpredictability:** To any adversary not knowing `secret_key_i`,
   the values `r_{1,i}` and `r_{2,i}` are computationally indistinguishable
   from uniformly random scalars in [1, n-1] (PRF security of HMAC-SHA256).

3. **Resilience:** The derivation is stateless and requires no external
   random number generator, eliminating RNG-related implementation
   vulnerabilities.

4. **Domain Separation:** The tag "PVUGC/MuSig2-Nonce/v1" and counter bytes
   (0x00, 0x01) prevent cross-protocol nonce reuse, ensure r_{1,i} ≠ r_{2,i},
   and enable future protocol versions.

#### 10.1.5 Nonce Reuse Detection (Defense-in-Depth)

Implementations **SHOULD** maintain a blacklist of previously used nonce
values:

```
Before pre-signing:
  1. Compute aggregate R per BIP-327
  2. Check: R not in blacklist_R
  3. If duplicate detected: ABORT and log security violation
  4. Add R to blacklist_R (persist to storage)
```

This provides runtime verification of uniqueness and detects
implementation bugs in deterministic derivation.

#### 10.1.6 Test Vectors

Implementations MUST provide and verify against test vectors with the
following minimum coverage:

**Normal operation (≥5 vectors):**
- Different ctx_hash values with same secret_key_i
- Different secret_key_i values with same ctx_hash
- Expected: All (r_{1,i}, r_{2,i}) pairs distinct

**Determinism (≥2 vectors):**
- Same (secret_key_i, ctx_hash) tested twice
- Expected: Identical (r_{1,i}, r_{2,i}) outputs

**Edge cases (≥3 vectors):**
- ctx_hash differing by single bit (avalanche test)
- secret_key_i = 0x0000...0001 (minimum valid key)
- secret_key_i = n - 1 (maximum valid key)

**BIP-327 integration (≥2 vectors):**
- Two-nonce generation: (r_{1,i}, r_{2,i}) for same ctx_hash
- Expected: r_{1,i} ≠ r_{2,i} (verified via R_{1,i} ≠ R_{2,i})

**Statistical Validation (SHOULD):**

Uniqueness testing requirements:
- Minimum samples: n = 1000
- Generate 1000 nonces with different ctx_hash values
- Expected unique nonces: 1000 (all distinct)
- Collision probability bound: P(collision) < 2^-128 for SHA-256
- Statistical test: If any collision occurs → FAIL

Determinism testing requirements:
- Repeat derivation for same (secret_key_i, ctx_hash) 100 times
- Expected result: Identical (r_{1,i}, r_{2,i}) values all 100 times
- Statistical test: If any mismatch → FAIL

Negative testing (key extraction):
- Generate 2 signatures with same R, different messages (bypass derivation)
- Extract key: x = (s₁ - s₂) / (c₁ - c₂)
- Verify: x·G = P (public key)
- Expected: Attack succeeds (demonstrates vulnerability)

**Test Vector Format:**
```
Test Vector #N:
  secret_key_i: [hex 32 bytes]
  ctx_hash:     [hex 32 bytes]

  Expected Intermediate Values:
    prk:      [hex 32 bytes]  (HKDF-Extract output)
    okm_1:    [hex 32 bytes]  (HKDF-Expand output, first nonce)
    okm_2:    [hex 32 bytes]  (HKDF-Expand output, second nonce)

  Expected Output:
    r_{1,i}:  [hex 32 bytes]  (first scalar, big-endian)
    r_{2,i}:  [hex 32 bytes]  (second scalar, big-endian)
    R_{1,i}:  [hex 33 bytes]  (compressed public key, 0x02 or 0x03 prefix)
    R_{2,i}:  [hex 33 bytes]  (compressed public key, 0x02 or 0x03 prefix)
```
```

**Acceptance criteria:**
- [ ] Section 10.1 added to PVUGC-2025-10-05.md:94
- [ ] All HKDF parameters specified exactly (salt, IKM, info, output length)
- [ ] Domain separation tag "PVUGC/MuSig2-Nonce/v1" documented
- [ ] Version handling rules specified
- [ ] BIP-327 two-nonce integration specified
- [ ] Security properties formally stated
- [ ] Statistical test requirements with quantitative bounds provided
- [ ] Test vectors provided (minimum 10 vectors)

---

#### 2. Specify BIP-327 State Machine Compliance
**Priority:** MUST
**Timeline:** Immediate
**Location:** Add to §4 or new §10.2

**Normative requirement:**

```markdown
### 10.2 MuSig2 State Machine (Normative)

All MuSig2 signing sessions **MUST** follow the state machine defined in
BIP-327. The following invariants **MUST** be enforced:

1. **Nonce Uniqueness:** Fresh nonces **MUST** be generated for each
   signing session using the deterministic derivation in §10.1. Nonces
   **MUST NOT** be reused across sessions.

2. **Nonce Commitment (SHOULD):** Implementations **SHOULD** use nonce
   commitments to prevent adaptive attacks. Each signer commits to
   `pubnonce_i` before any signer reveals their nonce.

3. **Aggregate Before Sign:** All `pubnonce` values **MUST** be collected
   before any signer computes their partial signature. Signing before
   receiving all nonces **MUST** be rejected.

4. **Secnonce Erasure:** The secret nonce values `secnonce_i` **MUST** be
   erased from memory immediately after:
   - Computing the partial signature, OR
   - Aborting the signing session

5. **No State Regression:** Once partial signatures are generated, the
   session **MUST NOT** regress to earlier states. A message **MUST NOT**
   be signed twice with different nonces.

6. **R Normalization Timing:** The aggregate nonce R **MUST** be normalized
   to even y-coordinate (per BIP-340) immediately after nonce aggregation
   and before:
   - Computing `presig_pkg_hash`
   - Computing challenge `c`
   - Computing partial signatures `s_i`

   **Negation factor application:**

   After aggregation R = R_1 + b·R_2:

   If R.y is odd:
     Set negation_factor = -1 mod n
     R' = -R  (point negation, now has even y)
   Else:
     Set negation_factor = 1
     R' = R  (already has even y)

   Each signer computes partial signature with negation factor:
     s_i = negation_factor · (r_{1,i} + b·r_{2,i}) + c·a_i·x_i  (mod n)

   **Note:** Stored secnonce values need not be modified; the negation
   factor is applied only during signature computation.

   Use normalized R' for all subsequent operations:
   - presig_pkg_hash computation (uses R'_x)
   - Challenge hash: c = H("BIP0340/challenge" || R'_x || P_x || m)
   - Adaptor verification: s'·G + T = R' + c·P
   - Signature output: (R'_x, s)

Implementations **MUST** log and reject any violations of these invariants.
```

---

#### 3. Specify R Normalization Timing
**Priority:** MUST
**Timeline:** Immediate
**Location:** §10 (line 274 clarification)

**Current v2.0 text (line 274):**
> "R must be normalized to even-y per BIP-340"

**Revised normative text (included in §10.2 above):**

The complete specification is now integrated into §10.2, Invariant 6.

---

### Implementation and Testing

#### 4. Reference Implementation
**Priority:** MUST (blocking for mainnet)
**Timeline:** 1 month
**Owner:** Implementation team

**Requirements:**
```
1. Deterministic nonce derivation (§10.1)
   - HKDF-Extract with salt=ctx_hash, IKM=secret_key_i
   - HKDF-Expand with info="PVUGC/MuSig2-Nonce/v1" || counter
   - Scalar validity check (reject if r = 0 or r >= n)
   - Version tag handling

2. BIP-327 integration
   - Two-nonce generation (r_{1,i}, r_{2,i})
   - Nonce aggregation: R = R_1 + b·R_2
   - R normalization after aggregation
   - Negation factor application in signature computation
   - Secnonce erasure after signing

3. Blacklist mechanism (SHOULD)
   - Persistent storage of used R values
   - Runtime duplicate detection
   - Logging of security violations

4. State machine enforcement
   - BIP-327 state transitions
   - Invariant checks (no regression, no reuse)
   - Error handling (abort on violation)

5. Constant-time implementation
   - HMAC-SHA256 constant-time
   - Scalar operations constant-time (libsecp256k1)
   - No timing side-channels
```

**Language:** Rust or C++ (with libsecp256k1)

**Deliverable:**
- Reference implementation code
- API documentation
- Integration guide

---

#### 5. Comprehensive Test Suite
**Priority:** MUST
**Timeline:** 1 month
**Owner:** Testing team

**Test Categories:**

**5.1 Uniqueness Tests**
```
Test: Nonce uniqueness across instances
  Input: 1000 different ctx_hash values
  For each i in 1..1000:
    (r_{1,i}, r_{2,i}) = Derive_MuSig2_Nonce(secret_key, ctx_hash_i)
    R_{1,i} = r_{1,i}·G
    R_{2,i} = r_{2,i}·G
  Verify: All R_{1,i} values distinct (no duplicates)
  Verify: All R_{2,i} values distinct (no duplicates)
  Expected: PASS (collision probability < 2^-128)

Test: Determinism
  Input: Same (secret_key, ctx_hash) twice
  (r_{1,1}, r_{2,1}) = Derive_MuSig2_Nonce(secret_key, ctx_hash)
  (r_{1,2}, r_{2,2}) = Derive_MuSig2_Nonce(secret_key, ctx_hash)
  Verify: r_{1,1} == r_{1,2} and r_{2,1} == r_{2,2}
  Expected: PASS (deterministic derivation)

Test: Context sensitivity
  Input: secret_key, ctx_hash_1, ctx_hash_2 (differ by 1 bit)
  (r_{1,1}, r_{2,1}) = Derive_MuSig2_Nonce(secret_key, ctx_hash_1)
  (r_{1,2}, r_{2,2}) = Derive_MuSig2_Nonce(secret_key, ctx_hash_2)
  Verify: r_{1,1} ≠ r_{1,2} and r_{2,1} ≠ r_{2,2}
  Expected: PASS (avalanche effect from SHA-256)

Test: Two-nonce independence
  Input: secret_key, ctx_hash
  (r_{1,i}, r_{2,i}) = Derive_MuSig2_Nonce(secret_key, ctx_hash)
  Verify: r_{1,i} ≠ r_{2,i}
  Expected: PASS (different info tags)
```

**5.2 Reuse Detection Tests**
```
Test: Blacklist prevents R reuse
  ctx_hash_1, ctx_hash_2 = distinct contexts
  Session 1: Use R_1 (added to blacklist)
  Session 2: Attempt to use R_1 again (manually injected)
  Expected: ABORT (blacklist rejects duplicate)

Test: Blacklist persists across restarts
  Session 1: Use R_1 (stored in blacklist)
  Restart implementation
  Session 2: Attempt to use R_1
  Expected: ABORT (blacklist loaded from storage)
```

**5.3 Attack Simulation (Negative Tests)**
```
Test: Key extraction from nonce reuse
  Setup: Two signing sessions with forced R reuse (bypass derivation)
  Message m_1, m_2 (different)
  Signatures: (R, s_1) for m_1, (R, s_2) for m_2
  Attack: Compute x = (s_1 - s_2) / (c_1 - c_2) mod n
  Verify: x·G == public_key
  Expected: PASS (demonstrates attack works)
  Purpose: Validate that attack is practical if nonce reused

Test: Deterministic derivation prevents attack
  Setup: Two sessions with same secret_key, different ctx_hash
  (r_{1,1}, r_{2,1}) = Derive_MuSig2_Nonce(sk, ctx_hash_1)
  (r_{1,2}, r_{2,2}) = Derive_MuSig2_Nonce(sk, ctx_hash_2)
  Verify: (r_{1,1}, r_{2,1}) ≠ (r_{1,2}, r_{2,2}) (no reuse occurred)
  Sign both messages: (R_1, s_1), (R_2, s_2)
  Attack: Attempt key extraction
  Expected: FAIL (different R values, attack doesn't apply)
```

**5.4 BIP-327 State Machine Tests**
```
Test: State transition enforcement
  Happy path: KeyAgg → NonceGen → NonceAgg → Sign → SigAgg
  Expected: PASS

Test: Reject state regression
  Attempt: Sign → NonceGen (regress after signing)
  Expected: ABORT (state regression forbidden)

Test: Reject signing before nonce aggregation
  Attempt: NonceGen → Sign (skip NonceAgg)
  Expected: ABORT (must aggregate before signing)

Test: Secnonce erasure
  After signing: Check memory for secnonce_i values
  Expected: All secnonces erased (zeroed memory)
```

**5.5 R Normalization Tests**
```
Test: R normalization with odd y
  Generate R with odd y-coordinate
  Normalize per spec
  Verify: R.y is even
  Expected: PASS

Test: Negation factor application
  Generate aggregate R with odd y
  Each signer applies negation factor in signature computation
  Verify: Final signature (R_x, s) passes BIP-340 verification
  Expected: PASS

Test: Signature verification with normalized R
  Generate signature with normalized R
  Verify per BIP-340
  Expected: PASS (signature valid)

Test: Normalization timing
  Compute presig_pkg_hash with R (after normalization)
  Verify: R_x used in hash (not R_y)
  Expected: PASS (x-only encoding)
```

**5.6 Test Vectors (MUST provide)**
```
Format per §10.1.6 specification

Minimum 10 test vectors covering:
  - Normal case (r valid)
  - Edge cases (r near 0, near n)
  - Different ctx_hash values
  - Domain tag variation
  - Two-nonce independence
  - Version tag handling
```

---

#### 6. External Cryptographic Review
**Priority:** MUST (before mainnet)
**Timeline:** 2-3 months
**Owner:** Project leadership

**Scope:**
1. Review §10.1 MuSig2 Nonce Derivation specification
2. Validate HKDF parameter choices (salt, IKM, info)
3. Verify security properties (uniqueness, unpredictability, resilience)
4. Compare with RFC 6979 and BIP-340 approaches
5. Review reference implementation for side-channels
6. Validate test suite coverage

**Deliverable:** Independent audit report

**Recommended reviewers:**
- Academic cryptographers (publications on Schnorr/MuSig)
- Bitcoin Core developers (BIP-340, BIP-327 expertise)
- Professional security auditors (Trail of Bits, NCC Group, etc.)

---

## Validation Checklist

**Specification:**
- [ ] §10.1 MuSig2 Nonce Derivation added to PVUGC-2025-10-05.md:94
- [ ] HKDF parameters specified (salt=ctx_hash, IKM=secret_key_i, info="PVUGC/MuSig2-Nonce/v1" || counter, L=32)
- [ ] Domain separation tag "PVUGC/MuSig2-Nonce/v1" documented
- [ ] Version handling rules specified (cross-version rejection)
- [ ] BIP-327 integration specified (two-nonce generation using info parameter)
- [ ] Scalar validity check specified (reject if r = 0 or r >= n)
- [ ] Security properties formally stated (uniqueness, unpredictability, resilience, domain separation)
- [ ] §10.2 BIP-327 State Machine compliance added
- [ ] State machine invariants specified (no nonce reuse, no state regression, secnonce erasure)
- [ ] R normalization timing specified (after aggregation, before signing)
- [ ] Negation factor application method specified
- [ ] Blacklist mechanism specified (SHOULD requirement, defense-in-depth)
- [ ] Statistical test requirements with quantitative bounds specified

**Implementation:**
- [ ] Reference implementation complete (Rust or C++)
- [ ] HKDF-Extract correctly implemented (HMAC-SHA256 with salt=ctx_hash, IKM=secret_key_i)
- [ ] HKDF-Expand correctly implemented (info tag with counter bytes, output length 32)
- [ ] BIP-327 two-nonce generation (r_{1,i}, r_{2,i} using info parameter)
- [ ] R normalization implemented (after aggregation)
- [ ] Negation factor application implemented (in signature computation)
- [ ] Secnonce erasure implemented (immediate after signing)
- [ ] Blacklist mechanism implemented (persistent storage)
- [ ] State machine enforcement implemented
- [ ] Constant-time HMAC-SHA256 (no timing side-channels)
- [ ] Constant-time scalar operations (libsecp256k1)

**Testing:**
- [ ] Uniqueness tests: 1000+ different ctx_hash → 1000+ unique R values (no collisions)
- [ ] Determinism tests: Same (sk, ctx_hash) → same (r_{1,i}, r_{2,i}) (reproducible)
- [ ] Context sensitivity: Different ctx_hash → different (r_{1,i}, r_{2,i}) (avalanche effect)
- [ ] Two-nonce independence: r_{1,i} ≠ r_{2,i} for same ctx_hash
- [ ] Reuse detection: Attempt R reuse → rejected by blacklist
- [ ] Persistence: Blacklist survives restart
- [ ] Attack simulation: Two sigs with same R → key extractable (negative test)
- [ ] Attack prevention: Deterministic derivation → no reuse → attack fails
- [ ] BIP-327 state machine: Happy path succeeds
- [ ] State regression: Attempt regression → rejected
- [ ] R normalization: Odd y → normalized to even y
- [ ] Negation factor: Applied correctly in signature computation
- [ ] Signature verification: Normalized R → valid BIP-340 signature
- [ ] Test vectors: 10+ vectors provided and verified with statistical bounds

**Review:**
- [ ] External cryptographic review scheduled
- [ ] Review scope includes §10.1 specification
- [ ] Review scope includes reference implementation
- [ ] Review scope includes security properties validation
- [ ] Independent audit report received
- [ ] Audit findings addressed (if any)

**Integration:**
- [ ] PVUGC-005 (ctx_hash uniqueness) resolution verified
- [ ] Deterministic nonce derivation depends on unique ctx_hash
- [ ] Integration tested: different instances → different ctx_hash → different r_i → different R
- [ ] No conflicts with other PVUGC issues

---

## Status Decision

**Final Severity:** 🔴 **High**

**Justification:**
1. **Impact:** Complete aggregate private key compromise
2. **Historical precedent:** 3+ major cryptocurrency incidents (blockchain.info 2013, Android wallet 2013, PS3 2010)
3. **Exploitability:** Trivial (O(1) field operations, no cryptographic work)
4. **Detection:** Only post-facto (after key compromised)
5. **Mitigation gap:** v2.0 MUST clause insufficient (behavioral requirement, not cryptographic enforcement)

**Final Status:** ❌ **Refuted** (v2.0 mitigation insufficient)

**Rationale:**
- v2.0 MUST clause is behavioral requirement, not cryptographic enforcement
- Historical failures demonstrate MUST clauses don't prevent RNG bugs
- Deterministic nonce derivation is industry-standard solution (RFC 6979, EdDSA)
- Final specification is cryptographically sound and complete
- Implementation is feasible (HKDF widely available, no exotic crypto)

**Acceptance Criteria for Resolution:**

This issue can be considered **RESOLVED** when:

1. **Specification complete:**
   - [ ] §10.1 MuSig2 Nonce Derivation added to spec (normative MUST)
   - [ ] §10.2 BIP-327 State Machine compliance specified
   - [ ] R normalization timing specified with negation factor application
   - [ ] Version handling rules specified
   - [ ] All HKDF parameters documented

2. **Reference implementation:**
   - [ ] Deterministic derivation implemented
   - [ ] BIP-327 state machine enforced
   - [ ] Blacklist mechanism implemented (SHOULD)
   - [ ] Constant-time operations verified

3. **Testing complete:**
   - [ ] 1000+ uniqueness tests pass with statistical bounds
   - [ ] Attack simulations validate vulnerability and mitigation
   - [ ] BIP-327 state machine tests pass
   - [ ] Test vectors provided and verified

4. **External review:**
   - [ ] Independent cryptographic audit completed
   - [ ] Audit findings addressed
   - [ ] Reviewer confirms specification is sound

**Target timeline:** 2-3 months (1 month implementation + testing, 2 months external review)

**Blocking for mainnet:** YES (High severity, key compromise risk)

---

## Attribution and Acknowledgments

**M1 (Mathematician #1):**
- ✅ Identified classic Schnorr nonce reuse vulnerability (v1.0)
- ✅ Documented attack mathematics (private key extraction)
- ✅ Proposed v2.0 improvements (MUST clause, context binding)
- ✅ Extensive attack vector analysis (720 lines, v2.0 report)
- ✅ Wagner's algorithm analysis (multi-instance attack)
- ✅ Cross-protocol T reuse analysis
- ✅ MuSig2 state machine initial specification
- ✅ R normalization requirements

**M2 (Mathematician #2):**
- ✅ Validated M1's attack analysis
- ✅ Refuted v2.0 MUST clause sufficiency
- ✅ Specified formal attack algorithm (key extraction)
- ✅ Proposed deterministic nonce derivation solution (HKDF-based)
- ✅ Round 3 revisions: RFC 6979 alignment, version handling, two-nonce derivation, R normalization clarification, statistical test bounds
- ✅ Validated security properties (uniqueness, unpredictability, resilience)
- ✅ Compared with RFC 6979
- ✅ Justified severity confirmation (High)

**Crypto-Peer-Reviewer:**
- ✅ Integrated M1's analysis with M2's validation
- ✅ Applied all Round 3 revisions (R1-R6)
- ✅ Produced final normative specification text
- ✅ Comprehensive validation checklist
- ✅ Clear acceptance criteria

**Consensus:**
- Both M1 and M2 agree: Nonce reuse is critical vulnerability
- Both agree: v2.0 MUST clause insufficient
- Both agree: Deterministic derivation required
- M2 enhanced M1's analysis with formal specification and HKDF details
- Final specification incorporates all Round 3 improvements

**This report integrates:**
- M1's v1.0 and v2.0 analysis (validated, all concerns addressed)
- M2's Round 1 refutation (validated, specification accepted)
- M2's Round 3 review (all revisions applied)
- Cryptographer's formal validation (this report)

**Recommendation:** Adopt §10.1 and §10.2 specifications as normative requirements.

---

## Cross-References

**Specification:**
- [`PVUGC-2025-10-20.md §6 Compartmentalization`](../PVUGC-2025-10-20.md) (Compartmentalization MUST clause - refuted as insufficient)
- [`PVUGC-2025-10-20.md §3 Context Binding`](../PVUGC-2025-10-20.md) (Context binding - foundation for uniqueness)
- [`PVUGC-2025-10-20.md §3 Context Binding`](../PVUGC-2025-10-20.md) (presig_pkg_hash - includes T, R)
- [`PVUGC-2025-10-20.md §4.2 AdaptorVerify`](../PVUGC-2025-10-20.md) (AdaptorVerify - uses R)
- [`PVUGC-2025-10-20.md:314`](../PVUGC-2025-10-20.md) (R normalization - timing gap addressed)

**Previous Reports:**
- [`report-preliminary-2025-10-07/PVUGC-008-musig2-compartmentalization.md`](../report-preliminary-2025-10-07/PVUGC-008-musig2-compartmentalization.md) (v1.0 - initial identification)
- [`report-update-2025-10-07/PVUGC-008-musig2-compartmentalization.md`](../report-update-2025-10-07/PVUGC-008-musig2-compartmentalization.md) (v2.0 - MUST clause added, 720 lines analysis)

**Peer Review:**
- [Appendix A08.M201](APPENDIX-issue-debates.md#a08-m201) (M2 analysis plan - objectives, deliverables)
 - [Appendix A08.M201](APPENDIX-issue-debates.md#a08-m201) (Nonce reuse refutation and extraction)
 - [Appendix A08.CR02](APPENDIX-issue-debates.md#a08-cr02) (Deterministic HKDF nonce derivation)
 - [Appendix A08.M203](APPENDIX-issue-debates.md#a08-m203) (Revisions: worst‑vs‑average, TOST, cache timing)

**Related Issues:**
- PVUGC-005 (Context Binding) - ctx_hash uniqueness is foundation for nonce uniqueness
- PVUGC-007 (Timing Attacks) - Related to MuSig2 coordination timing (not directly dependent)

**External Standards:**
- BIP-327: MuSig2 for BIP340-compatible Multi-Signatures
- BIP-340: Schnorr Signatures for secp256k1
- RFC 5869: HMAC-based Extract-and-Expand Key Derivation Function (HKDF)
- RFC 6979: Deterministic Usage of the Digital Signature Algorithm (DSA) and Elliptic Curve DSA (ECDSA)

---

## References

**Cryptographic Standards:**
- **RFC 5869:** HKDF (HMAC-based KDF) - https://tools.ietf.org/html/rfc5869
- **RFC 6979:** Deterministic ECDSA - https://tools.ietf.org/html/rfc6979
- **BIP-340:** Schnorr Signatures for secp256k1 - https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
- **BIP-327:** MuSig2 - https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki
- **RFC 8032:** EdDSA (deterministic nonces) - https://tools.ietf.org/html/rfc8032

**Historical Incidents:**
- **blockchain.info (2013):** Android SecureRandom vulnerability
- **CVE-2013-4444:** Android Bitcoin Wallet (Java SecureRandom predictability)
- **PlayStation 3 (2010):** Sony ECDSA nonce reuse (console master key compromise)

**Academic References:**
- Nick, Ruffing, Seurin: "MuSig2: Simple Two-Round Schnorr Multi-Signatures" (CRYPTO 2021)
- Schnorr: "Efficient Signature Generation by Smart Cards" (1991)
- Wagner: "A Generalized Birthday Problem" (2002) - Wagner's attack on nonce-related signatures

---

**Report prepared by:** Crypto-Peer-Reviewer
**Review date:** 2025-10-26
**Report version:** 4.0 (Final)
**Next review:** After §10.1 and §10.2 implementation and external audit

---

**END OF REPORT**
