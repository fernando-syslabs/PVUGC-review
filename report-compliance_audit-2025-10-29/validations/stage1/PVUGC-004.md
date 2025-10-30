# PVUGC-004: PoCE Soundness - Regression Check

**Date:** 2025-10-28
**Original Status:** ✅ Resolved/Validated (2025-10-26)
**Regression Status:** ✅ No Regression
**Decision Method:** 🔐 Crypto-Peer-Reviewer Consultation

---

## Mitigation Summary (from 2025-10-26 peer review)

The 2025-10-26 peer review validated PVUGC-004 as resolved through the **Hardened PoCE-A Circuit** which cryptographically prevents liveness griefing attacks.

### Original Vulnerability (Pre-Mitigation)
**From report-peer_review-2025-10-26/PVUGC-004.md, lines 60-80:**

The original PoCE-A proved:
- Mask correctness: `D_j = U_j^ρ_i` (G2), `D_k = V_k^ρ_i` (G1)
- Adaptor share commitment: `T_i = s_i·G`
- Randomness linking: `ρ_link = H_tag(ρ_i)`
- Non-zero check: `ρ_i ≠ 0`

**Critical Gap:** Original PoCE-A did NOT prove:
- `ct_i` actually encrypts `s_i`
- `ct_i` uses the correct key `K_i` derived from `ρ_i`
- Tag `τ_i` correctly binds to ciphertext

This enabled **liveness griefing attack** (lines 244-281):
- Malicious armer generates valid `(ρ_i, s_i)` and correct public artifacts
- Publishes **garbage ciphertext** `ct_i' = random_bytes()`
- Arm-time checks PASS (original PoCE-A only verifies masks/commitment)
- Honest prover wastes resources generating expensive Groth16+GS proof
- Decapsulation FAILS (garbage ciphertext cannot decrypt)
- Protocol halts, funds locked until timeout

**Attack Cost-Benefit (Original Design):**
- Attacker cost: ~2-10 seconds (1 NIZK proof)
- Victim cost: Minutes to hours (wasted Groth16 proving)
- Success probability: 100% (deterministic)
- Risk to attacker: Zero (no bonds, no attribution)

### Primary Mitigation: Hardened PoCE-A Circuit
**From report-peer_review-2025-10-26/PVUGC-004.md, lines 87-99:**

**Specification Location:** PVUGC-2025-10-20.md §8, lines 220-224

**Normative Requirements:**
```
PoCE-A (Arm-time verifiable encryption & mask-link, SNARK-friendly) - A NIZK proving:
- Knowledge of (ρ_i, s_i, h_i) such that:
  - (i) D_{1,j} = U_j^{ρ_i} for all j, D_{2,k} = V_k^{ρ_i} for all k
  - (ii) T_i = s_i·G
  - (iii) ρ_link = H_tag(ρ_i)
- Key derivation (in-circuit): K_i = Poseidon2(ser_GT(G_{G16}^{ρ_i}) || H_bytes(ctx_hash) || GS_instance_digest)
- DEM correctness (SNARK-friendly): ct_i = (s_i|h_i) ⊕ Poseidon2(K_i, AD_core) and τ_i = Poseidon2(K_i, AD_core, ct_i)
- ρ_i ≠ 0 (via auxiliary witness)
```

**Cryptographic Properties (lines 413-436):**
- **Constraint C4.1:** `M_i = G_{G16}^{ρ_i}` (G_T exponentiation in-circuit)
- **Constraint C4.2:** `K_i = Poseidon2(ser(M_i) || ...)` (Key derivation in-circuit)
- **Constraint C5.1:** `pt = s_i || h_i` (Plaintext construction)
- **Constraint C5.2:** `keystream = Poseidon2(K_i, AD_core)` (Keystream derivation)
- **Constraint C5.3:** `ct_i = pt ⊕ keystream` (Ciphertext binding) [CRITICAL]
- **Constraint C5.4:** `τ_i = Poseidon2(K_i, AD_core, ct_i)` (Tag binding)

**Why This Prevents the Attack:**
- If adversary publishes garbage `ct_i' ≠ (s_i || h_i) ⊕ Poseidon2(K_i, AD_core)`, constraint C5.3 FAILS during proof generation
- SNARK soundness prevents publishing valid proof with invalid ciphertext
- Attack becomes **cryptographically impossible** (not just detectable)

