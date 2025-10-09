# PVUGC-007: Timing/Race Conditions

**Flaw Code:** PVUGC-007
**Severity:** 🟡 **MEDIUM**
**Status:** 🔓 Open (Implementation-dependent)
**Date Identified:** 2025-10-07 (v1.0)
**Last Updated:** 2025-10-07 (v2.0)

---

## Component
Multi-stage protocol flow timing and synchronization

## Location
- **v1.0:** §9 (no explicit timing/timeout specifications)
- **v2.0:** §9 (Abort path with CSV mentioned, but main path still underspecified)

---

## Description

The PVUGC protocol involves multiple sequential and parallel phases:
1. **Arming phase:** Multiple armers publish shares
2. **Pre-signature phase:** MuSig2 protocol for adaptor pre-signature
3. **Proof phase:** Prover generates Groth16+GS proof
4. **Decapsulation phase:** Decapper(s) derive α and finalize signature
5. **Broadcast phase:** Transaction broadcasted to Bitcoin network

**Problem:** The protocol lacks explicit specifications for:
- Phase transition timing and ordering
- Timeout mechanisms (except optional CSV in Abort path)
- Synchronization between concurrent participants
- Race condition handling (multiple decappers)
- Timing side-channel mitigation

**Impact:** Potential for race conditions, front-running, timing side-channels, and liveness failures.

---

## What Changed in v2.0

### 🔧 Partial Improvements

