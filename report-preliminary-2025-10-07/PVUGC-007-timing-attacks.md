# PVUGC-007: Timing Attacks - Race Conditions and Side-Channels

**Flaw Code:** PVUGC-007
**Severity:** 🟡 **MEDIUM**
**Status:** 🔓 Open
**Date Identified:** 2025-10-07

---

## Component
Multi-stage protocol flow and constant-time operations

## Location
- **Section:** §9 (Activation patterns)
- **Line:** 292 (constant-time mandate)

---

## Description

Protocol involves multiple phases (arming → pre-sign → decrypt → broadcast) without explicit timeouts or synchronization. Line 292 mentions constant-time requirements but doesn't specify enforcement.

**Concerns:** Race conditions, front-running, timing side-channels leaking witness information.

---

## Specific Issues

### 1. No Phase Timeouts
- Arming: What if some armers never complete?
- Pre-sign: What if MuSig2 doesn't converge?
- Only timeout: CSV in optional Abort path

### 2. Front-Running Window
After α decrypted and s computed, window before broadcast:
- Mempool monitoring could observe transaction
- Attacker extracts s, builds competing tx with higher fee
- Note: SIGHASH_ALL binds outputs, but fee/CPFP anchor modifiable

### 3. Timing Side-Channels
- **Pairing count:** Number of pairings reveals m₁, m₂ (proof structure)
- **DEM decryption:** Success/failure timing differs
- **Cache timing:** Table lookups in pairing computation

---

## Attack Vectors

### Attack 7.1: Mempool Front-Running
```
1. Honest party decrypts α, computes s, builds tx
2. Broadcasts tx to Bitcoin network
3. Attacker monitors mempool, extracts s from witness
4. Builds tx' with higher fee (or RBF)
5. Miners prefer tx' (higher fee)
6. Honest party loses funds or outputs redirected
```

### Attack 7.2: Timing Side-Channel on Proof Structure
```
1. Attacker submits multiple proofs (different witnesses)
2. Measures decapsulation timing
3. Pairing computation time reveals m₁, m₂ (commitment count)
4. Infers witness structure information
5. Violates zero-knowledge beyond "proof exists"
```

---

## Recommendations

### Specification

#### 1. Add Phase Timeouts
**Priority:** P2, **Timeline:** 1 week

```
Phase 1: Arming (deadline: T₁)
- All armers must publish by T₁
- If incomplete: abort, optional refund mechanism

Phase 2: Pre-sign (deadline: T₂)
- MuSig2 completes by T₂
- If fail: abort

Phase 3: Proof window (T₃ to T₄)
- Valid proof accepted in window
- After T₄: Abort path activatable

Phase 4: Broadcast (deadline: T₅)
- Decapper must broadcast by T₅
- Grace period for network propagation
```

#### 2. Constant-Time Operations (Normative)
**Priority:** P2, **Timeline:** Ongoing

Update §12 (line 292) with specific requirements:
```markdown
### Constant-Time Requirements (Normative)

MUST use constant-time implementations:
☐ Pairing computation (library-dependent, verify with vendor)
☐ HKDF (constant-time HMAC)
☐ DEM encryption/decryption
  - Pad timing to max expected duration
  - Use constant-time comparison for tag verification
☐ PoCE-B checks (constant-time regardless of success/failure)

SHOULD mitigate cache timing:
☐ Disable table-based optimizations in sensitive operations
☐ Use blinding techniques for pairing computation
```

#### 3. Front-Running Mitigations
**Priority:** P2, **Timeline:** 2-3 weeks (implementation)

Options:
- **Commit-reveal:** Decapper commits to tx hash before broadcasting
- **Encrypted mempool:** Use Bitcoin Core v24+ encrypted mempool features
- **Direct miner submission:** Bypass public mempool entirely
- **Batch submission:** Multiple txs submitted simultaneously (reduces targeting)

### Testing

#### 4. Timing Analysis
**Timeline:** 2 weeks

- [ ] Measure pairing computation timing across different proof structures
- [ ] Verify constant-time DEM operations (use tools like `valgrind --tool=cachegrind`)
- [ ] Test front-running resistance (simulated adversarial mempool)

---

## Acceptance Criteria

- [ ] Phase timeouts specified
- [ ] Constant-time requirements documented (normative)
- [ ] Front-running mitigation chosen and implemented
- [ ] Timing analysis performed, no significant leakage
- [ ] External audit validates constant-time properties

---

**Last Updated:** 2025-10-07