**Mathematical Validation (lines 659-701):**
- Theorem: Hardened PoCE-A satisfies Decrypt-on-Proof Soundness
- Proof: By SNARK soundness, if proof verifies, all constraints C1-C5 hold
- Therefore: Decapsulation with same `K_i` MUST succeed
- QED

### Status Assessment (lines 17-26)
**From report-peer_review-2025-10-26/PVUGC-004.md:**
```
Verdict: ✅ VALIDATED
Status: High-severity vulnerability already addressed in specification (lines 220-224),
        requires implementation compliance verification

Key Findings:
1. Soundness Gap Confirmed - Original design had exploitable vulnerability
2. v2.0 Specification Includes Fix - Hardened PoCE-A with in-circuit constraints
3. Attack Practical & Demonstrable - Vulnerability analysis remains valid
4. Verification Required - Design is sound; operational risk is implementation divergence
```

---

## Verification in PVUGC-2025-10-27.md

### Search Methodology
**Keywords Used:** "PoCE-A", "in-circuit", "Key derivation", "DEM correctness", "Poseidon2"
**Sections Searched:** §8 (PoCE and DEM details)

### Location: §8 (PoCE and DEM details), Lines 274-278

**Normative Language (PVUGC-2025-10-27.md):**
```markdown
**PoCE-A (Arm-time verifiable encryption & mask-link, SNARK-friendly).** A NIZK proving, for share $i$:
* Knowledge of $(\rho_i,s_i,h_i)$ s.t. (i) $D_j=Y_j^{\rho_i}$ for all $j \in [0,n_B-1]$, $D_\delta=[\delta]_2^{\rho_i}$; (ii) $T_i=s_iG$; (iii) $\rho\_\text{link}=H_\text{tag}(\rho_i)$.
* Key derivation (in-circuit): $K_i = \mathrm{Poseidon2}(\mathrm{ser}_{\mathbb{G}_T}(R(\mathsf{vk},x)^{\rho_i}) || H_\text{bytes}(\texttt{ctx\_hash}) || \texttt{GS\_instance\_digest})$.
* DEM correctness (SNARK-friendly): $\texttt{ct}_i = (s_i\|h_i) \oplus \mathrm{Poseidon2}(K_i, \text{AD\_core})$ and $\tau_i = \mathrm{Poseidon2}(K_i, \text{AD\_core}, \texttt{ct}_i)$.
* $\rho_i\neq 0$ (via auxiliary $\rho_i\cdot u_i=1$)
```

### Detailed Constraint Mapping

| Constraint | PVUGC-2025-10-20 (v2.0) | PVUGC-2025-10-27 (v2.7) | Status |
|------------|-------------------------|--------------------------|--------|
| **C1: Mask Correctness** | `D_{1,j} = U_j^{ρ_i}`, `D_{2,k} = V_k^{ρ_i}` | `D_j=Y_j^{ρ_i}` (all j), `D_δ=[δ]_2^{ρ_i}` | ✅ PRESERVED |
| **C2: Adaptor Share** | `T_i = s_i·G` | `T_i=s_iG` | ✅ PRESERVED |
| **C3: Randomness Link** | `ρ_link = H_tag(ρ_i)` | `ρ_link=H_tag(ρ_i)` | ✅ PRESERVED |
| **C3.2: Non-zero** | `ρ_i ≠ 0` (via auxiliary) | `ρ_i≠0` (via auxiliary `ρ_i·u_i=1`) | ✅ PRESERVED |
| **C4.1: G_T Exponentiation** | `M_i = G_{G16}^{ρ_i}` | `R(vk,x)^{ρ_i}` (in K_i derivation) | ✅ PRESERVED |
| **C4.2: Key Derivation** | `K_i = Poseidon2(ser(M_i) || ...)` | `K_i = Poseidon2(ser_GT(R(vk,x)^{ρ_i}) || H_bytes(ctx_hash) || GS_instance_digest)` | ✅ PRESERVED |
| **C5.1-C5.2: Keystream** | `keystream = Poseidon2(K_i, AD_core)` | Implicit in `ct_i = (s_i\|h_i) ⊕ Poseidon2(K_i, AD_core)` | ✅ PRESERVED |
| **C5.3: Ciphertext Binding** | `ct_i = pt ⊕ keystream` | `ct_i = (s_i\|h_i) ⊕ Poseidon2(K_i, AD_core)` | ✅ PRESERVED |
| **C5.4: Tag Binding** | `τ_i = Poseidon2(K_i, AD_core, ct_i)` | `τ_i = Poseidon2(K_i, AD_core, ct_i)` | ✅ PRESERVED |