**v2.0 mentions (but doesn't mandate):**

**1. Abort Path with CSV Timeout (§9, optional)**
```
SHOULD: Enable Timeout/Abort path with CSV (CheckSequenceVerify)
- Allows recovery if proof never arrives
- Δ parameter for proving + network latency
```

**What this provides:**
- Optional escape hatch for locked funds
- CSV enforces minimum time before abort spend valid

**What's still missing:**
- No timeout for main (happy) path phases
- CSV is SHOULD, not MUST
- No specification of Δ value

**2. Constant-Time Operations Mentioned (§7, line 292 equivalent)**
```
Generic mention: Implementations should use constant-time pairing operations
```

**What this provides:**
- Awareness of timing side-channel risk

**What's still missing:**
- No normative requirements for constant-time
- No specification of which operations must be constant-time
- No testing/validation requirements

### 🔓 What Remains Unaddressed

**v1.0 and v2.0 both lack:**
- ❌ Phase transition state machine
- ❌ Explicit timeouts for arming, pre-signing, proof generation
- ❌ Synchronization guarantees between participants
- ❌ Race condition resolution (multiple decappers)
- ❌ Front-running mitigation strategies
- ❌ Normative constant-time requirements
- ❌ Timing side-channel test requirements

---

## Security Impact

**Consequence:** Various timing-related issues that can affect:
- **Security:** Timing side-channels leak information
- **Liveness:** Deadlocks or indefinite waiting
- **Fairness:** Front-running allows value extraction
- **Economics:** MEV (Miner Extractable Value) opportunities

**Impact severity:** Medium (mostly affects liveness and fairness, not core cryptographic security)

---

## Specific Concerns

### 1. Phase Ordering and Transitions (Underspecified)

**Problem:** No formal state machine for protocol phases.

**Questions:**
- Can pre-signing start before all armers publish?
- Can proof be submitted before pre-signature completes?
- What if proof arrives during arming phase?
- How to handle out-of-order message delivery?

**Current state (v2.0):**
- Implicit ordering from spec narrative
- No explicit state machine
- No enforcement mechanism

**Risks:**
```
Scenario 1: Premature pre-signature
- Pre-signing starts with incomplete arming set
- Missing armers publish later
- Do we restart pre-signing? Or reject late armers?

Scenario 2: Early proof submission
- Prover submits proof before pre-signature completes
- Decapper can't finalize (missing s', R)
- Wait indefinitely? Timeout? Reject proof?

Scenario 3: Concurrent phase transitions
- Multiple participants transition phases independently
- No consensus on current phase
- Protocol divergence, potential deadlock
```

**Recommendation:**
```
Define explicit state machine:

State 0: SETUP
  → Collect arming shares
  → Transition when: All n armers published + verified
  → Timeout: T_arm (e.g., 24 hours)

State 1: PRESIGN
  → MuSig2 nonce commitment + aggregation + signing
  → Transition when: Valid s', R produced
  → Timeout: T_presig (e.g., 1 hour)

State 2: PROOF_WAIT
  → Wait for valid Groth16+GS proof
  → Transition when: Valid proof received
  → Timeout: T_proof (e.g., 48 hours)

State 3: DECAP
  → Decapsulation + signature finalization
  → Transition when: Valid signature produced
  → Timeout: T_decap (e.g., 1 hour)

State 4: BROADCAST
  → Bitcoin transaction broadcast
  → Complete when: Transaction confirmed

State E: ABORT
  → CSV timeout expired, abort path enabled
```

### 2. No Explicit Timeouts (Liveness Risk)

**Problem:** Except for optional CSV abort, no timeout mechanisms specified.

**Scenarios where timeout needed:**

**Timeout 1: Arming phase**
```
Problem: One armer never publishes share
Result: Protocol waits indefinitely
Needed: T_arm timeout (e.g., 24 hours)
Action: Abort if not all shares received within T_arm
```

**Timeout 2: MuSig2 coordination**
```
Problem: MuSig2 participant goes offline during nonce aggregation
Result: Pre-signature never completes
Needed: T_presig timeout (e.g., 1 hour)
Action: Abort if pre-sig doesn't complete within T_presig
```

**Timeout 3: Proof generation**
```
Problem: Prover never generates proof (too expensive, hardware failure, etc.)
Result: Funds locked indefinitely (if no abort path)
Needed: T_proof timeout (e.g., 48 hours)
Action: Enable abort path if proof doesn't arrive
```

**v2.0 partial mitigation:**
- CSV abort path (optional)
- Allows recovery after Δ blocks/time
- But no specification of Δ value or main path timeouts

### 3. Race Conditions: Multiple Decappers

**Problem:** After proof published, multiple parties might attempt decapsulation simultaneously.

**Scenario:**
```
t=0: Proof published on-chain or broadcast
t=1: Decapper A starts: Derive M, compute K, decrypt, finalize sig
t=1: Decapper B starts: (same process, parallel)
t=1: Decapper C starts: (same process, parallel)

t=2: All derive same α (deterministic)
t=3: All compute same s = s' + α
t=4: All build transaction with same signature

t=5: A broadcasts tx_A
t=5: B broadcasts tx_B (identical to tx_A except maybe fees)
t=5: C broadcasts tx_C

Result: All three txs are identical (or nearly identical)
Outcome: First to confirm wins, others wasted computation
```

**Impact:**
- Wasted computation (not security issue)
- Incentive problem: Why be second if first wins?
- MEV-like: Rational actors compete, increase fees

**Potential issues:**
- Transaction malleability: Different outputs/fees
- RBF (Replace-By-Fee): Higher fee tx replaces lower
- CPFP (Child-Pays-For-Parent): Manipulation via child tx

**v2.0 mitigation:** None specified.

**Recommendations:**
- **Option A:** Designated decapper (only one authorized)
- **Option B:** First-decap-wins (explicit in protocol)
- **Option C:** Coordination layer (off-chain agreement on who decaps)

### 4. Front-Running: Mempool Observation

**Problem:** After signature finalized, window before confirmation allows front-running.

**Attack vector:**
```
1. Honest decapper computes s, builds tx_honest
2. Broadcasts tx_honest to Bitcoin mempool
3. Attacker monitors mempool, observes tx_honest
4. Extracts signature s from tx_honest witness
5. Builds tx_attack with:
   - Same inputs (same Bitcoin spend)
   - Different outputs (attacker's addresses)
   - Higher fee (or RBF flag)
6. Broadcasts tx_attack
7. Miners prefer tx_attack (higher fee)
8. Honest party's outputs replaced with attacker's

Result: Value extraction despite valid protocol execution
```

**Mitigation considerations:**

**Bitcoin SIGHASH_ALL:**
- Signature commits to inputs AND outputs
- Cannot change outputs without invalidating signature
- **BUT:** Some flexibility exists:
  - Change outputs (attacker can redirect change)
  - Fee manipulation (via input/output values)
  - CPFP anchor outputs

**Partial mitigation (inherent to protocol):**
- Signature commits to specific transaction template
- txid_template in ctx_core binds to exact tx structure
- Cannot arbitrarily change outputs

**Remaining vulnerability:**
- If transaction template has flexibility (e.g., change address)
- Attacker might manipulate within allowed parameters

**v2.0 mitigation:** None explicitly specified.

**Recommendations:**
- **Option A:** Encrypted mempool (Bitcoin Core v24+ feature)
- **Option B:** Direct miner submission (bypass public mempool)
- **Option C:** Commit-reveal: Commit to tx before broadcasting
- **Option D:** Multi-sig constraint: Outputs must include specific keys

### 5. Timing Side-Channels

**Problem:** Timing variations leak information about protocol execution.

**Side-channel 1: Pairing computation timing**
```
Issue: Number of pairings reveals GS attestation structure
- Different m₁, m₂ → different computation time
- Attacker measures time → infers attestation size
- Size might correlate with witness structure

Mitigation:
- Constant-time pairing operations (library-dependent)
- Pad timing to maximum expected (e.g., always wait for 96 pairings time)
```

**Side-channel 2: DEM decryption timing**
```
Issue: Success vs failure timing differs
- Successful decrypt: Parse plaintext, verify hash
- Failed decrypt: Abort early (wrong tag, padding error)
- Timing difference reveals success/failure

Information leakage:
- Attacker submits multiple proofs
- Measures decryption timing for each
- Identifies which proofs have valid structure

Mitigation:
- Constant-time decryption (DEM library must support)
- Always complete full decryption + verification
- No early abort on error
```

**Side-channel 3: PoCE-B verification timing**
```
Issue: Timing reveals which armer shares are valid
- Valid share: Tᵢ = sᵢ·G passes quickly
- Invalid share: Verification fails at different stages
- Timing reveals failure point

Information leakage:
- In threshold setting (k-of-n)
- Attacker infers which armers are honest
- Could influence future protocol rounds

Mitigation:
- Constant-time point multiplication
- Constant-time comparison for Tᵢ ?= sᵢ·G
- Verify all shares (don't abort on first failure)
```

**v2.0 mention (§7, line 292 equivalent):**
```
Generic: "Use constant-time implementations"
```

**What's missing:**
- No normative requirements (MUST vs SHOULD)
- No specification of which operations
- No testing requirements for timing uniformity

### 6. Proof Submission During Arming (Edge Case)

**Problem:** What if valid proof arrives before arming completes?

**Scenario:**
```
t=0: Arming phase starts
t=1: Armer 1 publishes share
t=2: Armer 2 publishes share
...
t=5: Prover (who pre-computed proof for this vk,x) submits proof
     → Proof is valid (matches expected vk,x)
     → But arming not complete yet
t=6: Armer n publishes last share

Questions:
- Is early proof accepted?
- Does it trigger premature phase transition?
- Could this break security assumptions?
```

**Security analysis:**
- Proof doesn't depend on arming shares (only on vk, x, witness)
- Early proof shouldn't break security (can't decrypt without full α)
- But could confuse state machine

