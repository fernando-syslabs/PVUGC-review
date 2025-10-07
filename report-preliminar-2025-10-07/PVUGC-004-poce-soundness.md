# PVUGC-004: PoCE-A Soundness - Malicious Armer Detection Gaps

**Flaw Code:** PVUGC-004
**Severity:** 🟠 **HIGH**
**Status:** 🔓 Open
**Date Identified:** 2025-10-07

---

## Component
Proof of Correct Encryption (arm-time verification)

## Location
- **Section:** §5 (Distributed T setup), §8 (PoCE and DEM details)
- **Lines:** 97-106 (PoK/PoCE requirements), 196-223 (PoCE specification)

---

## Description

PoCE (Proof of Correct Encryption) consists of two stages:
- **PoCE-A (arm-time):** NIZK proving knowledge of (ρᵢ, sᵢ) such that D₁,ⱼ = Uⱼ^ρᵢ, D₂,ₖ = Vₖ^ρᵢ, Tᵢ = sᵢ·G, and ρᵢ ≠ 0
- **PoCE-B (decap-time):** Key-commitment check when decrypting (local, not publicly verifiable)

**The gap:** PoCE-A proves mask consistency but doesn't cryptographically bind the ciphertext. PoCE-B only executes at decryption time (after proof generation cost). Malicious armers can pass arm-time checks but publish garbage ciphertexts.

---

## Security Impact

**Griefing/DoS:** Malicious armer locks funds until timeout
**Completeness break:** Valid proofs can't finalize spends if ciphertext is invalid
**Collusion risk:** Multiple malicious armers could zero out α or create other degenerate conditions

---

## Specific Concerns

### 1. Ciphertext Disconnection at Arm-Time

PoCE-A proves:
- D₁,ⱼ, D₂,ₖ correctly formed from same ρᵢ ✓
- Tᵢ = sᵢ·G ✓
- ρᵢ ≠ 0 ✓

But **doesn't prove:**
- ctᵢ actually encrypts sᵢ (checked only at PoCE-B, decap-time)
- ctᵢ uses key derived from ρᵢ

**Attack:** Armer publishes valid masks but encrypts garbage or wrong sᵢ.

### 2. PoCE-B is Decapper-Local (Line 211)

> "PoCE-B is decapper-local (not publicly verifiable unless plaintext revealed)"

**Implications:**
- Honest participants can't detect malicious ciphertexts at arm-time
- Detection requires:
  1. Generating expensive proof (Groth16 + GS)
  2. Attempting decapsulation
  3. PoCE-B check fails
- Cost borne by honest prover, benefit to griefing attacker

### 3. Collusion Scenarios

**Attack: Zero out α**
```
Armers 1 and 2 collude:
- Armer 1: s₁ = r (random)
- Armer 2: s₂ = -r (mod n)
- Both publish valid PoCE-A
- T = T₁ + T₂ = r·G + (-r)·G = O (point at infinity)
```

Line 203 checks "reject if T = O" but only **after** all armers publish. Race condition or missing early check could allow this.

### 4. Special ρᵢ Values

PoCE-A only checks ρᵢ ≠ 0 via auxiliary relation. No bounds:
- **ρᵢ = 1:** M = G_G16^1 = G_G16 (publicly computable, see PVUGC-006)
- **ρᵢ near boundaries:** May have algebraic properties
- **ρᵢ small:** Brute-forceable in some contexts

---

## Attack Vectors

### Attack 4.1: Griefing via Invalid Ciphertext

```
1. Malicious armer generates valid D₁,ⱼ, D₂,ₖ, Tᵢ, PoCE-A
2. Publishes ctᵢ = random_bytes() (garbage, not actual encryption)
3. Arm-time checks pass (PoCE-A verifies)
4. Pre-signature phase completes
5. Prover generates expensive Groth16+GS proof
6. Decapsulation: K derived correctly, but ct decrypts to garbage
7. PoCE-B fails (T_i ≠ s_i·G or h_i mismatch)
8. Spend cannot complete
9. Funds locked until timeout (if timeout path exists)

Impact: DoS, wasted proof generation cost
```

### Attack 4.2: Collusion to Zero α

```
1. Armers 1,2 collude, choose s₁, s₂ = -s₁
2. Both pass PoCE-A
3. If aggregation check (T = Σ Tᵢ ≠ O) missing or has timing gap:
   - T = O slips through
   - α = s₁ + s₂ = 0
4. Adaptor: s = s' + 0 = s' (pre-signature already public)
5. Anyone can finalize without decrypting

Impact: Complete break of gating (no proof needed)
```