**Note:** The notation changed slightly (`G_{G16}` → `R(vk,x)`, `U_j/V_k` → `Y_j/[δ]_2`) but the cryptographic constraints are identical.

### Verification: In-Circuit Constraints Present

**Constraint C4 (Key Derivation) - Line 276:**
```
Key derivation (in-circuit): K_i = Poseidon2(ser_GT(R(vk,x)^{ρ_i}) || H_bytes(ctx_hash) || GS_instance_digest)
```
- ✅ **"in-circuit"** explicitly stated
- ✅ G_T exponentiation `R(vk,x)^{ρ_i}` present (equivalent to `G_{G16}^{ρ_i}`)
- ✅ Key derivation from pairing product result

**Constraint C5 (DEM Correctness) - Line 277:**
```
DEM correctness (SNARK-friendly): ct_i = (s_i|h_i) ⊕ Poseidon2(K_i, AD_core) and τ_i = Poseidon2(K_i, AD_core, ct_i)
```
- ✅ **"DEM correctness"** explicitly stated (constraint group C5)
- ✅ **"SNARK-friendly"** indicates in-circuit verification
- ✅ Ciphertext equation binding `ct_i` to witnesses `(s_i, h_i)` and key `K_i`
- ✅ Tag binding `τ_i` to key and ciphertext

**Implementation Note - Line 280:**
```
**Implementation note (MUST):** PoCE-A MUST use per-column Schnorr commitments...
```
- ✅ Normative requirement (MUST) for implementation
- ✅ Prevents related attack vector (collapsing columns)

### Verification: Attack Prevention Mechanism

**Line 307 (Why PoCE is needed):**
```
**Why PoCE is needed:** Without PoCE-A, a malicious armer could publish masks/ciphertexts
not derived from the same ρ_i, breaking decrypt-on-proof soundness.
```
- ✅ Security rationale explicitly stated
- ✅ Attack vector acknowledged
- ✅ Mitigation mechanism identified

**Line 296 (Ceremony rule):**
```
Ceremony rule (MUST): do not pre-sign unless all PoCE-A proofs and PoK verify for all shares.
```
- ✅ Enforcement point specified (arm-time)
- ✅ Normative requirement (MUST)

---

## Expert Consultation

Given the **High severity** of PVUGC-004 and its **cryptographic domain**, I consulted the crypto-peer-reviewer to validate that:
1. The Hardened PoCE-A circuit constraints are sufficient to prevent the liveness griefing attack
2. The in-circuit key derivation and DEM correctness provide the required soundness property
3. No subtle weakening or omission has occurred in the v2.7 specification

### Crypto-Peer-Reviewer Vote

**Vote:** PASS (No Regression)

**Key Reasoning (from consultation response):**

1. **Cryptographic Enforcement (Constraint C5):**
   - Line 277's equation `ct_i = (s_i|h_i) ⊕ Poseidon2(K_i, AD_core)` is a **constraint equation**
   - In NIZK context, prover must witness values satisfying this equation
   - Label "SNARK-friendly" confirms in-circuit verification
   - By SNARK soundness: Valid proof → constraint holds → decapsulation succeeds
   - **Attack Prevention:** Garbage `ct_i'` → constraint unsatisfied → proof generation fails

2. **In-Circuit Key Derivation (Constraint C4):**
   - Line 276 term `R(vk,x)^{ρ_i}` requires in-circuit G_T exponentiation
   - Label "in-circuit" confirms computation within NIZK
   - Binding chain: `ρ_i → M_i = R(vk,x)^{ρ_i} → K_i = Poseidon2(...) → ct_i`
   - Adversary cannot use different `ρ_i` for masks vs ciphertext

3. **Attack Prevention Analysis:**
   - Adversary cannot satisfy line 277's constraint with garbage ciphertext
   - `ct_i` is public input (verifier checks published value matches circuit output)
   - Publishing incorrect value → constraint violated → no valid proof possible
   - SNARK soundness: `Pr[valid proof with incorrect ct] ≤ negl(λ)`