**Recommendation:**
- Reject proofs until STATE >= PROOF_WAIT
- Or: Accept proof but don't process until arming complete

---

## Attack Vectors (Updated for v2.0)

### Attack 7.1: Mempool Front-Running

**Severity:** Medium (value extraction)
**Likelihood:** High (if public mempool used)

```
Preconditions:
- Transaction broadcasted to public Bitcoin mempool
- Attacker monitors mempool
- Transaction structure allows some flexibility

Attack steps:
1. Honest decapper computes s, builds tx_honest
2. Broadcasts tx_honest (signature s in witness)
3. Attacker observes tx_honest in mempool
4. Extracts signature s from witness
5. If tx template allows flexibility:
   - Modify change address to attacker
   - Increase fee (RBF)
   - Manipulate CPFP anchor
6. Broadcasts tx_attack with higher fee
7. Miners include tx_attack (higher fee)
8. Attacker extracts value

Impact:
- Economic loss to honest party
- MEV extraction
- Reduced protocol fairness
```

**v2.0 mitigation:** None specified.

**Recommendations:**
- Use encrypted mempool (Bitcoin Core v24+)
- Direct miner submission
- Transaction template with no flexibility (fixed outputs)

### Attack 7.2: Timing Side-Channel on Proof Structure

**Severity:** Low (information leakage)
**Likelihood:** Medium (if implementations not constant-time)

