# PVUGC-004: PoCE-B Decapper-Local

**Flaw Code:** PVUGC-004
**Severity:** 🟠 **HIGH**
**Status:** 🔓 Open (Publication SHOULD added)
**Date Identified:** 2025-10-07 (v1.0)
**Last Updated:** 2025-10-07 (v2.0)

---

## Component
Proof of Correct Encryption (PoCE) - Key-commitment verification

## Location
- **v1.0:** §5, §8 (PoCE-A arm-time, PoCE-B decap-time)
- **v2.0:** §5, §8, Line 240 (publication SHOULD clause added)

---

## Description

The protocol uses a two-phase verification system for encrypted arming shares:

**PoCE-A (Arm-time, publicly verifiable):**
- Proves masks {D₁,ⱼ}, {D₂,ₖ} correctly formed
- Proves ρᵢ ≠ 0
- Proves Tᵢ = sᵢ·G (commitment to secret share)
- ✅ Anyone can verify

**PoCE-B (Decap-time, decapper-local):**
- Verifies decrypted (sᵢ, hᵢ) matches commitment Tᵢ
- Checks Tᵢ ?= sᵢ·G and hash consistency
- ❌ Only decapper can verify (requires decryption first)

**Problem:** Malicious armers can publish valid PoCE-A but broken ciphertexts. Only discovered when someone generates proof and attempts decapsulation (expensive operation wasted).

---

## What Changed in v2.0

### 🔧 Partial Improvement

**v2.0 Addition (Line 240):**
> "**SHOULD:** Implementations SHOULD publish a minimal PoCE-B verification transcript (hashes of inputs/outputs) alongside the broadcasted spend to aid auditing."

**What this provides:**
- After successful decapsulation, decapper publishes verification transcript
- Other observers can audit (after-the-fact)
- Malicious armers can be identified post-spend
- Enables reputation/penalty systems (external to protocol)

### 🔓 What's Still Missing

**PoCE-B remains decapper-local:**
- ❌ Cannot verify ciphertext correctness **before** generating proof
- ❌ Cannot detect malicious arming until proof generation (expensive)
- ❌ No on-chain penalty for malicious armers
- ❌ Publication is SHOULD, not MUST (optional)

**Impact:**
- Griefing attacks still possible (force wasted proof generation)
- No prevention mechanism (only detection after-the-fact)
- Economic attacks: Malicious armers have low cost, honest provers have high cost

---

## Security Impact

**Consequence:** Malicious armers can grief honest participants by publishing invalid ciphertexts that pass arm-time checks but fail at decryption.

### Attack Cost vs Honest Cost

| Party | Action | Cost | Outcome |
|-------|--------|------|---------|
| **Malicious Armer** | Publish valid PoCE-A, broken ct | Low (one NIZK) | Griefing succeeds |
| **Honest Prover** | Generate Groth16+GS proof | High (minutes) | Wasted computation |
| **Decapper** | Attempt decapsulation | Medium (pairing ops) | Discovers failure |

**Asymmetry:** Attacker cost << Victim cost (griefing amplification).

### Liveness Impact

**Scenario:** Malicious armer in threshold setting
- Protocol requires k-of-n arming shares
- If 1 malicious armer among n participants
- All shares appear valid (PoCE-A passes)
- Pre-signature created, transaction template locked
- Prover generates expensive proof
- Decapsulation fails on malicious share
- **Result:** Funds locked until timeout, wasted resources

---

## Specific Concerns

### 1. Griefing Vector (Unchanged in v2.0)

**Problem:** PoCE-A proves mask correctness but doesn't bind ciphertext to masks.

**What PoCE-A verifies:**
```
✅ D₁,ⱼ = Uⱼ^ρᵢ for all j
✅ D₂,ₖ = Vₖ^ρᵢ for all k
✅ Tᵢ = sᵢ·G (commitment)
✅ ρᵢ ≠ 0 (via auxiliary relation)
```

**What PoCE-A does NOT verify:**
```
❌ ctᵢ actually encrypts sᵢ
❌ ctᵢ uses correct key Kᵢ = HKDF(ctx_hash, ser(Mᵢ), "PVUGC/KEM/v1")
❌ hᵢ is correctly computed hash of sᵢ
❌ ctᵢ format is valid
```

**Attack:**
```
Malicious armer:
1. Generate valid ρᵢ, sᵢ
2. Compute valid masks: D₁,ⱼ = Uⱼ^ρᵢ, D₂,ₖ = Vₖ^ρᵢ
3. Compute valid commitment: Tᵢ = sᵢ·G
4. Generate valid PoCE-A proof
5. Publish garbage ciphertext: ctᵢ = random_bytes()
   OR ctᵢ encrypts wrong value: Enc_K(wrong_sᵢ || wrong_hᵢ)
6. All arm-time checks pass ✅

Later (after expensive proof generation):
7. Decapper derives Kᵢ from proof, attempts decryption
8. PoCE-B check: Tᵢ ?= sᵢ·G fails (wrong sᵢ or decryption error)
9. Spend aborted, funds locked
```

