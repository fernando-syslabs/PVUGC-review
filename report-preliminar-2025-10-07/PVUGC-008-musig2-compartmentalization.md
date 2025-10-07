# PVUGC-008: MuSig2 - Adaptor Compartmentalization Enforcement

**Flaw Code:** PVUGC-008
**Severity:** 🟡 **MEDIUM**
**Status:** 🔓 Open
**Date Identified:** 2025-10-07

---

## Component
MuSig2 pre-signature and adaptor construction

## Location
- **Section:** §4 (Keys & one-time pre-sign), §10 (Adaptor verification)
- **Line:** 90 (Compartmentalization MUST), 241 (AdaptorVerify)

---

## Description

Protocol requires (line 90):
> **Compartmentalization (MUST):** one adaptor ⇒ one unique T and one unique R. Never reuse across paths/epochs or templates.

**The gap:** Uniqueness is a "MUST" requirement but no cryptographic enforcement. Relies on honest implementation. Reusing R across two messages enables classic Schnorr nonce-reuse attack, extracting private key.

---

## Specific Issues

### 1. No Cryptographic Enforcement
- Protocol doesn't prove T, R are fresh
- No blacklist mechanism for previously used nonces
- Relies on implementation discipline

### 2. Nonce Reuse Attack
```
Two adaptors with same R:
s'₁·G + T = R + e₁·P  (message m₁)
s'₂·G + T = R + e₂·P  (message m₂)

After finalization (s₁ = s'₁ + α, s₂ = s'₂ + α):
s₁·G = R + e₁·P
s₂·G = R + e₂·P

Subtraction:
(s₁ - s₂)·G = (e₁ - e₂)·P
⟹ P = [(s₁ - s₂)/(e₁ - e₂)]·G

Private key extracted: x = (s₁ - s₂)/(e₁ - e₂)
```

### 3. MuSig2 Coefficient Binding
Line 252: presig_pkg_hash binds key list L and coefficients aᵢ

**Question:** Who verifies coefficients computed correctly?
- Incorrect coefficients: wrong P = Σᵢ aᵢ·Xᵢ
- Signature verifies against wrong key → spend fails (liveness)

### 4. R Normalization
R must be normalized to even-y (BIP-340). If implementations differ on when normalization happens:
- Pre-sign with R_even
- Verification uses R_odd
- Signature fails

---

## Attack Vectors

### Attack 8.1: Nonce Reuse Key Extraction
```
1. Buggy/malicious signer reuses R for two transactions
2. Both use same T (same arming round)
3. Pre-signatures: s'₁, s'₂ with same R, different m₁, m₂
4. After decryption: s₁ = s'₁ + α, s₂ = s'₂ + α
5. Both signatures broadcast
6. Attacker observes (R, s₁, m₁, P) and (R, s₂, m₂, P)
7. Extracts: x = (s₁ - s₂)/(e₁ - e₂)
8. Recovers aggregate private key
9. Can spend from any address using P
```

### Attack 8.2: Wagner's Algorithm (Many Reused Nonces)
```
If protocol used across k contexts with reused R:
1. Attacker collects k adaptors with same R
2. Applies Wagner's generalized birthday attack
3. Finds subset where Σ eᵢ = 0
4. Solves for private key with reduced complexity
5. Practical for k ≥ 20-30
```

---

## Recommendations

### Specification

#### 1. Deterministic T, R Derivation
**Priority:** P2, **Timeline:** 2 weeks

Make nonces deterministic from ctx_hash:
```
R_seed = HKDF(secret_key_material, ctx_hash || "PVUGC/NONCE" || counter)
R = R_seed · G  (normalize to even-y)

T_seed = HKDF(secret_key_material, ctx_hash || "PVUGC/ADAPTOR")
T = T_seed · G
```

Benefits:
- Ensures T, R unique per context (cryptographically)
- Prevents accidental reuse
- Deterministic → reproducible, testable

#### 2. Nonce Blacklist Mechanism
**Priority:** P2, **Timeline:** 2 weeks

Maintain persistent set of used (R, T) pairs:
```
On pre-sign:
1. Check R ∉ previous_R_set
2. Check T ∉ previous_T_set
3. Add (R, T) to sets
4. Abort if duplicate detected
```

Requires:
- Persistent storage
- Synchronized across signers (in MuSig2 context)

#### 3. MuSig2 State Machine Enforcement
**Priority:** P3, **Timeline:** 1 week

Specify exact BIP-327 state transitions:
```
1. Nonce commitments (all signers)
2. Nonce reveals (verify commitments)
3. Nonce aggregation: R = Σᵢ Rᵢ (normalize to even-y)
4. Partial signature generation
5. Signature aggregation
```

Add verification:
```
☐ All nonce commitments received before reveals
☐ R normalized immediately after aggregation
☐ No state machine violations (logged and rejected)
```

#### 4. Standardize R Normalization
**Priority:** P2, **Timeline:** 1 week

Update presig_pkg_hash specification:
```
1. Aggregate MuSig2 nonces per BIP-327
2. R_agg = R₁ + R₂ + ... (elliptic curve point addition)
3. Normalize to even-y:
   if R_agg.y is odd:
       R = -R_agg  (negate y-coordinate)
   else:
       R = R_agg
4. R_x = x-coordinate of R (even-y guaranteed)
5. Use R_x in challenge hash: e = H("BIP0340/challenge" || R_x || P_x || m)
```

### Testing

#### 5. Nonce Reuse Detection Tests
**Timeline:** 1 week

- [ ] Attempt R reuse → must be rejected
- [ ] Attempt T reuse → must be rejected
- [ ] Deterministic derivation: same ctx_hash → same (R,T)
- [ ] Different ctx_hash → different (R,T)

#### 6. MuSig2 Interoperability
**Timeline:** 2 weeks

- [ ] Test with multiple MuSig2 implementations (BIP-327 reference, libsecp256k1-zkp)
- [ ] Verify R normalization consistency
- [ ] Verify coefficient computation (aᵢ = KeyAggCoeff(L, Xᵢ))

---

## Acceptance Criteria

- [ ] Deterministic (R,T) derivation implemented OR nonce blacklist enforced
- [ ] MuSig2 state machine specified and enforced
- [ ] R normalization standardized with test vectors
- [ ] Nonce reuse tests pass (reuse rejected)
- [ ] External review validates compartmentalization

---

## Related Flaws
- **PVUGC-007:** Timing attacks (nonce generation must be secure, not just unique)

---

**Last Updated:** 2025-10-07