```
Preconditions:
- Attacker can submit multiple proofs
- Attacker can measure decapsulation timing
- Implementation not constant-time

Attack steps:
1. Attacker generates multiple proofs (different witnesses)
2. Each proof might have different GS structure (m₁, m₂)
3. Submits each proof, measures decapsulation time
4. Timing reveals:
   - Number of pairings (m₁ + m₂)
   - Inference about witness structure
5. Violates zero-knowledge property (leaks beyond intended)

Impact:
- Information leakage about witness
- Might reveal sensitive computation details
- Privacy violation (not direct security break)
```

**v2.0 mitigation:** Generic mention of constant-time (not normative).

**Recommendations:**
- MUST use constant-time pairing library
- Pad timing to maximum (96 pairings always)
- Blind timing measurements (random delays)

### Attack 7.3: Liveness DoS via Timeout Abuse

**Severity:** Medium (liveness/DoS)
**Likelihood:** Low (requires participant cooperation)

```
Preconditions:
- No explicit timeouts for phases
- Participants can delay indefinitely

Attack steps:
1. Malicious armer delays share publication
2. Protocol waits indefinitely (no T_arm timeout)
3. Other participants locked (cannot proceed)
4. Funds locked until abort path enabled (if exists)

Or:
1. Malicious MuSig2 participant stops responding
2. Pre-signature never completes
3. Protocol deadlocked

Impact:
- Protocol liveness degraded
- Funds locked (denial of service)
- Requires abort path for recovery
```

**v2.0 mitigation:** Optional CSV abort path (SHOULD).

**Recommendations:**
- Mandatory timeouts for all phases
- Abort path MUST (not SHOULD)
- Penalty mechanism for timeout abusers

---

## Recommendations

### Phase 1: Specification Enhancements (1-2 months)

#### 1. Define Explicit State Machine (HIGHEST PRIORITY)
**Owner:** Protocol designers
**Timeline:** 2 weeks
**Deliverable:** State machine specification

```
Formal state machine:

States: {SETUP, ARMING, PRESIGN, PROOF_WAIT, DECAP, BROADCAST, ABORT}

Transitions:
- SETUP → ARMING: Context initialized
- ARMING → PRESIGN: All n armers published + verified
- PRESIGN → PROOF_WAIT: Valid s', R produced
- PROOF_WAIT → DECAP: Valid proof received
- DECAP → BROADCAST: Signature finalized
- BROADCAST → COMPLETE: Transaction confirmed
- * → ABORT: Timeout expired

Timeout parameters:
- T_arm = 24 hours (arming phase)
- T_presig = 1 hour (pre-signature phase)
- T_proof = 48 hours (proof generation)
- T_decap = 1 hour (decapsulation)
- T_abort = CSV Δ (abort path enablement)

State invariants:
- Cannot skip states (except to ABORT)
- Cannot regress (except via ABORT)
- All participants must agree on current state
```

#### 2. Mandatory Timeouts for All Phases
**Owner:** Protocol designers
**Timeline:** 1 week
**Deliverable:** Timeout specification

```
Normative requirements:

MUST: Arming phase timeout (T_arm)
- If not all armers published within T_arm: Abort
- Refund mechanism or restart

MUST: Pre-signature timeout (T_presig)
- If MuSig2 not complete within T_presig: Abort
- Restart with different participant set

MUST: Proof timeout (T_proof)
- If no valid proof within T_proof: Enable abort path
- CSV-based recovery

MUST: Decapsulation timeout (T_decap)
- If signature not finalized within T_decap: Retry or abort
- Designate backup decapper
```

#### 3. Race Condition Resolution
**Owner:** Protocol designers
**Timeline:** 2 weeks
**Deliverable:** Coordination mechanism

**Option A: Designated decapper**
```
Specify: Only designated party can decap
- Rotation mechanism for fairness
- Fallback if designated party fails
```

**Option B: First-decap-wins**
```
Specify: First valid signature accepted
- Others abort if signature already broadcasted
- Coordination via monitoring broadcast channels
```

**Option C: Off-chain coordination**
```
External: Coordination layer (not protocol-level)
- Participants agree off-chain who decaps
- Protocol accepts any valid signature
```

**Recommendation:** Start with Option B (simplest, most decentralized).

### Phase 2: Implementation Requirements (2-4 months)

#### 4. Constant-Time Operations (MUST)
**Owner:** Implementation team
**Timeline:** 1 month
**Deliverable:** Constant-time implementation + tests