4. **Notation Equivalence:**
   - `G_{G16}` → `R(vk,x)`: Both represent pairing product target in G_T ✅
   - `M_i = G_{G16}^{ρ_i}` → `R(vk,x)^{ρ_i}`: Same G_T exponentiation ✅
   - All notation changes syntactic (variable renaming) or clarifying (explicit groups) ✅
   - No semantic changes to cryptographic properties ✅

5. **Constraint Completeness:**
   - All 10 critical constraints (C1-C5.4) present explicitly or implicitly
   - Consolidated form uses compact mathematical notation (standard practice)
   - "SNARK-friendly" label confirms in-circuit constraints
   - Sufficient for unambiguous implementation

**Risk Assessment:** Security Impact = NONE, Attack Scenarios Enabled = NONE

**Standards Compliance:**
- §3A.1 (Decrypt-on-Proof Soundness): ✅ MET (line 277 enforces ciphertext correctness)
- §5A.3 (Circuit Constraint Implementation): ✅ MET (line 280 MUST requirements)
- §8.1 (Constant-Time Execution): ✅ MET (line 306 constant-time DEM)

**Full Consultation Details:**
- Request: `/sandbox/report-compliance_audit-2025-10-29/.consultation-pvugc-004-crypto.md`
- Response: `/sandbox/report-compliance_audit-2025-10-29/.consultation-pvugc-004-crypto-response.md`

---

## Regression Decision

**Result:** ✅ **No Regression**

**Reasoning:**

### 1. Essential Mitigation Preserved (Validated by Expert)

The **Hardened PoCE-A Circuit** is the essential mitigation that resolves PVUGC-004. This is:
- **Explicitly specified** in PVUGC-2025-10-27.md lines 274-278
- **Cryptographically sound** per crypto-peer-reviewer validation
- **Attack-preventive** through SNARK soundness guarantees

The mitigation mechanism is **unchanged**:
- v2.0: Hardened PoCE-A with constraints C4 (key derivation) + C5 (DEM correctness)
- v2.7: Hardened PoCE-A with same constraints using consolidated notation

### 2. All Critical Constraints Present

**Constraint C4 (In-Circuit Key Derivation) - Line 276:**
```
Key derivation (in-circuit): K_i = Poseidon2(ser_GT(R(vk,x)^{ρ_i}) || H_bytes(ctx_hash) || GS_instance_digest)
```
- ✅ G_T exponentiation `R(vk,x)^{ρ_i}` explicit (C4.1)
- ✅ Poseidon2 key derivation explicit (C4.2)
- ✅ "in-circuit" label confirms NIZK verification

**Constraint C5 (In-Circuit DEM Correctness) - Line 277:**
```
DEM correctness (SNARK-friendly): ct_i = (s_i|h_i) ⊕ Poseidon2(K_i, AD_core) and τ_i = Poseidon2(K_i, AD_core, ct_i)
```
- ✅ Ciphertext binding equation explicit (C5.3)
- ✅ Tag binding equation explicit (C5.4)
- ✅ "SNARK-friendly" label confirms in-circuit constraints
- ✅ Plaintext and keystream implicit in equation (C5.1-C5.2)

Per crypto-peer-reviewer: "All 10 critical constraints (C1-C5.4) present explicitly or implicitly. Consolidated form uses compact mathematical notation (standard practice). Sufficient for unambiguous implementation."

### 3. Attack Prevention Mechanism Intact

**Crypto-Peer-Reviewer Analysis:**
- Garbage ciphertext attack: Constraint C5.3 prevents proof generation with incorrect `ct_i`
- SNARK soundness: `Pr[valid proof with incorrect ct] ≤ negl(λ)`
- Attack outcome: Cryptographically impossible (proof generation fails)

**Historical Attack:** Malicious armer publishes `ct_i' = random_bytes()`
- v1.0 (Original): ✅ Arm-time passes, ❌ Decap fails (vulnerable)
- v2.0 (Hardened): ❌ Proof generation fails (attack prevented)
- v2.7 (Current): ❌ Proof generation fails (attack prevented)

### 4. Notation Changes Validated as Benign

Per crypto-peer-reviewer:
- `G_{G16}` → `R(vk,x)`: ✅ Cryptographically equivalent (both G_T pairing product)
- `M_i = G_{G16}^{ρ_i}` → `R(vk,x)^{ρ_i}`: ✅ Same operation
- `ser()` → `ser_GT()`: ✅ Clarification (explicit group serialization)