### 2. Delayed Verification (Unchanged)

**Problem:** Malicious ciphertext only detected after proof generation.

**Timeline:**
```
t=0:  Arming phase
      - Armers publish {Dⱼ, Dₖ, Tᵢ, ctᵢ, PoCE-A}
      - All checks pass ✅
      - No indication of malicious ct

t=1:  Pre-signature phase
      - MuSig2 protocol runs
      - s', R computed and published
      - Transaction template committed

t=2:  Proof generation (EXPENSIVE - minutes to hours)
      - Prover generates Groth16 proof π
      - Prover generates GS attestation
      - Proof published on-chain or to decapper

t=3:  Decapsulation attempt
      - Decapper derives M from proof
      - Computes K = HKDF(ctx_hash, ser(M), "PVUGC/KEM/v1")
      - Attempts Dec_K(ctᵢ) → (sᵢ, hᵢ)
      - PoCE-B check: Tᵢ ?= sᵢ·G
      - ❌ FAILS - malicious ct discovered

Result: Wasted t=2 computation, locked funds
```

**v2.0 SHOULD clause** only helps with post-mortem analysis (not prevention).

### 3. No On-Chain Penalty (v2.0 Unchanged)

**Problem:** Malicious armers have no economic disincentive.

**Current state (v1.0 and v2.0):**
- No bond/deposit required from armers
- No slashing mechanism for invalid ciphertexts
- No on-chain proof of malicious behavior
- Publication SHOULD clause is off-chain (optional)

**Consequence:**
- Rational attacker: Low-cost griefing with no penalty
- No deterrent against repeated attacks
- Reputation systems must be external to protocol

### 4. Cross-Share Collusion (Unchanged)

**Problem:** Multiple malicious armers can coordinate attacks.

**Attack scenarios:**

**Attack 4.1: Threshold bypass attempt**
```
Setup: k-of-n threshold (need k valid shares)
Attack: (k-1) malicious armers + 1 honest armer

Malicious armers:
1. Compute shares such that Σᵢ₌₁ᵏ⁻¹ sᵢ = -s_honest (mod n)
2. Publish valid PoCE-A for malicious shares
3. Honest armer publishes valid share

Aggregation:
- T_agg = Σᵢ Tᵢ = Σᵢ sᵢ·G = 0·G = O (point at infinity)
- Check at line 203: Reject if T = O

Result: Attack caught at aggregation (but after arming)
```

**Note:** v2.0 includes T ≠ O check (line 203 equivalent), mitigates this specific attack.

**Attack 4.2: Griefing amplification**
```
Setup: n armers, m are malicious (m < k)
Attack: All m malicious armers publish broken cts

Impact:
- Every proof attempt will fail (at least one bad ct)
- Maximum griefing: m bad shares means guaranteed failure
- Cost to attacker: m × (PoCE-A cost)
- Cost to victims: Wasted proof generation every attempt

Mitigation: None in v2.0 (publication SHOULD doesn't prevent)
```

### 5. Special ρᵢ or sᵢ Values (Partially Addressed)

**v1.0 gap:** Only checked ρᵢ ≠ 0, no checks on sᵢ or boundary values.

**v2.0 improvements (from PVUGC-006 resolution):**
- ✅ G_G16 ≠ 1 check (line 164)
- ✅ Subgroup membership checks (line 165)
- ✅ T = O check after aggregation (line 203 equivalent)

**Remaining concerns:**
- ❌ No per-share check: sᵢ ≠ 0 before aggregation
- ❌ No range restriction: sᵢ could be 1 or r-1 (edge values)
- ❌ No check: ρᵢ = 1 (leaks M = G_G16, see PVUGC-006)

**Attack:**
```
Malicious armer sets sᵢ = 0:
1. Tᵢ = 0·G = O (point at infinity)
2. PoCE-A: Might not explicitly check Tᵢ ≠ O per-share
3. If caught at aggregation: Still causes arming phase failure
4. If not caught: α = Σⱼ sⱼ (missing sᵢ contribution)
```

---

## Attack Vectors (Updated for v2.0)

### Attack 4.1: Griefing via Invalid Ciphertext (Unchanged)

**Severity:** High
**Likelihood:** Medium (requires malicious armer participation)

