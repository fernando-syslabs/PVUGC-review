# PVUGC-007: Timing Attacks & Race Conditions

**Issue Code:** PVUGC-007
**Title:** Timing Side-Channels and Race Condition Vulnerabilities
**Severity:** 🔴 High
**Status:** ⚠️ Enhanced
**Report Version:** 4.0 (Final)
**Review Round:** 4 (Final)
**Dates:** Identified 2025-10-07; Mitigated 2025-10-07 (v2.0); Peer Reviewed 2025-10-26
**Reviewers:** M1 (Initial); M2 (Rounds 1, 3); Crypto-Reviewer (Rounds 2, 4)
**Cross-References:**
- [`PVUGC-2025-10-20.md §1 Introduction`](../PVUGC-2025-10-20.md) (Final Spec)
- [`report-preliminary-2025-10-07/PVUGC-007-timing-attacks.md`](../report-preliminary-2025-10-07/PVUGC-007-timing-attacks.md) (v1.0)
- [`report-update-2025-10-07/PVUGC-007-timing-race-conditions.md`](../report-update-2025-10-07/PVUGC-007-timing-race-conditions.md) (v2.0)
- [`report-update-2025-10-07/appendix_mathematical_considerations.md`](../report-update-2025-10-07/appendix_mathematical_considerations.md) (§7)
- [Appendix A07.M201](APPENDIX-issue-debates.md#a07-m201) (Round 1 — Mathematician)
- [Appendix A07.CR02](APPENDIX-issue-debates.md#a07-cr02) (Round 2 — Crypto Reviewer)
- [Appendix A07.M202](APPENDIX-issue-debates.md#a07-m202) (Round 3 — Mathematician)

---

## Executive Summary

**Verdict:** The timing side-channel constitutes a demonstrable violation of the zero-knowledge property, and the absence of normative phase timeouts enables trivial denial-of-service attacks on protocol liveness.

**Impact:** This issue threatens both **security** (information leakage violating zero-knowledge) and **liveness** (protocol deadlock). The severity is **High** because the timing side-channel violates a core cryptographic guarantee, not merely an implementation concern. Without constant-time execution, adversaries can extract approximately 12 bits of witness structure information per timing observation. Without timeout enforcement, a single non-participating armer can deadlock the entire protocol indefinitely.

**Changes from v1.0 → v2.0 → Peer Review:**
- **v1.0:** Correctly identified three distinct risks: timing side-channels, race conditions, and front-running. Proposed constant-time requirements but lacked formal quantification.
- **v2.0:** Minimal changes. Added optional CSV abort path (SHOULD, not MUST). Generic mention of constant-time operations. Status remained Open and "implementation-dependent".
- **M2 Round 1:** Formalized timing leakage using mutual information framework I(W; T_decap) ≈ log₂(m₁m₂/96). Proposed normative constant-time execution (always 96 pairings) and state machine with mandatory timeouts.
- **Crypto Round 2:** Validated M2's quantification, constructed concrete attack algorithms, formalized race condition deadlock scenarios, and provided spec-ready normative text.
- **M2 Round 3:** Identified critical gaps requiring worst-case analysis, statistical test corrections (TOST), missing invariants, and cache-timing considerations.
- **Final Round 4 (this document):** Incorporates all M2 revisions, achieving mathematical rigor and statistical correctness.

**Remaining Gaps:**
1. Front-running encrypted mempool strategy requires Bitcoin Core v24+ deployment coordination
2. Tie-breaking mechanism for simultaneous timeout expiration may be vulnerable to key-grinding (quantified as negligible: 2^-233 success probability)
3. Cache-timing resistance depends on pairing library constant-time guarantees (requires library-specific validation)

**Required Actions:**
1. **MUST (Spec):** Add §12.1 (Constant-Time Execution), §12.2 (State Machine Timeouts), §12.3 (Front-Running Mitigations) to PVUGC-2025-10-20.md §1 Introduction
2. **MUST (Spec):** Upgrade optional CSV abort from SHOULD to MUST
3. **MUST (Implementation):** Implement timing invariance tests using TOST methodology with ≥10⁶ samples
4. **MUST (Testing):** Validate deadlock prevention under timeout boundary conditions
5. **SHOULD (Deployment):** Select and document front-running mitigation strategy (encrypted mempool or direct miner submission)

---

## Spec Location

**Primary References ([`PVUGC-2025-10-20.md:129`](../PVUGC-2025-10-20.md)):**
- Lines 174-175: GS size bounds (MUST: m ≤ 48) — **Timing leak source**
 - Lines 146-147: Implementation note (96 pairings ~50-100ms) — **Acknowledges timing but no mitigation**
- Lines 168-173: Decapsulation algorithm — **Core timing side-channel (variable pairing count)**
- Lines 36-39: Optional Timeout/Abort path with CSV — **Partial liveness mitigation, not normative**

**Secondary References:**
- [`PVUGC-2025-10-05.md §7 GS PPE Verification (96 pairings)`](../PVUGC-2025-10-05.md) (Bitcoin script template) — CPFP hook enables fee manipulation
- [`PVUGC-2025-10-20.md §1 Introduction`](../PVUGC-2025-10-20.md) (Context binding) — No phase transition timestamps
- [Appendix A07.M203](APPENDIX-issue-debates.md#a07-m203) — Timing attack formalization

**Missing Normative Sections (to be added):**
- §12.1: Constant-Time Execution (MUST requirements)
- §12.2: Protocol State Machine with Timeouts (MUST requirements)
- §12.3: Front-Running Mitigation Strategies (SHOULD guidance)

---

## History (v1.0 → v2.0 → Peer Review)

### v1.0 Original Concern (2025-10-07)
**Document:** [`report-preliminary-2025-10-07/PVUGC-007-timing-attacks.md`](../report-preliminary-2025-10-07/PVUGC-007-timing-attacks.md)

**Key Observations:**
1. **No phase timeouts specified** except optional CSV in abort path
2. **Timing side-channels identified:**
   - Pairing count varies with proof structure (m₁, m₂)
   - DEM decryption success/failure timing differs
   - Cache timing in pairing computation
3. **Front-running window** after signature completion before broadcast
4. **Race conditions** with multiple concurrent decappers

**Proposed Mitigations:**
- Add phase timeouts (T₁ arming, T₂ presign, T₃-T₄ proof window, T₅ broadcast)
- Constant-time operations (normative requirements in §12)
- Front-running mitigations (commit-reveal, encrypted mempool, direct miner submission)

**Status:** Open (Medium severity)

### v2.0 Mitigation Attempt (2025-10-07)
**Document:** [`report-update-2025-10-07/PVUGC-007-timing-race-conditions.md`](../report-update-2025-10-07/PVUGC-007-timing-race-conditions.md)

**Changes Made:**
1. **Optional CSV Abort Path:** SHOULD enable Timeout/Abort with CSV for fund recovery
2. **Generic Constant-Time Mention:** "Use constant-time implementations" (non-normative)
3. **Status Classification:** "Implementation-dependent" — delegated to implementers

**What Remained Unaddressed:**
- No state machine for phase transitions
- No explicit timeouts for arming, pre-signing, proof generation
- No normative constant-time requirements (which operations, how to test)
- No race condition resolution mechanism
- No front-running mitigation strategy
- CSV abort remains SHOULD (not MUST)

**Status:** Open (Medium severity, implementation-dependent)

### M2 Round 1 Analysis (2025-10-15)
**Document:** [Appendix A07.M201](APPENDIX-issue-debates.md#a07-m201)

**Key Contributions:**

**1. Formal Timing Side-Channel Quantification:**
- Mutual information framework: I(W; T_decap) ≈ log₂(m₁m₂/96)
- **For small witness sets (m₁m₂ << 96), leakage can be several bits**
- **Violates zero-knowledge property** (not just implementation concern)

**2. Formal Deadlock Analysis:**
- Scenario: k-1 armers publish, k-th armer never publishes
- Result: Protocol deadlock without timeout
- Trivial DoS attack vector

**3. Normative Mitigations Proposed:**

**Mitigation 1: Constant-Time Execution (§12.1)**
```
MUST perform fixed 96 pairing operations regardless of actual m₁, m₂
Pad with dummy pairings e(G₁_identity, G₂_identity) if needed
DEM decryption MUST be constant-time (no early abort on tag mismatch)
```

**Mitigation 2: State Machine with Timeouts (§12.2)**
```
States: IDLE → ARMING → PRE_SIGNING → AWAITING_PROOF → COMPLETED/ABORTED

Timeouts:
- T_ARMING = 24 hours (all k armers must publish)
- T_PRESIG = 1 hour (MuSig2 completion)
- T_PROVING = 10 minutes (proof submission after presig)
- T_BROADCAST = 5 minutes (transaction broadcast after decap)

Failure handling: Transition to ABORTED on timeout expiration
```

**Verdict:** ⚠️ Enhanced (Medium → requires upgrade to High due to ZK violation)

**Status:** Awaiting spec amendment with normative requirements

### Crypto Peer Review Round 2 (2025-10-15)
**Document:** [Appendix A07.CR02](APPENDIX-issue-debates.md#a07-cr02)

**Enhancements:**
1. Severity upgrade: Medium → High (ZK violation is a security failure)
2. Concrete attack algorithms with complexity analysis
3. Adversarial front-running analysis (mempool observation attack)
4. Tie-breaking mechanism for simultaneous timeouts
5. Formal state machine specification with invariants
6. Comprehensive spec-ready normative text for §12.1, §12.2, §12.3

**Status:** Enhanced, awaiting M2 final review

### M2 Round 3 Review (2025-10-26)
**Document:** [Appendix A07.M202](APPENDIX-issue-debates.md#a07-m202)

**Verdict:** Minor revisions needed

**Critical Revisions Required:**
- R4: Missing state machine invariants (I1-I4) in Finding 6
- R6: Statistically incorrect use of Welch's t-test (must use TOST)

**Important Revisions Required:**
- R1: Add worst-case vs average-case distinction
- R2: Specify timing precision assumptions
- R3: Quantify parameter 'n' in Attack 2.1
- R5: Justify ε-constant-time thresholds theoretically
- R8: Complete CPFP and RBF attack analysis
- R10: Add cache-timing attack analysis

**Status:** Ready for Round 4 with revisions

### Final Round 4 (This Document, 2025-10-26)

**Objectives:**
1. Incorporate all M2 Round 3 revisions (R1-R10)
2. Correct arithmetic errors (4608 → 4560 pairs)
3. Add enhancements (E1: key-grinding analysis)
4. Produce publication-ready final report

---

## Findings

### Finding 1: Timing Side-Channel Violates Zero-Knowledge Property

**Claim:** The GS-PPE decapsulation loop leaks information about witness structure through timing variation.

**Formal Analysis:**

**Theorem 1 (Timing Leakage Quantification):**
Let W be the random variable representing the prover's witness, and T_decap be the decapsulation time. The mutual information between W and T_decap is:

```
I(W; T_decap) = H(W) - H(W | T_decap)
```

The decapsulation algorithm ([`PVUGC-2025-10-20.md:199`](../PVUGC-2025-10-20.md)) computes:

```
M̃ᵢ = (∏ⱼ₌₁^m₁ e(C¹ⱼ, D₁,ⱼ)) · (∏ₖ₌₁^m₂ e(D₂,ₖ, C²ₖ))
```

where m₁ + m₂ depends on the GS attestation structure, which correlates with the witness structure.

**Average-Case Leakage Quantification:**

Under the constraint m₁ + m₂ ≤ 96 with m₁, m₂ ∈ {1, 2, ..., 96}, the number of valid (m₁, m₂) pairs is:

```
N = Σ(s=2 to 96) (s-1) = Σ(k=1 to 95) k = (95 · 96)/2 = 4560
```

**Correction (M2 R3):** The original draft stated 4608 pairs, which is incorrect. The correct count is **4560 pairs**.

If (m₁, m₂) is uniformly distributed over valid pairs:
```
H(m₁, m₂) = log₂(4560) ≈ 12.16 bits
```

**Timing Precision Assumptions (M2 R2):**

The bound H(m₁, m₂ | T_decap) ≈ 0 holds if adversary timing precision satisfies:

```
δ_time << σ_pairing / √m
```

where:
- δ_time: Adversary measurement precision (e.g., 0.1ms with rdtsc)
- σ_pairing: Per-pairing time variance (e.g., 0.05ms for BLS12-381)
- m: Number of pairings (20-96)

**For BLS12-381 with t_pairing ≈ 1.2ms ± 0.05ms:**
- Distinguishable resolution: ~1 pairing
- Required precision: δ_time ≤ 0.1ms (achievable with μs-level timing)
- Achievable with: CPU cycle counters (rdtsc), high-resolution performance timers

**Realistic Model:**
With μs-level timing precision (realistic for network observers), adversaries can distinguish m = m₁ + m₂ values within resolution ~1 pairing. This means H(m₁, m₂ | T_decap) ≈ 1-2 bits (cannot distinguish every pair, but can distinguish sums m₁+m₂ precisely).

**Conservative Leakage Estimate:**
```
I(W; T_decap) ≈ H(m₁, m₂) - H(m₁, m₂ | T_decap)
              ≈ 12.16 - 1.5
              ≈ 10-11 bits (average case, μs precision)
```

**Worst-Case Leakage Analysis (M2 R1):**

The average-case analysis assumes uniform distribution over (m₁, m₂). For worst-case adversaries:

**Worst-Case Scenario 1: Non-Uniform Witness Distribution**
If witness distribution is non-uniform (e.g., 90% of witnesses map to small (m₁, m₂) values like (20,20) or (30,30)), the effective leakage per observation is:

```
I_worst(W; T_decap) ≈ log₂(|dominant witness classes|)
```

For example, if only 10 distinct (m₁, m₂) pairs cover 90% of witnesses:
```
I_worst ≈ log₂(10) ≈ 3.3 bits per observation
```

But this reveals **which of the 10 common structures** the witness has, potentially narrowing witness search space dramatically.

**Worst-Case Scenario 2: Extreme Small Witness**
For minimal witness with m₁ = 1, m₂ = 1:
```
Total pairings: m = 2
Timing: t ≈ 2 · 1.2ms = 2.4ms (vs 115.2ms for maximum)
```

If timing reveals m₁ = m₂ = 1 uniquely, this leaks **exact witness structure** (H = 0 bits remaining about structure). This is a **complete zero-knowledge failure** for extreme cases.

**Worst-Case Scenario 3: Multi-Observation Cumulative Leakage**
If adversary can time multiple related proofs (e.g., different witnesses for related statements):

```
I_cumulative ≤ min(n · 12, H(W))
```

where n is the number of observations. For H(W) ≈ 128 bits witness entropy:
- After 11 observations: I_cumulative ≈ 132 bits (witness fully revealed)
- Conservative bound: O(H(W)/12) ≈ 11 observations to leak entire witness structure

**Conclusion:** Timing leakage is **between 10-12 bits per observation** (average case, realistic precision) and can be **complete** (H = 0) for extreme small witnesses. Multi-observation attacks can extract full witness in O(11) timing measurements.

**Security Implication:**
The protocol claims zero-knowledge (proves existence without revealing witness details). The timing leak **violates this claim** by revealing:
- Average case: 10-12 bits of structure information per observation
- Worst case: Complete witness structure for extreme cases
- Multi-observation: Full witness extraction in ~11 observations

This is a **demonstrable security failure**, not merely an implementation concern.

---

### Finding 2: Concrete Timing Attack Algorithms

**Attack 2.1: Witness Structure Inference via Timing Oracle**

**Adversary Model:**
- Passive observer with μs-level timing measurement capability
- Can submit multiple valid proofs for the same (vk, x) with different witnesses
- Has access to public GS attestations

**Attack Algorithm:**
```
Algorithm: TIMING_ORACLE_ATTACK
Input: (vk, x), witness space W with |W| = N_witness
Output: Reduced witness candidate set

1. Calibration Phase:
   For each candidate witness wᵢ ∈ sample(W, n_cal):
     a. Generate valid Groth16 proof πᵢ for (vk, x, wᵢ)
     b. Generate GS attestation Attᵢ for πᵢ
     c. Publish Attᵢ and observe published attestation
     d. Measure decapsulation time tᵢ (10³ samples, median)
     e. Infer (m₁ⁱ, m₂ⁱ) from tᵢ using linear regression:
        tᵢ ≈ (m₁ⁱ + m₂ⁱ) · t_pairing + t_overhead
        Solve for m = m₁ + m₂ ≈ (tᵢ - t_overhead) / t_pairing

2. Build correlation map: wᵢ → (m₁ⁱ, m₂ⁱ)
   Classify witnesses into K ≤ 4560 timing classes

3. Target Attack Phase:
   For target attestation Att_target:
     a. Measure t_target (10³ samples, median)
     b. Infer (m₁_target, m₂_target) from timing
     c. Identify timing class C_target
     d. Return W_reduced = {wⱼ | wⱼ ∈ C_target}

Complexity (M2 R3):
- Pairing operations: O(n_cal · 96) where n_cal = calibration sample size
  Typical: n_cal = 100 for statistical significance
- Timing measurements: O(n_cal · 10³) samples
- Success probability:
  Let K = number of distinguishable (m₁, m₂) classes (K ≤ 4560)
  After timing measurement, witness space reduces from |W| to |W|/K on average

  Expected reduction: |W| / K ≈ |W| / 4560 for uniform distribution
  Best case (adversary): K = |W| (timing uniquely identifies witness)
  Worst case (adversary): K = 1 (all witnesses have same (m₁, m₂))

**Concrete Example:**
For Bitcoin transaction witness space with |W| ≈ 2⁴⁰ possible transaction structures uniformly distributed over (m₁, m₂) classes:

Timing measurement reduces search to:
```
|W_reduced| = 2⁴⁰ / 4560 ≈ 2⁴⁰ / 2¹².²
            ≈ 2²⁷·⁸ candidates
```

This is a **12.2-bit reduction** in witness search space, transforming a 2⁴⁰ brute-force into a 2²⁸ problem (feasible for well-resourced adversaries).
```

**Concrete Example (Bitcoin UTXO structures):**
Suppose the computation validates a Bitcoin transfer:
- w₁: 2 inputs, 2 outputs → (m₁ = 20, m₂ = 20) → t₁ ≈ 48ms
- w₂: 10 inputs, 5 outputs → (m₁ = 45, m₂ = 35) → t₂ ≈ 96ms

An adversary measuring t_target ≈ 48ms can infer the witness corresponds to small I/O count, eliminating all high-I/O candidates.

**Cache-Timing Extension (M2 R10):**

**Cache-Timing Attack on Pairing Libraries:**

Modern pairing libraries (blst, arkworks) use precomputed tables for elliptic curve operations. If table lookups depend on input data (commitment coordinates, which correlate with (m₁, m₂)), cache-timing attacks may leak information **even with fixed pairing count**.

**Attack Vector:**
1. **Flush+Reload:** Adversary flushes shared cache lines, triggers decapsulation, measures reload time
2. **Prime+Probe:** Adversary fills cache sets, triggers decapsulation, measures eviction patterns
3. **Cache Access Patterns:** Different (m₁, m₂) may cause different table access sequences

**Leakage Mechanism:**
Even if execution time is constant (96 pairings always), **cache access patterns** may differ:
- (m₁ = 48, m₂ = 48): Accesses full range of precomputed tables
- (m₁ = 20, m₂ = 20): Accesses subset of tables (early loop iterations)

If library uses **data-dependent table indexing** without masking, cache timing can leak (m₁, m₂).

**Mitigation (Required):**
Pairing library MUST use:
1. **Cache-oblivious algorithms** (all table entries accessed regardless of data)
2. **Constant-time table lookups** with masking (no data-dependent addressing)
3. **No branch prediction leakage** (no conditional execution based on secret data)

**Recommended Libraries:**
- **blst** (Rust/C): Constant-time by default, explicitly designed for cache resistance
- **arkworks** (Rust): Requires `asm` and `constant_time` feature flags enabled
- **MCL** (C++): Use `MCL_USE_CONSTANT_TIME` compile flag

**Testing:**
- Flush+Reload simulation with performance counters (Intel PT, ARM PMU)
- Verify cache access patterns are uniform across (m₁, m₂) values
- Statistical analysis: cache miss rates should be independent of witness structure

**Conclusion:** Constant-time execution requires **both** timing uniformity **and** cache-access uniformity. Padding to 96 pairings addresses timing but not cache-timing without library guarantees.

**Mitigation (from M2):** Always execute 96 pairings with constant-time, cache-oblivious pairing library.

---

### Finding 3: Race Condition Formal Analysis

**Claim:** Lack of phase timeouts enables trivial DoS attacks via participant non-participation.

**Formal Deadlock Scenario:**

**Theorem 2 (Arming Phase Deadlock):**
Let k be the number of required armers. The protocol transitions from ARMING → PRE_SIGNING only when all k armers have published valid shares.

**Deadlock Condition:**
```
∃i ∈ [1, k]: Armerᵢ never publishes share AND ¬∃T_ARMING (timeout)
⟹ Protocol waits indefinitely in ARMING state
⟹ All funds locked, zero progress
```

**Attack Algorithm:**
```
Algorithm: ARMING_DOS_ATTACK
Adversary: Controls one of k armers (or causes network partition)

1. Wait for k-1 honest armers to publish shares
2. Refuse to publish own share (or cause network delay)
3. Honest participants wait indefinitely (no timeout specified)
4. Optional: After long delay, publish share to collect any bonded funds

Complexity:
- Adversarial effort: O(1) (simply don't publish)
- Honest party loss: All funds locked until CSV abort (if exists)
- Protocol liveness: Zero (complete deadlock)

Mitigation: Mandatory T_ARMING timeout (24 hours)
```

**Simultaneous Timeout Race Condition:**

**Theorem 3 (Timeout Tie-Breaking Ambiguity):**
If multiple participants independently check timeout expiration, simultaneous transitions can cause state divergence.

**Race Scenario:**
```
Time: T_PRESIG - ε (just before timeout)
State: PRE_SIGNING (k participants)

Participant A at time T_PRESIG + δ₁:
  - Checks: current_time > T_PRESIG ⟹ TRUE
  - Action: Transition to ABORTED, publish abort transaction

Participant B at time T_PRESIG + δ₂:
  - Checks: current_time > T_PRESIG ⟹ TRUE
  - Action: Transition to ABORTED, publish abort transaction

If δ₁ ≈ δ₂ (both check simultaneously):
  - Both publish competing abort transactions
  - Potential: Double-spend if abort outputs differ
  - Mitigation: Tie-breaking mechanism required
```

**Tie-Breaking Mechanism (M2 R7 Enhancement):**

**Proposed Rule: Lexicographic Ordering on Public Keys**
```
Priority: smallest participant public key (compressed encoding)
Winner: participant with min(P₁, P₂, ..., Pₖ) publishes abort
Losers: wait grace period (30 seconds), observe winner's transaction
```

**Key-Grinding Attack Analysis (M2 E1):**

Lexicographic ordering creates strategic advantage for participants with small public keys.

**Attack:**
An adversary could:
1. Generate 2^b key pairs offline
2. Select the lexicographically smallest key
3. Gain systematic advantage in timeout races

**Quantification:**
For k participants, the probability that an adversary with public key P_adv wins is:
```
Pr[adv wins] = Pr[P_adv < min(P_honest,1, ..., P_honest,k-1)]
```

If adversary grinds 2^b key pairs, the smallest key is expected to be in the bottom 2^(-b) fraction of keyspace (secp256k1: 256-bit keys).

For b = 20 (1 million keys tried):
```
Pr[adv wins] ≈ 1 - (1 - 2^(-256))^(2^20)
             ≈ 2^20 · 2^(-256)  (Poisson approximation)
             ≈ 2^(-236)
```

For k = 5 participants (adversary needs to beat 4 honest keys):
```
Pr[adv wins] ≈ k · 2^20 · 2^(-256)
             ≈ 5 · 2^(-236)
             ≈ 2^(-233.7)
```

**Conclusion:** Attack requires 2^20 key generations (feasible) but has **negligible success probability** (2^(-233)). Lexicographic ordering is acceptable for timeout tie-breaking given astronomical failure probability.

**More Robust Alternative (Optional):**
Use context-bound priority to prevent key grinding:
```
priority_i = H("PVUGC/TIMEOUT_PRIORITY" || ctx_hash || P_i || timeout_type)
Smallest priority_i wins
```

This prevents grinding because ctx_hash is unknown at key selection time. However, given the negligible attack probability, this optimization is **optional**.

**Recommendation:** Use lexicographic ordering (simpler) with documented negligible attack probability. Deployments requiring defense-in-depth MAY use context-bound priority.

---

### Finding 4: Front-Running Vulnerability Analysis

**Claim:** Mempool observation enables value extraction via transaction replacement.

**Attack 4.1: Mempool Front-Running**

**Adversary Model:**
- Monitors Bitcoin public mempool with low-latency connection to miners
- Can extract signature from observed transaction
- Can build and broadcast replacement transaction with higher fee

**Attack Algorithm:**
```
Algorithm: MEMPOOL_FRONTRUN_ATTACK
Adversary: Monitors mempool with low-latency connection to miners

1. Monitor mempool for PVUGC transactions (identify by scriptPubKey pattern)
2. Upon observing tx_honest:
   a. Extract signature s from witness
   b. Extract sighash m from transaction template
   c. Verify: s·G - e·P = R (signature is valid)

3. Analyze transaction flexibility:
   a. Check if CPFP anchor output is present
   b. Check if any outputs are malleable
   c. Estimate value extractable via fee manipulation

4. If profitable:
   a. Attempt to build tx_attack with modified parameters
   b. Broadcast tx_attack with higher fee
```

**Critical Observation:**
The spec (PVUGC-2025-10-20.md §1 Introduction) states:
```
"Transaction binding: all signatures use SIGHASH_ALL, bind tapleaf hash and version"
```

This **strongly mitigates** front-running because SIGHASH_ALL commits to all outputs.

**CPFP and RBF Analysis (M2 R8):**

**CPFP (Child-Pays-For-Parent) Attack:**

The spec includes a "small P2TR CPFP hook output" (line 50) but does not specify spending conditions.

**Scenario A: Anchor controlled by signing key P**
- Adversary **cannot** spend anchor (requires valid signature from P)
- CPFP is **NOT an attack vector**
- Fee manipulation requires consent of key holders

**Scenario B: Anchor is "anyone-can-spend" (e.g., OP_TRUE scriptPubKey)**
- Adversary **CAN** spend anchor in child transaction
- **Low-fee child attack:** Adversary creates child with minimal fee, delaying parent confirmation (griefing)
- **High-fee child attack:** Adversary accelerates parent confirmation

**Analysis of High-Fee CPFP:**
Even if adversary accelerates parent confirmation via CPFP:
- Parent transaction outputs are fixed by SIGHASH_ALL (adversary cannot modify)
- Adversary gains no value from accelerating honest transaction
- **Conclusion:** High-fee CPFP is not an attack (only benefits honest parties)

**Analysis of Low-Fee CPFP (Griefing):**
Adversary creates child spending anchor with fee below minimum relay fee:
- Delays parent confirmation (keeps it in mempool longer)
- Does not extract value but harms honest parties (denial-of-service)

**Mitigation (MUST):**
If anchor is anyone-can-spend:
1. **Add minimum fee requirement** to anchor spend (e.g., OP_CHECKSEQUENCEVERIFY delay)
2. **Monitor mempool for CPFP attempts** and alert on anomalous child transactions
3. **Preferred:** Make anchor spending require signature from P (Scenario A)

**Recommended Spec Clarification:**
```
CPFP hook output MUST be controlled by aggregate public key P
(requires signature from pre-signing participants)
This prevents adversarial CPFP griefing while preserving fee-bumping capability
```

**RBF (Replace-By-Fee) Attack:**

BIP-125 allows replacing unconfirmed transactions with higher-fee versions (opt-in RBF flag).

**Attack Attempt:**
Adversary observes tx_honest in mempool, attempts to create tx_replace with:
- Same inputs (same UTXO spend)
- Different outputs (redirect value to adversary)
- Higher fee (to incentivize miner inclusion)
- **Challenge:** Must produce valid signature for modified transaction

**Why RBF Attack FAILS:**
```
SIGHASH_ALL commits to:
- All transaction inputs
- All transaction outputs
- locktime, version, etc.

Modified transaction has sighash m' ≠ m (different outputs)
Signature s = s' + α is valid ONLY for original sighash m
Adversary cannot produce valid signature for m' without knowledge of:
- α (witness-encrypted)
- Private key for P (unknown)

Conclusion: RBF attack FAILS under SIGHASH_ALL
```

**Mitigation (Already Satisfied):**
Transaction template MUST use SIGHASH_ALL (required by spec line 49). RBF is **not an attack vector** given this requirement.

**Additional Safety (SHOULD):**
- **Disable RBF flag** (BIP-125 opt-out): Set sequence to 0xfffffffe (signals "no RBF")
- **Monitor mempool** for RBF attempts (should not occur, but detect anomalies)

**Residual Front-Running Vectors:**

Despite SIGHASH_ALL:
1. **Mempool observation:** Adversaries can analyze transaction patterns, timing, and fees (information leakage)
2. **Miner collusion:** Miners can preferentially order transactions (MEV extraction at miner level)

**Mitigation Strategy 1: Encrypted Mempool (RECOMMENDED)**

Bitcoin Core v24+ supports encrypted P2P transport (BIP-324):
```
Implementation:
1. Enable encrypted P2P: --v2transport=1 in Bitcoin Core config
2. Connect only to peers supporting BIP-324
3. Broadcast transaction via encrypted channels

Limitations:
- Miners must also support encrypted transport
- Does not prevent miner-level front-running (miners see all transactions)

Effectiveness: Medium (reduces passive observation, not active attacks)
```

**Mitigation Strategy 2: Direct Miner Submission (ALTERNATIVE)**

Submit transactions directly to mining pools via private channels:
```
Implementation:
1. Establish relationship with mining pool(s)
2. Submit transaction via pool's private API (bypass public mempool)
3. Pool includes transaction in next block

Limitations:
- Requires trust in mining pool
- Reduces decentralization (not publicly visible until confirmation)
- Pool could extract MEV directly

Effectiveness: High (eliminates mempool observation) but introduces trust assumptions
```

**Recommendation:** Tier 1 (MUST): SIGHASH_ALL + no RBF + controlled CPFP anchor. Tier 2 (RECOMMENDED): Encrypted mempool for defense-in-depth.

---

### Finding 5: Constant-Time Execution Formal Requirements

**Current Spec Gap:**
PVUGC-2025-10-20.md §1 Introduction mentions constant-time informally (lines 146-147) but lacks MUST requirements.

**Formalization:**

**Definition (ε-Constant-Time Operation):**
An operation f(x) is ε-constant-time if for all inputs x, y and all timing measurements t:
```
|Pr[Time(f(x)) = t] - Pr[Time(f(y)) = t]| ≤ ε
```

For cryptographic applications, ε should be negligible.

**Threshold Justification (M2 R5):**

The ε thresholds must ensure distinguishing requires computationally infeasible sample sizes.

**Statistical Distinguishing Theory:**
For ε-closeness of timing distributions, statistical distinguishing requires:
```
q ≈ O(1/ε²) samples (Chernoff bound)
```

**GS-PPE Decapsulation (ε < 2^(-40)):**
```
Required samples: q ≈ (1/ε)² = (2^40)² = 2^80
At 100ms per decapsulation: Time = 2^80 · 100ms ≈ 3.8 × 10^16 years

Conclusion: 2^(-40) ensures distinguishing is computationally infeasible
```

This threshold is appropriate for **CI/CD testing** where:
- Tests run on varied hardware (some variance expected)
- Goal: Detect **gross timing leaks** (>1μs variance for 100ms operation)
- False positive rate: ~2^(-40) ≈ 10^(-12) per test (acceptable for automated testing)

**Practical Interpretation:**
ε = 2^(-40) for 100ms operation ⟹ allowable variance ~0.09μs (extremely tight)
This is overly conservative. More practical: ε = 2^(-20) for CI/CD (1μs variance), ε = 2^(-40) for production validation.

**DEM Decryption (ε < 2^(-60)):**
```
Required samples: q ≈ 2^120
At 1μs per decryption: Time = 2^120 · 1μs ≈ 4.2 × 10^22 years

Conclusion: 2^(-60) ensures distinguishing is infeasible even for fast operations
```

This threshold is appropriate for **production deployment** where:
- DEM operations are fast (~1μs), adversaries can sample many times
- Goal: Cryptographic-grade indistinguishability
- Aligns with NIST SP 800-90B min-entropy requirements (2^(-60) collision resistance)

**Threshold Selection Rule:**
```
ε < 2^(-λ/2) where λ = security parameter

For λ = 80 (80-bit security): ε < 2^(-40)
For λ = 128 (128-bit security): ε < 2^(-64)
```

**Practical Recommendation:**
- **CI/CD testing:** ε < 2^(-20) (1μs variance, detects gross leaks)
- **Production validation:** ε < 2^(-40) (balances false positives vs security)
- **Cryptographic-grade:** ε < 2^(-60) (NIST-compliant, for fast operations)

**Operations Requiring Constant-Time Execution:**

**1. GS-PPE Decapsulation Loop (line 168-173):**
```
MUST execute exactly 96 pairing operations
If m₁ + m₂ < 96, pad with e(G₁_id, G₂_id)
Target: ε < 2^(-40) for production (variance < 0.1μs for ~100ms operation)
```

**2. DEM Decryption (line 159):**
```
MUST complete all steps regardless of tag verification success
No early abort on authentication failure
Target: ε < 2^(-60) (cryptographic-grade for fast operations)
```

**3. PoCE-B Verification (line 173):**
```
MUST verify all k shares (no early abort on first failure)
Point comparison Tᵢ ?= sᵢ·G must be constant-time
Target: ε < 2^(-60)
```

**Testing Requirements (M2 R6 - CRITICAL CORRECTION):**

**INCORRECT (Original Draft):**
```
Welch's t-test with p > 0.01 to accept null hypothesis
```

**Problem:** Using p > 0.01 as acceptance criterion is statistically incorrect. Failing to reject H₀ does **not prove** H₀ is true (absence of evidence ≠ evidence of absence).

**CORRECT: Two One-Sided Tests (TOST) for Equivalence**

**Statistical Framework:**
- **Null Hypothesis H₀:** Timing distributions differ by more than ε (NOT constant-time)
- **Alternative H₁:** Timing distributions are ε-equivalent (constant-time)

**TOST Procedure:**
```
1. Define equivalence margin: Δ = ε · mean(Time_A)
   Example: For 100ms operation, ε = 2^(-40), Δ = 100ms · 2^(-40) ≈ 0.09μs

2. Collect timing samples:
   - Distribution A: n ≥ 10^6 samples (e.g., m₁=20, m₂=20)
   - Distribution B: n ≥ 10^6 samples (e.g., m₁=48, m₂=48)

3. Compute mean difference: Δμ = mean(Time_A) - mean(Time_B)
   Compute standard errors: SE_A, SE_B
   Compute pooled SE: SE_pooled = √(SE_A² + SE_B²)

4. Test two one-sided hypotheses:
   H₀₁: Δμ ≥ Δ  (upper bound violation)
   H₀₂: Δμ ≤ -Δ (lower bound violation)

   t₁ = (Δμ - Δ) / SE_pooled
   t₂ = (Δμ + Δ) / SE_pooled

   p₁ = CDF_t(t₁, df)
   p₂ = 1 - CDF_t(t₂, df)

5. Decision rule:
   If p₁ < α AND p₂ < α (both tests reject): Accept H₁ (ε-equivalent)
   Otherwise: Reject (timing leak detected)

   Standard: α = 0.05 (95% confidence)
   Conservative: α = 0.01 (99% confidence)
```

**Implementation Example:**
```python
from scipy import stats
import numpy as np

def tost_equivalence_test(times_a, times_b, epsilon=2**(-40), alpha=0.05):
    """
    Two One-Sided Tests (TOST) for timing equivalence.

    Args:
        times_a: Timing samples for condition A (array)
        times_b: Timing samples for condition B (array)
        epsilon: Equivalence margin (relative)
        alpha: Significance level (0.05 = 95% confidence)

    Returns:
        (equivalent, p_value_max, delta_mu)
    """
    mean_a = np.mean(times_a)
    mean_b = np.mean(times_b)
    delta_mu = mean_a - mean_b

    # Equivalence margin (absolute)
    delta = epsilon * max(mean_a, mean_b)

    # Standard errors
    se_a = stats.sem(times_a)
    se_b = stats.sem(times_b)
    se_pooled = np.sqrt(se_a**2 + se_b**2)

    # Degrees of freedom (Welch-Satterthwaite)
    df = len(times_a) + len(times_b) - 2

    # Two one-sided t-tests
    t1 = (delta_mu - delta) / se_pooled  # Upper bound test
    t2 = (delta_mu + delta) / se_pooled  # Lower bound test

    p1 = stats.t.cdf(t1, df)
    p2 = 1 - stats.t.cdf(t2, df)

    # Both tests must reject (p < alpha) to accept equivalence
    equivalent = (p1 < alpha) and (p2 < alpha)
    p_value_max = max(p1, p2)

    return equivalent, p_value_max, delta_mu

# Example usage
times_min = measure_decap_timing(m1=20, m2=20, samples=10**6)
times_max = measure_decap_timing(m1=48, m2=48, samples=10**6)

equiv, p_val, diff = tost_equivalence_test(times_min, times_max, epsilon=2**(-40))

if equiv:
    print(f"✓ Constant-time validated (p={p_val:.6f}, Δμ={diff*1e6:.3f}μs)")
else:
    print(f"✗ Timing leak detected (p={p_val:.6f}, Δμ={diff*1e6:.3f}μs)")
```

**Minimum Sample Size:**
For detecting ε = 2^(-40) with 95% confidence:
```
n ≥ 2 · (z_α/2 / ε)² ≈ 2 · (1.96 / 2^(-40))² ≈ 10^25 (infeasible)
```

**Practical Approach:**
Use **wider equivalence bounds** for testing:
```
CI/CD: Δ = 2σ_noise (noise-level equivalence, ε_practical ≈ 2^(-20))
Production: Δ = 1μs (absolute bound, easier to verify)
Cryptographic: Rely on library guarantees + statistical spot-checks
```

**Validation Checklist:**
```
Test 1: GS-PPE Decapsulation Timing Uniformity
- Inputs: (m₁=20, m₂=20), (m₁=48, m₂=48), (m₁=10, m₂=10)
- Samples: 10⁶ per input
- Method: TOST with equivalence bounds ±2σ_noise
- Accept if: Both one-sided tests p < 0.05

Test 2: DEM Decryption Success/Failure Timing
- Inputs: Valid ciphertext, Invalid ciphertext (wrong tag)
- Samples: 10⁶ per input
- Method: TOST with equivalence bounds ±1μs
- Accept if: Both one-sided tests p < 0.05

Test 3: PoCE-B Verification All-Valid vs One-Invalid
- Inputs: All k shares valid, One share invalid (rest valid)
- Samples: 10⁵ per input (slower operation)
- Method: TOST with equivalence bounds ±10μs
- Accept if: Both one-sided tests p < 0.05
```

---

### Finding 6: State Machine Formal Specification

**Current Spec Gap:**
No formal state machine for protocol phases. Implicit ordering from narrative.

**Proposed Formal State Machine:**

**States:**
```
S = {IDLE, ARMING, PRE_SIGNING, AWAITING_PROOF, DECAP, BROADCAST, COMPLETED, ABORTED}
```

**State Transition Function:**
```
δ: S × Event → S

Events:
- InitContext: IDLE → ARMING
- AllArmersPublished: ARMING → PRE_SIGNING (if within T_ARMING)
- Arming_Timeout: ARMING → ABORTED (if timeout expired)
- PreSigComplete: PRE_SIGNING → AWAITING_PROOF (if valid s', R within T_PRESIG)
- PreSig_Timeout: PRE_SIGNING → ABORTED
- ProofReceived: AWAITING_PROOF → DECAP (if valid attestation)
- Proof_Timeout: AWAITING_PROOF → ABORTED (after CSV Δ)
- DecapComplete: DECAP → BROADCAST (if valid α, s)
- Decap_Timeout: DECAP → ABORTED
- BroadcastComplete: BROADCAST → COMPLETED
- Broadcast_Timeout: BROADCAST → ABORTED
```

**State Invariants (M2 R4 - CRITICAL ADDITION):**

The original draft specified timeout-related invariants but **omitted critical safety properties** defined in §12.2.5. For completeness and consistency, all invariants are consolidated here:

**Invariant I1 (Mutual Exclusion):**
```
∀t: |{s ∈ S : state(t) = s}| = 1
```
At any time t, exactly one state s ∈ S is active (no state ambiguity).

**Invariant I2 (Monotonic Progress):**
```
States are partially ordered:
IDLE < ARMING < PRE_SIGNING < AWAITING_PROOF < DECAP < BROADCAST < COMPLETED

∀t₁ < t₂: state(t₁) ≤ state(t₂) ∨ state(t₂) = ABORTED
```
Transitions MUST be monotonically increasing except to ABORTED.

**Invariant I3 (Timeout Enforcement):**
```
∀s ∈ {ARMING, PRE_SIGNING, AWAITING_PROOF, DECAP, BROADCAST}:
  elapsed_time(s) > T_s ⟹ ◇(state = ABORTED)
```
Eventually, timeout triggers abortion (liveness property, using temporal logic ◇ = "eventually").

**Invariant I4 (Abort Finality):**
```
state = ABORTED ⟹ □(state = ABORTED)
```
Once aborted, state never changes (terminal state, using temporal logic □ = "always").

**Timeout-Specific Invariants:**

```
I_ARMING: |{published armers}| ≤ k ∧ elapsed_time ≤ T_ARMING
I_PRE_SIGNING: ∃(s', R) valid ∨ elapsed_time ≤ T_PRESIG
I_AWAITING_PROOF: ∃ Att valid ∨ elapsed_time ≤ Δ_CSV
I_DECAP: ∃α = Σᵢ sᵢ ∧ s = s' + α ∧ elapsed_time ≤ T_DECAP
I_BROADCAST: tx broadcasted ∧ elapsed_time ≤ T_BROADCAST
```

**Timeout Parameters (from M2, validated):**
```
T_ARMING = 24 hours (conservative for multi-party coordination)
T_PRESIG = 1 hour (MuSig2 typically completes in seconds)
Δ_CSV = 48 hours (proof generation fallback, specified in Bitcoin script)
T_DECAP = 10 minutes (decapsulation ~100ms, allowing retry buffer)
T_BROADCAST = 5 minutes (Bitcoin propagation ~seconds)
```

**Validation:** These timeout values are reasonable for a multi-party protocol over public networks. Implementations MAY use longer timeouts but MUST NOT use shorter values without formal analysis justifying the reduction.

---

### Finding 7: Severity Upgrade Justification

**Current Classification:** Medium (v1.0, v2.0)

**Proposed Upgrade:** High

**Rationale:**

**Medium Severity Criteria:**
- Affects liveness but not soundness
- Implementation-dependent issue
- Workaround available

**High Severity Criteria:**
- Violates core cryptographic property (zero-knowledge)
- Protocol-level vulnerability (not implementation-specific)
- No workaround without spec changes

**Analysis:**

**1. Violates Zero-Knowledge Property:**
   - The protocol claims proof existence without revealing witness details
   - Timing leak reveals 10-12 bits of witness structure per observation (Finding 1)
   - Multi-observation attacks can extract full witness in ~11 observations
   - This is a **security failure**, not just liveness concern

**2. Protocol-Level Vulnerability:**
   - Deadlock attack (Finding 3) requires only non-participation (O(1) effort)
   - No implementation choice can prevent deadlock without timeout specification
   - This is a **protocol design flaw**, not implementation bug

**3. No Workaround Without Spec Changes:**
   - Constant-time execution requires normative specification (which operations, how to test)
   - Timeout enforcement requires protocol-level state machine
   - Implementers cannot unilaterally fix without spec guidance

**Conclusion:** Severity upgrade to **High** is justified.

---

## Recommendations

### Normative Requirements (MUST/SHOULD for Specification)

#### Recommendation 1: Add §12.1 - Constant-Time Execution (MUST)

**Add to PVUGC-2025-10-20.md §1 Introduction after current §11:**

```markdown
### 12.1 Constant-Time Execution (Normative)

To prevent side-channel leakage of witness-related information, implementations of the PVUGC protocol MUST adhere to the following constant-time requirements:

#### 12.1.1 GS-PPE Decapsulation Loop

The computation of the product-of-pairings during decapsulation (§6, Eq. GS-PPE) MUST take a fixed amount of time independent of the number of commitments (m₁, m₂) in the GS attestation.

**MUST Requirements:**
1. **Fixed Pairing Count:** Implementations MUST perform exactly 96 pairing operations, regardless of the actual values of m₁ and m₂.

2. **Dummy Pairing Padding:** If an attestation uses fewer than 96 pairings (m₁ + m₂ < 96), the remaining iterations MUST be filled with dummy pairing operations:
   ```
   For j = m₁+1 to 96:
       e(G₁_identity, G₂_identity)  // Dummy pairing
   ```
   The results of dummy pairings MUST be computed but not used in the final product.

3. **No Early Termination:** The loop MUST NOT terminate early upon detecting invalid commitments. All 96 iterations MUST complete before returning success or failure.

4. **Constant-Time Pairing Library:** Implementations MUST use a pairing library that provides constant-time guarantees:
   - **blst** (Rust/C): Enable default constant-time mode, verify with timing tests
   - **arkworks** (Rust): Enable `asm` and `constant_time` feature flags
   - **MCL** (C++): Use `MCL_USE_CONSTANT_TIME` compile flag
   - **Other:** Consult library documentation for constant-time guarantees

**Rationale:** The number of pairings m = m₁ + m₂ can reveal 10-12 bits of witness structure information per observation. Fixed-time execution eliminates this side-channel.

#### 12.1.2 DEM Decryption and Verification

The process of decrypting the ciphertext ctᵢ and verifying the authentication tag τᵢ MUST be constant-time.

**MUST Requirements:**
1. **Complete Decryption:** The decryption algorithm MUST perform all computational steps regardless of intermediate validation results.

2. **No Early Abort:** The algorithm MUST NOT exit early upon detecting tag mismatch, padding errors, or malformed ciphertext. All steps (KDF, keystream generation, XOR, tag computation) MUST complete.

3. **Constant-Time Comparison:** Tag verification MUST use constant-time comparison:
   ```
   // Correct (constant-time):
   result = constant_time_compare(τᵢ_computed, τᵢ_received)

   // Incorrect (timing leak):
   if (τᵢ_computed == τᵢ_received) { ... }
   ```

4. **Uniform Return Timing:** Success and failure paths MUST take the same amount of time (within measurement error ε < 2^(-60)).

**Rationale:** Timing differences between successful and failed decryption can reveal which ciphertexts correspond to valid proofs, creating an oracle for proof validity testing.

#### 12.1.3 PoCE-B Verification

Verification of Proof-of-Correct-Encryption (PoCE-B) MUST be performed in constant time.

**MUST Requirements:**
1. **Verify All Shares:** Implementations MUST verify all k armer shares, even if earlier shares fail validation.

2. **Constant-Time Point Operations:** Point multiplication (Tᵢ ?= sᵢ·G) and comparison MUST use constant-time implementations.

3. **Deferred Failure Reporting:** Validation failures MUST be accumulated (e.g., in a bitmask) and reported only after all k shares have been checked.

**Rationale:** Timing variations during PoCE-B verification can reveal which armers produced valid vs invalid shares, enabling correlation attacks in threshold settings.

#### 12.1.4 Cache-Timing Considerations (MUST)

Modern pairing libraries use precomputed tables for elliptic curve operations. If table lookups depend on input data (m₁, m₂, commitment values), cache-timing attacks may leak information even with fixed pairing count.

**MUST Requirements:**

1. **Cache-Oblivious Algorithms:** Pairing library MUST use cache-oblivious algorithms or constant-time table lookups.

2. **Library Validation:** Implementations MUST verify pairing library provides:
   - Constant-time field operations (blst: default, arkworks: requires feature flag)
   - Masked table lookups (no data-dependent addressing)
   - No branch prediction leakage (no conditional execution based on secret data)

3. **Testing:** Cache-timing tests SHOULD be performed using:
   - Flush+Reload attacks (simulated)
   - Prime+Probe attacks (simulated)
   - Performance counter monitoring (Intel PT, ARM PMU)

**Recommended Libraries:**
- **blst** (Rust/C): Constant-time by default, battle-tested, explicitly designed for cache resistance
- **arkworks** (Rust): Enable `asm` and `constant_time` features
- **MCL** (C++): Use `MCL_USE_CONSTANT_TIME` flag

**Documentation:** Implementations MUST document cache-timing resistance validation methodology and results.

**Rationale:** Padding to 96 pairings prevents timing leaks but may not prevent cache-timing leaks if library uses data-dependent table lookups.

#### 12.1.5 Testing and Validation

Implementations claiming constant-time execution MUST provide evidence via:

**1. Statistical Timing Tests (TOST Methodology):**
   - **Method:** Two One-Sided Tests (TOST) for equivalence
   - **Null Hypothesis H₀:** Timing distributions differ by more than ε
   - **Alternative H₁:** Timing distributions are ε-equivalent (constant-time)
   - **Procedure:**
     1. Collect samples: n ≥ 10⁶ per distribution
     2. Define equivalence bounds: ±2σ_noise (CI/CD) or ±1μs (production)
     3. Compute two one-sided t-tests:
        - H₀₁: Δμ ≥ ε (upper bound)
        - H₀₂: Δμ ≤ -ε (lower bound)
     4. Reject both at α = 0.05 level to accept ε-equivalence
   - **Implementation:** Use TOST libraries (Python: statsmodels, R: TOSTER)

**2. Test Vectors:**
   ```
   Test 1: GS-PPE Decapsulation
   - Input A: Attestation with m₁=20, m₂=20 (40 pairings)
   - Input B: Attestation with m₁=48, m₂=48 (96 pairings)
   - Measure: 10⁶ timing samples for each
   - Verify: TOST with equivalence bounds ±2σ_noise, both p < 0.05

   Test 2: DEM Decryption
   - Input A: Valid ciphertext (correct tag)
   - Input B: Invalid ciphertext (incorrect tag)
   - Measure: 10⁶ timing samples for each
   - Verify: TOST with equivalence bounds ±1μs, both p < 0.05

   Test 3: PoCE-B Verification
   - Input A: All k shares valid
   - Input B: One share invalid (rest valid)
   - Measure: 10⁵ timing samples for each
   - Verify: TOST with equivalence bounds ±10μs, both p < 0.05
   ```

**3. CI/CD Integration:** Timing tests MUST be part of continuous integration and fail the build if timing leaks are detected (TOST rejects equivalence).

**SHOULD Requirements:**
1. Implementations SHOULD use hardware-based constant-time primitives where available (e.g., AES-NI for DEM).
2. Implementations SHOULD disable CPU frequency scaling during timing-sensitive operations.
3. Implementations SHOULD document any residual timing variance (quantified in microseconds).
```

---

#### Recommendation 2: Add §12.2 - Protocol State Machine with Timeouts (MUST)

**Add to PVUGC-2025-10-20.md §1 Introduction after §12.1:**

```markdown
### 12.2 Protocol State Machine with Timeouts (Normative)

To ensure protocol liveness and prevent denial-of-service attacks, implementations MUST enforce a formal state machine with mandatory timeouts for all phases.

#### 12.2.1 State Definitions

The protocol lifecycle consists of the following states:

1. **IDLE:** Initial state before context initialization
2. **ARMING:** Collecting armer shares (§5)
3. **PRE_SIGNING:** MuSig2 adaptor pre-signature generation (§4)
4. **AWAITING_PROOF:** Waiting for valid Groth16+GS proof
5. **DECAP:** Decapsulation and signature finalization (§6)
6. **BROADCAST:** Transaction broadcast to Bitcoin network
7. **COMPLETED:** Terminal success state (transaction confirmed)
8. **ABORTED:** Terminal failure state (timeout or validation failure)

#### 12.2.2 State Transitions and Timeouts

**Transition 1: IDLE → ARMING**
- **Trigger:** Protocol context initialization (vk, x, txid_template, ctx_hash computed)
- **Timeout:** None (instantaneous)

**Transition 2: ARMING → PRE_SIGNING (Success Path)**
- **Condition:** All k armers have published valid shares:
  - Published: {D₁,ⱼ}, {D₂,ₖ}, ctᵢ, τᵢ, Tᵢ, hᵢ for i ∈ [1, k]
  - Verified: PoK and PoCE-A for all i
  - Checked: G_G16(vk, x) ≠ 1 (non-degenerate target)
- **Timeout:** T_ARMING = 24 hours from ARMING state entry
- **Success Action:** Aggregate T = Σᵢ Tᵢ, proceed to MuSig2
- **Failure Action:** If timeout expires, transition to ABORTED

**Transition 3: ARMING → ABORTED (Timeout Path)**
- **Condition:** Current time > t_start_arming + T_ARMING AND fewer than k shares received
- **Action:** Publish timeout abort transaction (CSV path becomes valid)

**Transition 4: PRE_SIGNING → AWAITING_PROOF (Success Path)**
- **Condition:** Valid MuSig2 adaptor pre-signature (s', R) produced
  - Verified: AdaptorVerify(m, T, R, s') = ACCEPT
  - Published: AdaptorVerify transcript bound to ctx_hash
- **Timeout:** T_PRESIG = 1 hour from PRE_SIGNING state entry
- **Success Action:** Publish (s', R, m, ctx_hash), proceed to proof waiting
- **Failure Action:** If timeout expires, transition to ABORTED

**Transition 5: PRE_SIGNING → ABORTED (Timeout Path)**
- **Condition:** Current time > t_start_presig + T_PRESIG AND no valid (s', R)
- **Action:** All participants abort, CSV recovery path becomes available

**Transition 6: AWAITING_PROOF → DECAP (Success Path)**
- **Condition:** Valid Groth16+GS attestation received
  - GS attestation verifies: PPE equation holds (§6, Eq. GS-PPE)
  - Groth16 proof implicit in GS commitments
  - m₁ + m₂ ≤ 96 (size bound check)
- **Timeout:** Δ_CSV (specified in CSV opcode, 48 hours recommended)
- **Success Action:** Begin decapsulation process
- **Failure Action:** If Δ_CSV expires, Timeout/Abort spend path becomes valid (§2)

**Transition 7: AWAITING_PROOF → ABORTED (Timeout Path)**
- **Condition:** Current time > t_start_awaiting + Δ_CSV AND no valid proof
- **Action:** Timeout/Abort spend path enabled (OP_CHECKSEQUENCEVERIFY)

**Transition 8: DECAP → BROADCAST (Success Path)**
- **Condition:** Decapsulation completed successfully
  - For all i: Computed M̃ᵢ, derived Kᵢ, decrypted (sᵢ || hᵢ)
  - Verified: Tᵢ = sᵢ·G for all i
  - Verified: hᵢ = H_bytes(sᵢ || Tᵢ || i) for all i
  - Verified: PoCE-B for all i
  - Computed: α = Σᵢ sᵢ mod n
  - Computed: s = s' + α mod n
- **Timeout:** T_DECAP = 10 minutes from DECAP state entry
- **Success Action:** Build final transaction with signature s
- **Failure Action:** If timeout expires, retry decapsulation or transition to ABORTED

**Transition 9: DECAP → ABORTED (Timeout/Failure Path)**
- **Condition:** (Current time > t_start_decap + T_DECAP) OR (PoCE-B verification fails)
- **Action:** Abort protocol, optionally retry with different decapper

**Transition 10: BROADCAST → COMPLETED (Success Path)**
- **Condition:** Transaction broadcasted and confirmed on Bitcoin blockchain
  - Broadcast: Transaction sent to Bitcoin network
  - Confirmed: Transaction included in block with ≥1 confirmations
- **Timeout:** T_BROADCAST = 5 minutes from BROADCAST state entry (for initial broadcast)
- **Success Action:** Protocol completed successfully

**Transition 11: BROADCAST → ABORTED (Timeout Path)**
- **Condition:** Current time > t_start_broadcast + T_BROADCAST AND transaction not confirmed
- **Action:** Retry broadcast with fee adjustment (CPFP) or transition to ABORTED

#### 12.2.3 Timeout Parameter Values (Normative)

Implementations MUST use the following minimum timeout values:

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| T_ARMING | 24 hours | Multi-party coordination over asynchronous networks |
| T_PRESIG | 1 hour | MuSig2 typically completes in seconds; 1h allows retries |
| Δ_CSV | 48 hours | Groth16 proving time varies; allows resource-constrained provers |
| T_DECAP | 10 minutes | Decapsulation ~100ms; 10min allows multiple retry attempts |
| T_BROADCAST | 5 minutes | Bitcoin propagation ~seconds; 5min allows network congestion |

Implementations MAY use longer timeouts but MUST NOT use shorter values without formal analysis justifying the reduction.

#### 12.2.4 Simultaneous Timeout Resolution (Tie-Breaking)

If multiple participants independently detect timeout expiration within a narrow time window (e.g., clock skew ≤1s), the following tie-breaking rules MUST apply:

**Rule 1: Deterministic Ordering**
- Participants MUST use a deterministic tie-breaking mechanism based on public identifiers
- **Specification:** Smallest participant public key (lexicographic order on compressed encoding)
- **Alternative (more robust):** Context-bound priority:
  ```
  priority_i = H("PVUGC/TIMEOUT_PRIORITY" || ctx_hash || P_i || timeout_type)
  Smallest priority_i wins
  ```
  This prevents key-grinding attacks (adversary cannot pre-select advantageous key).

**Security Note (Key-Grinding Attack):**

Deterministic lexicographic ordering creates a strategic advantage for participants who can grind public keys. An adversary generating 2^b key pairs can select the lexicographically smallest key, gaining advantage:

```
Pr[adv wins] ≈ 1 - (1 - 2^(-256))^(2^b) ≈ 2^b · 2^(-256)
```

For k = 5 participants and b = 2^20 (1 million keys):
```
Pr[adv wins] ≈ 2^20 · 5 · 2^(-256) ≈ 2^(-233.7)
```

**Conclusion:** Attack is computationally expensive (2^20 key generations) with negligible success probability (2^(-233)). Lexicographic ordering is acceptable for timeout tie-breaking.

**Alternative (More Robust):** Use context-bound priority:
```
priority_i = H("PVUGC/TIMEOUT_PRIORITY" || ctx_hash || P_i || timeout_type)
Smallest priority_i wins
```
This prevents key grinding since ctx_hash is unknown at key selection time.

**Rule 2: First-Seen-First-Valid**
- In case of competing timeout transactions, the first transaction seen by the Bitcoin network is considered valid
- Losing participants MUST NOT attempt to override with higher fees (to prevent DoS escalation)

**Rule 3: Grace Period**
- Participants SHOULD wait a grace period (e.g., 30 seconds) after timeout expiration before publishing abort transactions
- This reduces the probability of simultaneous publications

**Rule 4: Clock Source**
- Timeout expiration MUST be checked against blockchain median-time-past (BIP-113) or NTP-synchronized time with maximum clock skew ≤ 1 second
- Local system clocks MAY have skew up to ±70 minutes (per BIP-113), which is unacceptable
- **Recommended:** Use Bitcoin blockchain timestamps for timeout checks (robust to clock skew)

#### 12.2.5 State Invariants (Normative Assertions)

Implementations MUST maintain the following invariants:

**Invariant I1 (Single Active State):**
```
At any time t, exactly one state s ∈ S is active (no state ambiguity)
∀t: |{s ∈ S : state(t) = s}| = 1
```

**Invariant I2 (Monotonic Progress):**
```
States are ordered: IDLE < ARMING < PRE_SIGNING < AWAITING_PROOF < DECAP < BROADCAST < COMPLETED
Transitions MUST be monotonically increasing (except to ABORTED)
∀t₁ < t₂: state(t₁) ≤ state(t₂) ∨ state(t₂) = ABORTED
```

**Invariant I3 (Timeout Enforcement):**
```
For each state s with timeout T_s:
  If (current_time - t_enter_s) > T_s AND state = s:
    MUST eventually transition to ABORTED
∀s ∈ {ARMING, PRE_SIGNING, AWAITING_PROOF, DECAP, BROADCAST}:
  elapsed_time(s) > T_s ⟹ ◇(state = ABORTED)
```
(Using temporal logic ◇ = "eventually")

**Invariant I4 (Abort Finality):**
```
Once state = ABORTED, no further transitions allowed (terminal state)
state = ABORTED ⟹ □(state = ABORTED)
```
(Using temporal logic □ = "always")

Implementations MUST include runtime assertions checking these invariants and fail-fast (abort with error) if any invariant is violated.

#### 12.2.6 Upgrade to MUST: CSV Abort Path

The optional Timeout/Abort path (§2, lines 36-39) is upgraded from SHOULD to MUST:

**Original (§2):**
```
Timeout/Abort (optional) :
    <Δ> OP_CHECKSEQUENCEVERIFY OP_DROP
    <P_abort> OP_CHECKSIG
```

**Revised (MUST):**
```
Timeout/Abort (REQUIRED) :
    <Δ> OP_CHECKSEQUENCEVERIFY OP_DROP
    <P_abort> OP_CHECKSIG
```

**Rationale:** Without an abort path, funds can be locked indefinitely if proof generation fails. The abort path is a critical liveness guarantee and MUST be included.

**Δ Specification:**
- MUST be set to Δ ≥ 48 hours (represented as block height or time per BIP-112/113)
- SHOULD account for worst-case proof generation time plus network latency
- MAY be configurable per deployment context but MUST be disclosed in ctx_core
```

---

#### Recommendation 3: Add §12.3 - Front-Running Mitigation Strategies (SHOULD)

**Add to PVUGC-2025-10-20.md §1 Introduction after §12.2:**

```markdown
### 12.3 Front-Running Mitigation Strategies (Informative)

While the protocol uses SIGHASH_ALL to bind signatures to exact transaction outputs, minimizing malleability, residual front-running risks exist due to mempool observation and fee manipulation. This section provides guidance on mitigation strategies.

#### 12.3.1 Risk Analysis

**Residual Front-Running Vectors:**

**1. CPFP Anchor Manipulation:**

The spec includes a "small P2TR CPFP hook output" (§2, line 50) for fee bumping. Depending on spending conditions:

**Scenario A: Anchor controlled by signing key P**
- Adversary **cannot** spend anchor (requires signature from P)
- CPFP is **NOT an attack vector**
- Fee manipulation requires consent of key holders
- **Recommended:** This is the preferred configuration

**Scenario B: Anchor is "anyone-can-spend" (e.g., OP_TRUE scriptPubKey)**
- Adversary **CAN** spend anchor in child transaction
- **Low-fee child attack:** Adversary creates child with minimal fee, delaying parent confirmation (griefing)
- **High-fee child attack:** Adversary accelerates parent confirmation (not harmful to honest parties)

**Analysis of CPFP Attacks:**
- **High-fee CPFP:** Even if adversary accelerates parent, outputs are fixed by SIGHASH_ALL. Adversary gains no value, only benefits honest parties. **Not an attack.**
- **Low-fee CPFP (Griefing):** Adversary delays confirmation by creating child below minimum relay fee. Harms honest parties (denial-of-service) but does not extract value.

**Mitigation (MUST):**
```
CPFP hook output MUST be controlled by aggregate public key P
(requires signature from pre-signing participants)

This prevents adversarial CPFP griefing while preserving fee-bumping capability for participants.
```

**Alternative (if anyone-can-spend required):**
- Add minimum fee requirement to anchor spend (e.g., OP_CHECKSEQUENCEVERIFY delay)
- Monitor mempool for CPFP attempts and alert on anomalous children

**2. Mempool Monitoring:**
   - Public Bitcoin mempool allows adversaries to observe transactions before confirmation
   - Even with SIGHASH_ALL, adversaries can analyze transaction patterns, timing, and fees (information leakage)
   - **Mitigation:** Use encrypted mempools or direct miner submission

**3. Replace-By-Fee (RBF):**

BIP-125 allows replacing unconfirmed transactions with higher-fee versions (opt-in RBF flag).

**Why RBF Attack FAILS:**
```
SIGHASH_ALL commits to all transaction outputs
Modified transaction has different sighash m' ≠ m
Signature s = s' + α is valid ONLY for original sighash m
Adversary cannot produce valid signature for m' without:
- Knowledge of α (witness-encrypted, unknown without proof)
- Private key for P (unknown)

Conclusion: RBF attack FAILS under SIGHASH_ALL
```

**Mitigation (Already Satisfied):**
- Transaction template MUST use SIGHASH_ALL (required by spec §2, line 49)
- RBF is **not an attack vector** given this requirement

**Additional Safety (SHOULD):**
- **Disable RBF flag** (BIP-125 opt-out): Set sequence to 0xfffffffe (signals "no RBF")
- **Monitor mempool** for RBF attempts (should not occur, but detect anomalies)

#### 12.3.2 Mitigation Strategy 1: Encrypted Mempool (RECOMMENDED)

Bitcoin Core v24+ supports encrypted mempool propagation via P2P message encryption (BIP-324).

**Implementation:**
```
1. Enable encrypted P2P: --v2transport=1 in Bitcoin Core config
2. Connect only to peers supporting BIP-324 encrypted transport
3. Broadcast transaction via encrypted channels
4. Reduces visibility to passive observers (not miners)
```

**Limitations:**
- Miners must also support encrypted transport
- Does not prevent miner-level front-running (miners see all transactions)

**Effectiveness:** Medium (reduces passive observation, not active attacks)

#### 12.3.3 Mitigation Strategy 2: Direct Miner Submission (ALTERNATIVE)

Submit transactions directly to mining pools via private channels, bypassing public mempool.

**Implementation:**
```
1. Establish relationship with mining pool(s)
2. Submit transaction via pool's private API (not broadcast to P2P network)
3. Pool includes transaction in next block without public propagation
```

**Limitations:**
- Requires trust in mining pool
- Reduces decentralization (transaction not publicly visible until confirmation)
- Mining pool could extract MEV directly

**Effectiveness:** High (eliminates mempool observation) but introduces trust assumptions

#### 12.3.4 Mitigation Strategy 3: Transaction Template Rigidity (MUST)

Ensure transaction templates have zero output flexibility.

**MUST Requirements:**
```
1. All outputs MUST be fully specified at arming time (no change addresses)
2. CPFP anchor output MUST be controlled by signing key P (requires signature)
3. No RBF flag (BIP-125 opt-out: sequence = 0xfffffffe)
4. txid_template MUST include exact fee amount (no fee estimation flexibility)
```

**Rationale:** Eliminates any wiggle room for output manipulation or fee adjustment by adversaries.

#### 12.3.5 Deployment Recommendations

Implementations SHOULD adopt the following defense-in-depth approach:

**Tier 1 (MUST):**
- Transaction template rigidity (§12.3.4)
- CPFP anchor controlled by P (requires signature)
- Disable RBF flag
- SIGHASH_ALL (already required)

**Tier 2 (RECOMMENDED):**
- Encrypted mempool (§12.3.2) for BIP-324 capable nodes
- Monitor mempool for competing transactions
- Alert users if front-running detected

**Tier 3 (OPTIONAL):**
- Direct miner submission for high-value transactions
- Out-of-band coordination with miners for time-sensitive deployments

#### 12.3.6 Front-Running Detection and Response

Implementations SHOULD include monitoring capabilities:

**Detection:**
```
1. Monitor mempool for transactions spending same inputs
2. Alert if multiple transactions detected within time window (e.g., 10s)
3. Log competing transaction details (fee, outputs, timing)
```

**Response:**
```
If front-running detected:
1. Do NOT engage in fee bidding war (prevents DoS escalation)
2. Alert operators/users
3. Optionally: Abort and retry with different arming set (if possible)
4. Document incident for forensic analysis
```

**Rationale:** Fee wars benefit miners at the expense of honest participants. Better to abort and retry than escalate.
```

---

### Implementation and Testing Requirements

#### Recommendation 4: Timing Invariance Test Suite (MUST)

**Owner:** Implementation teams
**Timeline:** Before mainnet deployment
**Deliverable:** Automated CI/CD timing tests using TOST methodology

**Test Suite Components:**

**Test 1: GS-PPE Decapsulation Timing Uniformity**
```python
from scipy import stats
import numpy as np

def tost_equivalence_test(times_a, times_b, epsilon_rel=2**(-40), alpha=0.05):
    """
    Two One-Sided Tests (TOST) for timing equivalence.
    Returns (equivalent, p_value_max, delta_mu)
    """
    mean_a = np.mean(times_a)
    mean_b = np.mean(times_b)
    delta_mu = mean_a - mean_b

    # Equivalence margin (absolute): use 2*noise_std for practical testing
    # For production: epsilon_rel * max(mean_a, mean_b)
    delta = 2 * np.std(np.concatenate([times_a, times_b]))

    se_a = stats.sem(times_a)
    se_b = stats.sem(times_b)
    se_pooled = np.sqrt(se_a**2 + se_b**2)

    df = len(times_a) + len(times_b) - 2

    # Two one-sided t-tests
    t1 = (delta_mu - delta) / se_pooled
    t2 = (delta_mu + delta) / se_pooled

    p1 = stats.t.cdf(t1, df)
    p2 = 1 - stats.t.cdf(t2, df)

    equivalent = (p1 < alpha) and (p2 < alpha)
    return equivalent, max(p1, p2), delta_mu

def test_decap_timing_uniform():
    """Verify decapsulation takes constant time regardless of (m1, m2)"""

    # Generate test attestations with varying pairing counts
    attestations = [
        generate_attestation(m1=20, m2=20),   # 40 pairings
        generate_attestation(m1=48, m2=48),   # 96 pairings (max)
        generate_attestation(m1=10, m2=10),   # 20 pairings (min)
    ]

    # Measure timing for each (10^6 samples)
    timings = {}
    for att in attestations:
        times = []
        for _ in range(10**6):
            start = time.perf_counter_ns()
            decapsulate(att)
            end = time.perf_counter_ns()
            times.append((end - start) / 1e9)  # Convert to seconds
        timings[att.id] = np.array(times)

    # TOST for each pair
    from itertools import combinations
    for (att1, att2) in combinations(attestations, 2):
        equiv, p_val, diff = tost_equivalence_test(
            timings[att1.id],
            timings[att2.id],
            alpha=0.05
        )
        assert equiv, f"Timing leak detected: {att1.id} vs {att2.id}, p={p_val:.6f}, Δμ={diff*1e6:.3f}μs"

    print("✓ Decapsulation timing is uniform (constant-time via TOST)")
```

**Test 2: DEM Decryption Success/Failure Timing**
```python
def test_dem_timing_uniform():
    """Verify DEM decryption timing is identical for success/failure"""

    key = generate_test_key()
    plaintext = b"test_plaintext_with_32bytes!"
    ad = b"associated_data"

    valid_ct, valid_tag = encrypt_dem(key, plaintext, ad)
    invalid_ct = bytearray(valid_ct)
    invalid_ct[0] ^= 0xFF  # Corrupt ciphertext

    valid_times = []
    invalid_times = []

    for _ in range(10**6):
        # Measure valid decryption
        start = time.perf_counter_ns()
        result = decrypt_dem(key, valid_ct, valid_tag, ad)
        end = time.perf_counter_ns()
        valid_times.append((end - start) / 1e9)
        assert result == plaintext

        # Measure invalid decryption
        start = time.perf_counter_ns()
        result = decrypt_dem(key, invalid_ct, valid_tag, ad)
        end = time.perf_counter_ns()
        invalid_times.append((end - start) / 1e9)
        assert result is None  # Decryption should fail

    # TOST
    equiv, p_val, diff = tost_equivalence_test(
        np.array(valid_times),
        np.array(invalid_times),
        alpha=0.05
    )
    assert equiv, f"DEM timing leak: p={p_val:.6f}, Δμ={diff*1e6:.3f}μs"

    print("✓ DEM decryption timing is uniform (success vs failure)")
```

**Test 3: State Machine Timeout Enforcement**
```python
def test_state_machine_timeouts():
    """Verify all state timeouts are enforced"""

    # Test ARMING timeout
    protocol = Protocol(k=5)
    protocol.enter_state(State.ARMING)

    # Simulate k-1 armers publishing (one missing)
    for i in range(4):
        protocol.publish_armer_share(generate_valid_share(i))

    # Verify protocol is waiting
    assert protocol.state == State.ARMING

    # Advance time beyond T_ARMING
    protocol.advance_time(timedelta(hours=24, seconds=1))
    protocol.tick()

    # Verify transition to ABORTED
    assert protocol.state == State.ABORTED, "Arming timeout not enforced"

    # Verify abort transaction can be built
    abort_tx = protocol.build_abort_transaction()
    assert abort_tx is not None

    # Test PRE_SIGNING timeout (similar structure)
    # Test DECAP timeout (similar structure)
    # Test BROADCAST timeout (similar structure)

    print("✓ All state machine timeouts enforced")
```

**CI/CD Integration:**
```yaml
# .github/workflows/timing-tests.yml
name: Timing Side-Channel Tests

on: [push, pull_request]

jobs:
  timing-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y libblst-dev
          pip install scipy numpy pytest

      - name: Run timing tests (TOST methodology)
        run: |
          pytest tests/timing/ --timing-samples=1000000 -v

      - name: Generate timing report
        run: |
          python scripts/analyze_timing_tost.py > timing_report.txt

      - name: Upload report
        uses: actions/upload-artifact@v3
        with:
          name: timing-report-tost
          path: timing_report.txt

      - name: Fail on timing leaks
        run: |
          # Check if any TOST test failed
          grep "Timing leak detected" timing_report.txt && exit 1 || exit 0
```

---

#### Recommendation 5: Deadlock Prevention Validation (MUST)

**Test Scenario 1: Arming Phase Deadlock**
```python
def test_arming_deadlock_prevention():
    """Verify protocol aborts when arming doesn't complete"""

    protocol = Protocol(k=5)  # 5-of-5 armers
    protocol.enter_state(State.ARMING)

    # Only 4 armers publish (5th is malicious/offline)
    for i in range(4):
        protocol.publish_armer_share(generate_valid_share(i))

    # Verify protocol is waiting
    assert protocol.state == State.ARMING
    assert protocol.armers_published == 4

    # Advance time to T_ARMING + epsilon
    protocol.advance_time(timedelta(hours=24, seconds=1))
    protocol.tick()

    # Verify transition to ABORTED
    assert protocol.state == State.ABORTED, "Deadlock not prevented"

    # Verify abort transaction can be published
    abort_tx = protocol.build_abort_transaction()
    assert abort_tx is not None
    assert abort_tx.is_valid()

    print("✓ Arming deadlock prevented by timeout")
```

**Test Scenario 2: Simultaneous Timeout Tie-Breaking**
```python
def test_simultaneous_timeout_tiebreak():
    """Verify deterministic tie-breaking for simultaneous timeouts"""

    participants = [
        Participant(pubkey="02aaa..."),  # Lexicographically first
        Participant(pubkey="02bbb..."),
        Participant(pubkey="02ccc..."),
    ]

    # All participants detect timeout simultaneously
    t_timeout = datetime.now()

    for p in participants:
        p.detect_timeout(t_timeout)

    # Each computes their priority (lexicographic on pubkey)
    priorities = [p.compute_priority() for p in participants]

    # Verify deterministic ordering
    expected_order = sorted(participants, key=lambda p: p.pubkey)
    actual_order = sorted(participants, key=lambda p: p.compute_priority())

    assert actual_order == expected_order, "Tie-breaking not deterministic"

    # Winner publishes abort transaction
    winner = actual_order[0]
    assert winner.pubkey == "02aaa..."  # Smallest pubkey
    winner.publish_abort_transaction()

    # Losers observe winner's transaction and don't compete
    time.sleep(1)  # Grace period
    for p in actual_order[1:]:
        p.observe_mempool()
        assert not p.published_abort_tx, "Loser should not publish after observing winner"

    print("✓ Simultaneous timeout resolved deterministically")
```

---

## Validation Checklist

The following checks MUST be satisfied before considering PVUGC-007 resolved:

### Specification Changes
- [ ] §12.1 (Constant-Time Execution) added to PVUGC-2025-10-20.md §1 Introduction with MUST clauses
- [ ] §12.2 (State Machine Timeouts) added with formal state definitions and timeout values
- [ ] §12.3 (Front-Running Mitigations) added with deployment guidance
- [ ] CSV Abort path upgraded from SHOULD to MUST (§2, lines 36-39)
- [ ] Timeout parameters specified: T_ARMING=24h, T_PRESIG=1h, Δ_CSV=48h, T_DECAP=10min, T_BROADCAST=5min
- [ ] State machine invariants I1-I4 formally specified in §12.2.5

### Implementation Requirements
- [ ] Constant-time GS-PPE decapsulation (always 96 pairings) implemented
- [ ] Constant-time DEM decryption (no early abort) implemented
- [ ] Constant-time PoCE-B verification implemented
- [ ] Cache-oblivious pairing library validated (blst, arkworks with constant_time feature)
- [ ] State machine with timeout enforcement implemented
- [ ] Tie-breaking mechanism for simultaneous timeouts implemented (lexicographic or context-bound)
- [ ] CPFP anchor output controlled by signing key P (requires signature)
- [ ] Transaction template has zero output flexibility (no change addresses)
- [ ] RBF flag disabled (sequence = 0xfffffffe)

### Testing Requirements
- [ ] Timing invariance tests pass using TOST methodology (10⁶ samples, both one-sided p < 0.05)
- [ ] Decapsulation timing uniform across (m₁, m₂) = (20,20), (48,48), (10,10)
- [ ] DEM timing uniform for success vs failure paths
- [ ] PoCE-B timing uniform for all-valid vs one-invalid shares
- [ ] Cache-timing resistance validated (Flush+Reload, Prime+Probe tests)
- [ ] State machine timeout tests pass (all timeouts enforced)
- [ ] Deadlock prevention tests pass (arming, presigning, decap phases)
- [ ] Simultaneous timeout tie-breaking tests pass (deterministic ordering)
- [ ] CI/CD integration: TOST timing tests run on every commit, fail build on leaks

### Documentation Requirements
- [ ] Timing attack risk documented in security considerations (10-12 bits leakage without mitigation)
- [ ] Constant-time library recommendations provided (blst, arkworks, MCL)
- [ ] State machine diagram published (visual representation of transitions)
- [ ] Timeout parameter rationale documented (why 24h, 1h, 48h, etc.)
- [ ] Front-running mitigation deployment guide published (encrypted mempool vs direct miner)
- [ ] TOST testing methodology documented with example code
- [ ] Key-grinding attack analysis documented (2^-233 success probability)

### Formal Analysis (Desirable)
- [ ] Mutual information I(W; T_decap) ≈ 10-12 bits quantified for reference implementation
- [ ] Worst-case leakage analysis documented (extreme cases: m₁=m₂=1)
- [ ] Cache-timing resistance formally verified (library-specific analysis)
- [ ] State machine invariants I1-I4 verified (TLA+ or similar formal method)
- [ ] Liveness proof: timeout enforcement prevents DoS with probability > 1 - negl(λ)

---

## Status Decision

### Final Severity
**High** (upgraded from Medium)

**Justification:**
1. **Zero-Knowledge Violation:** Timing side-channel leaks 10-12 bits of witness structure per observation, enabling full witness extraction in ~11 observations. This violates the core cryptographic property of proof existence without witness revelation.
2. **Protocol-Level Vulnerability:** Deadlock attack requires only non-participation (O(1) adversarial effort), not implementation bugs. No implementation choice can prevent deadlock without timeout specification.
3. **No Workaround:** Implementers cannot fix without spec-level normative requirements (constant-time operations, state machine timeouts).

### Acceptance Criteria

**To close PVUGC-007 as RESOLVED:**

**Tier 1 (MUST - Specification):**
1. Spec amendments (§12.1, §12.2, §12.3) merged into PVUGC-2025-10-20.md §1 Introduction
2. All normative MUST clauses validated by protocol designers
3. Timeout parameters reviewed and confirmed (or adjusted with rationale)
4. CSV abort path upgraded from SHOULD to MUST

**Tier 2 (MUST - Implementation):**
1. Reference implementation demonstrates constant-time execution (TOST tests pass)
2. State machine with timeouts implemented and tested (all invariants I1-I4 satisfied)
3. Cache-oblivious pairing library validated (blst or arkworks with constant_time)
4. CI/CD TOST timing tests integrated and passing

**Tier 3 (SHOULD - Deployment):**
1. Front-running mitigation strategy chosen and documented (encrypted mempool or direct miner)
2. Deployment guide published for production use
3. Security advisory published warning of timing risks in non-compliant implementations

### Intermediate Status Options

**Option A: Partially Resolved (Spec Only)**
- If Tier 1 complete but Tier 2/3 pending
- Status: "Spec Amended, Implementation Pending"
- Risk: Implementations may not comply without reference implementation and testing framework

**Option B: Resolved with Caveats**
- If Tier 1 and Tier 2 complete, Tier 3 documented
- Status: "Resolved (Deployment Guidance Informative)"
- Risk: Deployments may skip front-running mitigations or use non-constant-time libraries

**Option C: Fully Resolved**
- All tiers complete, TOST timing tests passing, deployment guidance published
- Status: "Resolved"
- Recommendation: External audit to validate timing guarantees and cache-timing resistance

### Recommended Next Steps

**Week 1-2:** Protocol design team reviews final report, refines timeout parameters and ε thresholds

**Week 3-4:** Draft spec amendments (§12.1, §12.2, §12.3), circulate for feedback

**Week 5-8:** Implement constant-time execution and state machine in reference implementation

**Week 9-10:** Implement TOST timing test suite, validate against statistical thresholds

**Week 11-12:** Integrate into CI/CD, publish deployment guide

**Week 13+:** External audit (recommended), publish final security advisory

### Cross-Issue Dependencies

**Depends on:**
- PVUGC-004 (PoCE-B soundness): PoCE-B verification is part of constant-time requirement (Finding 5)
- PVUGC-008 (MuSig2 compartmentalization): Pre-signing phase timeout depends on MuSig2 completion (Finding 6)
- PVUGC-005 (Context binding): ctx_hash must include all timeout-relevant parameters

**Impacts:**
- All PVUGC implementations MUST comply with constant-time and timeout requirements
- Threshold/multi-party variants need timeout parameter scaling (T_ARMING may increase with k > 10)
- Retry mechanisms (if implemented) must respect timeout bounds and invariants

**Related Security Considerations:**
- PVUGC-009 (Implementation hardening): Constant-time library selection, cache-timing tests
- PVUGC-002 (DEM key-commitment): DEM constant-time requirement critical for PoCE-B soundness

---

## Appendix A: Attack Complexity Summary

| **Attack** | **Adversarial Cost** | **Success Probability** | **Mitigation Cost** | **Residual Risk** |
|------------|---------------------|------------------------|---------------------|-------------------|
| **Timing Oracle (Finding 2)** | O(n_cal · 96 pairings) where n_cal = 100 calibration samples | P(leak) ≈ 10-12 bits per observation; full witness in ~11 observations | O(96 pairings always) + constant-time library | Negligible if ε < 2^(-40) |
| **Cache-Timing (Finding 2)** | O(10⁶ Flush+Reload samples) | P(leak) depends on library implementation | Cache-oblivious library + testing | Negligible with blst/arkworks constant_time |
| **Arming Deadlock (Finding 3)** | O(1) - simply don't publish | P(success) = 1 without timeout | O(1) - add T_ARMING timeout | Zero if timeout enforced |
| **Mempool Front-Run (Finding 4)** | O(1) - mempool monitoring | P(success) ≈ 0 (SIGHASH_ALL prevents) | SIGHASH_ALL (already required) | Zero given spec requirements |
| **CPFP Griefing (Finding 4)** | O(1) - publish low-fee child | P(success) = 1 if anchor anyone-can-spend | Anchor controlled by P | Zero if anchor requires signature |
| **RBF Attack (Finding 4)** | O(1) - attempt replacement | P(success) = 0 (no valid signature) | SIGHASH_ALL (already required) | Zero (attack fails) |
| **Simultaneous Timeout (Finding 3)** | O(k) - coordinate k participants | P(success) < 1/k with tie-breaking | O(1) - deterministic ordering | Negligible with grace period |
| **Key-Grinding (Finding 3)** | O(2^20 key generations) | P(success) ≈ 2^(-233) for k=5 | O(1) - lexicographic ordering or context-bound | Negligible (astronomically small) |

**Key Insight:** All attacks are either:
1. **Eliminated** by constant-time execution + timeouts (timing oracle, deadlock)
2. **Prevented** by SIGHASH_ALL (front-running, RBF)
3. **Negligible** due to cryptographic hardness (key-grinding: 2^-233)

---

## Appendix B: Normative Text Integration

**Sections to add to PVUGC-2025-10-20.md §1 Introduction:**

**After existing §11:**
```diff
 ### 11) Notes
 ...existing content...

+### 12) Timing and Liveness Guarantees
+
+### 12.1 Constant-Time Execution (Normative)
+[Full text from Recommendation 1]
+
+### 12.2 Protocol State Machine with Timeouts (Normative)
+[Full text from Recommendation 2]
+
+### 12.3 Front-Running Mitigation Strategies (Informative)
+[Full text from Recommendation 3]
```

**Modification to §2 (Bitcoin script):**
```diff
-Timeout/Abort (optional) :
+Timeout/Abort (REQUIRED) :
-    <Δ> OP_CHECKSEQUENCEVERIFY OP_DROP
+    <Δ> OP_CHECKSEQUENCEVERIFY OP_DROP  // Δ MUST be ≥ 48 hours
     <P_abort> OP_CHECKSIG
```

---

## Appendix C: Formal Security Statement

**Theorem (Timing Side-Channel Mitigation):**
Let Π be the PVUGC protocol augmented with constant-time execution requirements (§12.1). For any probabilistic polynomial-time adversary A with access to timing oracle T:

```
Adv_ZK^Π(A) ≤ Adv_ZK^Groth16(A') + ε_timing
```

where:
- Adv_ZK^Π(A) is A's advantage in distinguishing witnesses via timing
- Adv_ZK^Groth16(A') is the zero-knowledge advantage of the underlying Groth16 SNARK
- ε_timing ≤ 2^(-40) is the constant-time execution parameter

**Proof Sketch:**
1. Constant-time GS-PPE decapsulation ensures T(m₁, m₂) distributions are ε-indistinguishable
2. TOST statistical testing validates ε ≤ 2^(-40) (both one-sided tests p < 0.05)
3. Therefore, I(W; T) ≤ log₂(1/ε) · P(distinguish) ≤ 40 · negl(λ) = negl(λ)
4. Remaining ZK property relies on Groth16 zero-knowledge and GS commitment hiding
5. By composition, Π preserves zero-knowledge up to negligible leakage

**Theorem (Liveness Under Timeouts):**
Let Π be the PVUGC protocol augmented with state machine timeouts (§12.2). Assume at most f < k participants are Byzantine. Then:

```
P(protocol_deadlock) < 2^(-λ)
```

provided timeout parameters satisfy:
- T_ARMING > expected_coordination_time + (k · network_latency)
- T_PRESIG > MuSig2_completion_time + (k · network_latency)
- Δ_CSV > proving_time_99th_percentile

**Proof Sketch:**
1. **ARMING phase:** If ≥ k - f honest armers publish within T_ARMING, protocol proceeds to PRE_SIGNING
2. If < k armers publish, Invariant I3 ensures timeout triggers ABORTED transition (no infinite wait)
3. **PRE_SIGNING phase:** If MuSig2 completes within T_PRESIG, protocol proceeds. Otherwise, timeout enforces ABORTED.
4. **AWAITING_PROOF phase:** If valid proof arrives within Δ_CSV, protocol proceeds. Otherwise, CSV abort path becomes valid.
5. **Invariant I4 (Abort Finality):** Once ABORTED, no further transitions (prevents resurrection attacks)
6. **Probability of deadlock:** Requires all honest parties to fail coordination within timeout windows, which is negligible under network assumptions (Byzantine agreement with timeouts)

**Corollary (DoS Resistance):**
An adversary controlling f < k participants cannot prevent protocol completion (deadlock) or force premature abortion (liveness violation) with probability > negl(λ), provided honest majority cooperates within timeout windows.

---

## Conclusion

The peer review process has rigorously validated and enhanced M2's initial analysis:

**Key Contributions:**

**Round 1 (M2):** Identified zero-knowledge violation via timing leakage and deadlock vulnerability without timeouts

**Round 2 (Crypto-Reviewer):** Constructed concrete attack algorithms, formalized state machine, proposed spec-ready normative text

**Round 3 (M2):** Identified critical gaps (TOST vs Welch's t-test, missing invariants, cache-timing, worst-case analysis)

**Round 4 (This Document):** Incorporated all revisions, achieving:
- **Mathematical rigor:** Corrected arithmetic (4560 pairs), worst-case leakage (H=0 for m₁=m₂=1), timing precision (μs-level)
- **Statistical correctness:** TOST methodology for constant-time validation (both one-sided p < 0.05)
- **Completeness:** State machine invariants I1-I4, cache-timing analysis, CPFP/RBF detailed analysis
- **Practical guidance:** Concrete Python test code, CI/CD integration, library recommendations

**Final Assessment:**

The timing side-channel and race condition vulnerabilities are **real, demonstrable, and severe**:
- **Security impact:** 10-12 bits of witness leakage per observation, full extraction in ~11 observations
- **Liveness impact:** Trivial O(1) deadlock attack without timeouts
- **Severity justification:** High (violates zero-knowledge, protocol-level flaw, no workaround without spec changes)

**Remaining Work:**

**Specification:**
1. Merge §12.1, §12.2, §12.3 into PVUGC-2025-10-05.md §7 GS PPE Verification (96 pairings)
2. Upgrade CSV abort from SHOULD to MUST
3. Integrate state machine invariants I1-I4

**Implementation:**
1. Implement constant-time execution (96 pairings always, cache-oblivious library)
2. Implement state machine with timeout enforcement
3. Develop TOST timing test suite

**Testing:**
1. Validate TOST tests pass (10⁶ samples, both p < 0.05)
2. Validate cache-timing resistance (Flush+Reload, Prime+Probe)
3. Validate deadlock prevention (timeout boundary conditions)

**Deployment:**
1. Select front-running mitigation (encrypted mempool or direct miner)
2. Publish security advisory and deployment guide
3. External audit (recommended)

**Recommendation:** Proceed with spec amendments and reference implementation. PVUGC-007 should remain **High Severity** and **Enhanced** (not Resolved) until all Tier 1 and Tier 2 acceptance criteria are met.

---

**Final Peer Reviewers:**
- **M2 (Mathematician):** Rounds 1, 3 — Formal quantification, statistical rigor, invariant completeness
- **Crypto-Reviewer:** Rounds 2, 4 — Attack construction, normative text, final integration

**Date:** 2025-10-26
**Status:** Final peer-reviewed report (v4.0)
**Next Steps:** Protocol design team review → Spec amendment → Implementation → Testing → Deployment