**Normative requirements:**
```
MUST: Pairing operations constant-time
- Use constant-time pairing library (e.g., blst)
- No branching on secret-dependent values
- Cache-timing resistant

MUST: DEM decryption constant-time
- Complete full decryption + verification always
- No early abort on error
- Pad timing if necessary

MUST: Point operations constant-time
- Scalar multiplication (for PoCE-B Tᵢ ?= sᵢ·G)
- Point addition/subtraction
- Comparison operations

Testing:
- Timing measurement tests
- Statistical analysis (t-tests for timing uniformity)
- dudect or similar timing analysis tools
```

#### 5. Timing Side-Channel Mitigation
**Owner:** Implementation team
**Timeline:** 1 month
**Deliverable:** Hardened implementation

**Strategies:**
```
1. Pad timing to maximum expected:
   - Always wait for 96 pairings time (even if fewer)
   - Adds latency but eliminates side-channel

2. Blind measurements:
   - Add random delays (within reasonable range)
   - Prevents precise timing measurements

3. Batch processing:
   - Process multiple operations together
   - Uniform timing for batch (harder to isolate individual ops)
```

#### 6. Front-Running Protection
**Owner:** Application layer / Integration team
**Timeline:** 1-2 months
**Deliverable:** Deployment strategy

**Strategies:**
```
Option A: Encrypted mempool
- Use Bitcoin Core v24+ encrypted mempool feature
- Only miners see transaction before confirmation
- Requires miner support

Option B: Direct miner submission
- Bypass public mempool entirely
- Submit directly to mining pool APIs
- Requires trust in mining pool

Option C: Transaction privacy enhancements
- Use transaction template with no flexibility
- Fixed outputs (no change address manipulation)
- Minimize MEV opportunities

Recommendation: Combination of A + C
```

### Phase 3: Testing & Validation (2-3 months)

#### 7. Timing Analysis Test Suite
**Owner:** Security testing team
**Timeline:** 2 months
**Deliverable:** Comprehensive timing tests

**Test categories:**
```
1. State transition timing:
   - Measure time for each phase transition
   - Ensure timeouts enforced
   - Test timeout edge cases

2. Side-channel resistance:
   - dudect or t-test analysis
   - Compare timing for success vs failure
   - Statistical uniformity tests

3. Race condition tests:
   - Multiple concurrent decappers
   - Out-of-order message delivery
   - Stress test with high concurrency

4. Front-running simulation:
   - Mempool monitoring attack simulation
   - RBF attack tests
   - CPFP manipulation tests
```

---

## Acceptance Criteria for Resolution

This flaw can be considered **RESOLVED** when:

### Option A: Full Specification + Testing (Preferred)
- [ ] Explicit state machine defined (with transitions and timeouts)
- [ ] Mandatory timeouts for all phases (normative MUST clauses)
- [ ] Race condition resolution mechanism specified
- [ ] Constant-time requirements (normative, all critical operations)
- [ ] Front-running mitigation strategy documented
- [ ] Timing analysis test suite (100+ tests, all passing)
- [ ] Reference implementation demonstrates timing uniformity

### Option B: Partial Specification + Guidelines (Acceptable)
- [ ] State machine defined with recommended timeouts (SHOULD clauses)
- [ ] Race condition guidelines (not enforced by protocol)
- [ ] Constant-time recommendations (best practices document)
- [ ] Front-running awareness documented (user responsibility)
- [ ] Basic timing tests (covers common cases)

---

## Related Flaws

- **PVUGC-004:** PoCE-B decapper-local (related - timing of PoCE-B verification affects this)
- **PVUGC-008:** MuSig2 compartmentalization (related - nonce timing and coordination)

---

## Notes

- **v2.0 Status:** Largely unchanged (CSV abort mentioned, but main issues remain)
- **Priority:** Medium (affects liveness and fairness, not core crypto security)
- **Mainnet considerations:** Acceptable with:
  - Clear timeout documentation
  - Front-running awareness
  - Monitoring/alerting for timing issues
- **User impact:** Potential for locked funds, MEV extraction, information leakage
- **Implementation-dependent:** Most issues depend on specific deployment choices

---

**Version History:**
- **v1.0 (2025-10-07):** Initial identification (no timeout/timing specs)
- **v2.0 (2025-10-07):** Minimal change (CSV abort mentioned, main issues unchanged)

**Last Updated:** 2025-10-07
**Next Review:** After implementation testing (target: 3-6 months)