"All notation changes syntactic (variable renaming) or clarifying (explicit groups). No semantic changes to cryptographic properties."

### 5. Standards Compliance Maintained

**Per crypto-peer-reviewer validation:**
- §3A.1 (Decrypt-on-Proof Soundness): ✅ MET
- §5A.3 (Circuit Constraint Implementation): ✅ MET
- §8.1 (Constant-Time Execution): ✅ MET

### 6. Normative Requirements Strengthened

**Line 280 (Implementation Note):**
```
**Implementation note (MUST):** PoCE-A MUST use per-column Schnorr commitments...
```
- Explicit MUST requirement
- Prevents related attack vector (column collapsing)

**Line 296 (Ceremony Rule):**
```
Ceremony rule (MUST): do not pre-sign unless all PoCE-A proofs and PoK verify for all shares.
```
- Explicit enforcement point
- Normative requirement (MUST)

**Line 306 (Mandatory Hygiene):**
```
**Mandatory hygiene:** ... constant-time pairings, constant-time DEM decryption, ...
```
- Constant-time requirements mandated
- Cross-reference to PVUGC-007

### 7. Security Rationale Preserved

**Line 307:**
```
**Why PoCE is needed:** Without PoCE-A, a malicious armer could publish masks/ciphertexts
not derived from the same ρ_i, breaking decrypt-on-proof soundness.
```
- Explicit acknowledgment of attack vector
- Confirms mitigation intent

---

## Gaps

**None identified.** All critical constraints present, attack prevented, standards compliance maintained.

**Cross-reference dependency:** PVUGC-010 (CRS Validation) must be validated to confirm binding enforcement and subgroup checks are preserved. (Per crypto-peer-reviewer note: "PoCE-A soundness depends on correct CRS bases and proper subgroup membership checks.")

---

## Acceptance Criteria for Continued Resolution

To maintain the resolved status of PVUGC-004 in implementations:

### Critical (MUST)
1. ✅ Implement all PoCE-A constraints C1-C5 (lines 274-278)
2. ✅ In-circuit key derivation: `K_i = Poseidon2(ser_GT(R(vk,x)^{ρ_i}) || ...)`
3. ✅ In-circuit DEM correctness: `ct_i = (s_i|h_i) ⊕ Poseidon2(K_i, AD_core)`
4. ✅ Per-column Schnorr commitments (line 280)
5. ✅ Verify all PoCE-A proofs before pre-signing (line 296)
6. ✅ Constant-time DEM decryption (line 306)

### Recommended (SHOULD)
1. ✅ Publish PoCE-B verification transcripts (line 297)
2. ✅ Implement timeout/abort path for liveness protection (line 315)

### Testing Requirements
1. ✅ Regression test: Garbage ciphertext attack must fail at proof generation
2. ✅ Performance validation: Proving time overhead within acceptable bounds
3. ✅ Soundness test: Valid proof → decapsulation succeeds

---

## Conclusion

The PVUGC-004 mitigation for PoCE Soundness shows **no regression** in PVUGC-2025-10-27.md:

- **Hardened PoCE-A Circuit:** Preserved with all constraints (C4, C5) present
- **Attack prevention:** Garbage ciphertext attack remains cryptographically impossible
- **Notation changes:** Validated as semantically equivalent
- **Standards compliance:** All framework requirements met
- **Expert validation:** Crypto-peer-reviewer confirms no regression

**Final Status:** ✅ No Regression Detected

**Confidence:** HIGH (validated by domain expert with detailed cryptographic analysis)

**Cross-validation Required:** PVUGC-010 regression check must confirm CRS validation requirements

---

**Related Issues:**
- PVUGC-010: CRS Validation (critical dependency - subgroup checks, binding property)
- PVUGC-007: Timing Attacks (constant-time DEM decryption requirement)
- PVUGC-006: Degenerate Values (R(vk,x) ≠ 1 check, line 286)

**Standards References:**
- STANDARDS-REFERENCE-FRAMEWORK.md §3A.1 (Decrypt-on-Proof Soundness)
- STANDARDS-REFERENCE-FRAMEWORK.md §5A.3 (Circuit Constraint Implementation)
- STANDARDS-REFERENCE-FRAMEWORK.md §8.1 (Constant-Time Execution)

---

**Last Updated:** 2025-10-28 16:35