```
Preconditions:
- Attacker participates as armer (or compromises armer)
- Protocol accepts arming shares without full verification

Attack steps:
1. Malicious armer generates valid (ρᵢ, sᵢ)
2. Computes valid D₁,ⱼ = Uⱼ^ρᵢ, D₂,ₖ = Vₖ^ρᵢ, Tᵢ = sᵢ·G
3. Generates valid PoCE-A proof
4. Publishes garbage ciphertext: ctᵢ = rand()
5. Arm-time checks pass ✅
6. Pre-signature created
7. Prover generates expensive Groth16+GS proof (minutes-hours)
8. Decapsulation: K derived, decryption fails or wrong sᵢ
9. PoCE-B check fails: Tᵢ ≠ Dec_K(ctᵢ)·G
10. Spend aborted, funds locked until timeout

Impact:
- Wasted computation (proof generation)
- Locked funds (CSV timeout needed)
- DoS if repeated
```

**v2.0 mitigation:** Publication SHOULD clause enables post-mortem identification (doesn't prevent attack).

### Attack 4.2: Threshold Manipulation (Mitigated)

**Severity:** High (if T=O check missing)
**Likelihood:** Low (v2.0 includes check)

```
Preconditions:
- k-of-n threshold arming
- (k-1) colluding malicious armers

Attack steps:
1. Malicious armers coordinate to set Σᵢ₌₁ᵏ⁻¹ sᵢ = -s_honest
2. Aggregated T = Σᵢ Tᵢ = O
3. If T=O check missing: α = Σᵢ sᵢ = 0
4. Adaptor equation: s = s' + α = s' + 0 = s'
5. Pre-signature is already valid signature
6. Anyone can spend without proof

Result: Complete break if check missing
```

**v2.0 mitigation:** ✅ T ≠ O check after aggregation (line 203) prevents this attack.

### Attack 4.3: Repeated Griefing (Unchanged)

**Severity:** Medium (liveness/DoS)
**Likelihood:** Medium (if attacker motivated)

```
Preconditions:
- Attacker repeatedly participates in arming rounds
- No penalty mechanism

Attack steps:
1. For each transaction attempt:
   a) Attacker publishes valid PoCE-A, broken ct
   b) Honest participants waste resources
   c) Transaction fails
2. Repeat indefinitely (low cost to attacker)

Impact:
- Protocol liveness degraded
- Economic attack: Wastes honest participants' resources
- No deterrent (no penalty in v1.0 or v2.0)
```

**v2.0 mitigation:** Publication SHOULD enables reputation tracking (external to protocol).

---

## Recommendations

### Phase 1: Short-Term Mitigations (1-2 months)

#### 1. Upgrade SHOULD to MUST for Publication
**Owner:** Protocol designers
**Timeline:** 1 week
**Deliverable:** Spec update

Change (line 240):
```diff
- SHOULD: Implementations SHOULD publish a minimal PoCE-B verification
- transcript (hashes of inputs/outputs) alongside the broadcasted spend
- to aid auditing.

+ MUST: Implementations MUST publish a PoCE-B verification transcript
+ (hashes of inputs/outputs) alongside the broadcasted spend. Format:
+ {
+   "armer_id": i,
+   "T_i": hash(Tᵢ),
+   "ct_i": hash(ctᵢ),
+   "s_i": hash(sᵢ),  // only if decap successful
+   "verification": "pass" | "fail"
+ }
```

**Impact:** Mandatory transparency, enables reputation systems.

#### 2. Per-Share Validation (Before Aggregation)
**Owner:** Implementation team
**Timeline:** 2 weeks
**Deliverable:** Validation functions

Add checks during arming phase:
```
For each share i:
- [ ] Tᵢ ≠ O (point at infinity) - reject immediately
- [ ] sᵢ ∈ [2, r-2] (after decryption) - reject edge values
- [ ] ρᵢ ∉ {0, 1} - prevent trivial cases
- [ ] order(Tᵢ) = n (full group order) - prevent small subgroup
```

**Impact:** Early detection of some malicious shares.

#### 3. Bond/Deposit Mechanism (External)
**Owner:** Application layer
**Timeline:** 1 month
**Deliverable:** Smart contract or protocol layer

**Option A: Bitcoin-native**
- Armers post bond in separate UTXO
- Bond locked with penalty condition
- If PoCE-B fails (proven via publication transcript): bond slashed
- Challenge: Requires oracle or multi-sig adjudication

**Option B: Layer-2 reputation**
- Off-chain registry of armers
- Failed PoCE-B → reputation penalty
- Low-reputation armers excluded from future rounds
- Challenge: Centralization concerns

**Impact:** Economic disincentive for malicious armers.

### Phase 2: Medium-Term Solutions (3-6 months)

#### 4. Make PoCE-B Publicly Verifiable (RECOMMENDED)

**Option A: Commitment-Based Approach**

Modify arming phase to include commitment:
```
PoCE-A proves:
1. D₁,ⱼ = Uⱼ^ρᵢ, D₂,ₖ = Vₖ^ρᵢ (existing)
2. Tᵢ = sᵢ·G (existing)
3. ρᵢ ≠ 0 (existing)
4. **NEW:** com_i = Commit(sᵢ, hᵢ, rᵢ) (Pedersen commitment)

Armer publishes: {Dⱼ, Dₖ, Tᵢ, ctᵢ, com_i, PoCE-A}

After decapsulation, decapper publishes:
- Opening: (sᵢ, hᵢ, rᵢ)
- Anyone can verify:
  * Open(com_i, sᵢ, hᵢ, rᵢ) ?= true
  * Tᵢ ?= sᵢ·G
```

**Pros:** Publicly verifiable, malicious armers immediately identified
**Cons:** Requires revealing sᵢ after decap (leaks adaptor secret α)

**Mitigation for cons:** Only reveal after spend confirmed (α already public from signature).

**Option B: VE-Based PoCE (Advanced)**

Use Verifiable Encryption:
```
PoCE-A proves (in ZK):
1. I know (ρᵢ, sᵢ, Kᵢ) such that:
   - D₁,ⱼ = Uⱼ^ρᵢ
   - D₂,ₖ = Vₖ^ρᵢ
   - Tᵢ = sᵢ·G
   - ctᵢ = Enc_Kᵢ(sᵢ, hᵢ)
   - Kᵢ is derived from correct KDF process
```

**Pros:** Fully publicly verifiable at arm-time
**Cons:** Complex ZK circuit, significant overhead

**Recommendation:** Start with Option A (commitment-based).

#### 5. Redundant Arming Shares
**Owner:** Protocol designers
**Timeline:** 2 months
**Deliverable:** Protocol extension

**Idea:** Overprovision arming shares
```
Setup: k-of-n threshold
Modification: Generate n+m shares (m redundant)
- If up to m shares are malicious: k honest shares remain
- Decapper attempts all shares, uses first k valid ones
```

**Pros:** Tolerates malicious shares without protocol failure
**Cons:** Increased arming cost, complexity in share selection

### Phase 3: Long-Term Solutions (6+ months)

#### 6. On-Chain Fraud Proofs
**Owner:** Research + implementation team
**Timeline:** 6-12 months
**Deliverable:** Fraud proof system

**Design:**
```
1. If PoCE-B fails, decapper generates fraud proof:
   - Proof that ct decrypts to (s'ᵢ, h'ᵢ) where T ≠ s'ᵢ·G
   - Or proof that ct doesn't decrypt properly

2. Fraud proof submitted on-chain (Bitcoin or side-chain)

3. Armer's bond slashed if fraud proof verifies

4. Slashed funds distributed to victims
```

**Challenges:**
- Bitcoin script limitations (no pairing verification)
- Requires covenant opcodes or side-chain
- Complexity in verification logic

**Alternative:** Use BitVM or similar for fraud proof verification on Bitcoin.

---

## Acceptance Criteria for Resolution

This flaw can be considered **RESOLVED** when **ONE** of the following is achieved:

### Option A: Publicly Verifiable PoCE-B (Preferred)
- [ ] PoCE-A extended to prove ciphertext correctness
- [ ] Verification possible at arm-time (before proof generation)
- [ ] Invalid ciphertexts rejected immediately
- [ ] No wasted computation from malicious armers

### Option B: Economic Penalties (Acceptable)
- [ ] Bond/deposit mechanism implemented
- [ ] Failed PoCE-B results in slashed bond
- [ ] Penalty is on-chain or cryptographically enforced
- [ ] Economic disincentive sufficient to deter attacks

### Option C: Redundancy + Detection (Minimum)
- [ ] Redundant arming shares (n+m for k-of-n)
- [ ] Publication MUST (not SHOULD) for verification transcripts
- [ ] Reputation system tracks malicious armers
- [ ] Protocol tolerates up to m malicious shares without failure

---

## Related Flaws

- **PVUGC-006:** Degenerate values (✅ resolved - includes T ≠ O check, mitigates Attack 4.2)
- **PVUGC-007:** Timing/race conditions (related - early aggregation check timing)

---

## Notes

- **v2.0 Status:** Partial improvement (publication SHOULD), but core issue remains
- **Priority:** High - affects liveness and creates griefing vector
- **Mainnet considerations:** Acceptable with:
  - Bond/deposit mechanism (external)
  - Redundant arming shares
  - Strong reputation system
- **User impact:** Potential for wasted resources and locked funds
- **Recommended:** Implement Option A (publicly verifiable PoCE-B) before mainnet

---

**Version History:**
- **v1.0 (2025-10-07):** Initial identification (PoCE-B decapper-local, no mitigation)
- **v2.0 (2025-10-07):** Partial improvement (publication SHOULD clause added)

**Last Updated:** 2025-10-07
**Next Review:** After PoCE-B redesign (target: 3-6 months)