### Attack 4.3: ρᵢ = 1 Secret Leakage

```
1. Malicious armer chooses ρᵢ = 1 (satisfies ρᵢ ≠ 0 check)
2. Publishes D₁,ⱼ = Uⱼ, D₂,ₖ = Vₖ (base elements, no exponentiation)
3. M_i = G_G16^1 = G_G16 (publicly computable from vk,x)
4. Anyone derives K_i without proof
5. If all armers do this: α fully leaked

Impact: Break if multiple armers use ρᵢ=1 (honest or malicious)
```

---

## Recommendations

### Immediate Actions

#### 1. Make PoCE-B Publicly Verifiable
**Priority:** P0
**Timeline:** 2-3 weeks

**Option A: Commitment at arm-time**
```
- At arm-time: Publish commitment com_i = Commit(s_i, r_i)
- PoCE-A proves: T_i = s_i·G AND com_i commits to s_i
- At decap-time: Reveal s_i, r_i; verify commitment opens
```

**Option B: Publicly verifiable encryption**
```
- Use El Gamal-style encryption (homomorphic, publicly verifiable)
- ct_i = Enc_PK(s_i) where PK derived from {D₁,ⱼ, D₂,ₖ}
- Anyone with proof can verify ct_i structure without decrypting
```

#### 2. Early Aggregation Check
**Priority:** P0
**Timeline:** 1 week (specification)

Add to protocol:
```
Immediately after all armers publish T_i:
1. Compute T = Σ T_i
2. Verify T ≠ O (point at infinity)
3. Abort if check fails (before pre-signing)
```

#### 3. Restrict ρᵢ Range
**Priority:** P1
**Timeline:** 1 week

Add to PoCE-A:
```
Prove: ρᵢ ∈ [2^128, r - 2^128]
- Prevents ρᵢ = 1 (too small)
- Prevents ρᵢ near r (boundary)
- Range proof techniques: Bulletproofs, zk-SNARKs
```

#### 4. Add Penalty/Bond Mechanism
**Priority:** P1
**Timeline:** 2-3 weeks

Design:
```
- Armers post bond (Bitcoin HTLC or similar)
- If PoCE-B fails at decap: bond slashed
- Requires economic incentive layer (out of protocol scope but recommended)
```

### Specification Improvements

#### 5. Enhanced PoCE-A
**Timeline:** 2 weeks

Update §8 to include:
```markdown
### PoCE-A Enhanced (Normative)

NIZK proving knowledge of (ρᵢ, sᵢ, r_com) such that:
1. D_{1,j} = U_j^{ρᵢ} for all j
2. D_{2,k} = V_k^{ρᵢ} for all k
3. T_i = s_i · G
4. 2^128 ≤ ρᵢ < r - 2^128 (range proof)
5. com_i = Commit(s_i || h_i, r_com) (commitment binding)
6. ct_i structure valid (format check)
```

#### 6. Per-Share Validation Checklist
**Timeline:** 1 week

Add normative checklist:
```
Before accepting armer i's submission:
☐ PoCE-A proof verifies
☐ T_i ≠ O
☐ T_i not previously used
☐ {D₁,ⱼ}, {D₂,ₖ} canonical encodings
☐ ct_i well-formed (length, format)
☐ com_i binding commitment

After all armers submit:
☐ T = Σ T_i ≠ O
☐ No duplicate T_i values
```

### Testing

#### 7. Attack Simulation Suite
**Timeline:** 2 weeks

Tests:
- [ ] Invalid ciphertext (random bytes) → must be detected
- [ ] Wrong sᵢ encrypted → must fail verification
- [ ] Collusion (s₁ + s₂ = 0) → must be rejected at aggregation
- [ ] ρᵢ = 1 → must be rejected (after range proof added)
- [ ] ρᵢ = 0 → must be rejected (existing check)

---

## Acceptance Criteria

- [ ] PoCE-B publicly verifiable OR penalty mechanism implemented
- [ ] Early T ≠ O check before pre-signing
- [ ] ρᵢ range restriction added to PoCE-A
- [ ] Per-share validation implemented
- [ ] Attack simulation tests pass
- [ ] External review validates changes

---

## Related Flaws
- **PVUGC-006:** Degenerate values (ρᵢ=1, T=O edge cases)

---

**Last Updated:** 2025-10-07
