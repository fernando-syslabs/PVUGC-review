# Cryptographic Standards Reference Framework

**Version:** 1.0
**Date:** 2025-10-28
**Purpose:** Independent reference framework defining cryptographic and protocol standards drawn from established external sources (papers, RFCs, BIPs, industry best practices)
**Scope:** Applicable to security reviews, audits, and compliance validations of cryptographic and Bitcoin-integrated protocols

---

## Overview

This document defines authoritative, external standards used to validate protocols employing pairing-based cryptography and Bitcoin integrations. It consolidates:

1. **Cryptographic standards** (AGM, pairing-based crypto, hash functions)
2. **Bitcoin protocol standards** (BIPs for Schnorr, MuSig2, Taproot)
3. **Implementation security standards** (constant-time execution, canonical encodings)
4. **Multi-party protocol standards** (commit-reveal, entropy management)
5. **DoS prevention and liveness standards** (resource limits, timeout mechanisms)

### How to Use This Document

- **Security reviewers**: Reference specific sections when validating protocol claims
- **Implementation teams**: Use as compliance checklist for normative requirements
- **Auditors**: Map target protocol specifications to these standards using a separate compliance guide (kept outside this document)
- **Future reviews**: Extend this framework as new standards emerge or protocols evolve

### Integration with Validation Process

When conducting validation:

1. **Stage 1 (Regression Testing)**: Focus on Sections 1-7A for protocol-level standards
2. **Stage 2 (Standards Validation)**: Apply all sections (1-12)
3. **Specification Mapping**: Use your protocol’s compliance guide to link specification anchors to these standards
4. **Validation Methodology**: Follow your organization’s standard methodology for mapping and evidence collection

### Document Status

- ✅ **Stable sections**: 1-7, 9-12 (based on established standards)
- 🔄 **Evolving sections**: 3A (Poseidon2), 5A (multi-party constructions)
- ℹ️ **Performance figures**: Indicative and hardware/library-dependent; treat benchmarks as guidance, not guarantees

---

## Table of Contents

1. [Cryptographic Security Model Standards](#1-cryptographic-security-model-standards)
2. [Pairing-Based Cryptography Standards](#2-pairing-based-cryptography-standards)
3. [Cryptographic Derivation Standards](#3-cryptographic-derivation-standards)
   - 3A. [Cryptographic Primitive Standards](#3a-cryptographic-primitive-standards)
4. [Bitcoin Protocol Standards](#4-bitcoin-protocol-standards)
5. [Multi-Party Protocol Standards](#5-multi-party-protocol-standards)
   - 5A. [Entropy and Nonce Management Standards](#5a-entropy-and-nonce-management-standards)
6. [Specification Language Standards](#6-specification-language-standards)
7. [Issue Classification Standards](#7-issue-classification-standards)
   - 7A. [DoS Prevention and Resource Limits](#7a-dos-prevention-and-resource-limits)
8. [Implementation Security Standards](#8-implementation-security-standards)
9. [Verification and Testing Standards](#9-verification-and-testing-standards)
   - [Testing Best Practices](#testing-best-practices)
     - [Acceptance Test Patterns](#acceptance-test-patterns)
     - [Advanced Statistical Validation](#advanced-statistical-validation)
     - [Bit-Exact Reproducibility](#bit-exact-reproducibility)
     - [Fuzzing and Differential Testing](#fuzzing-and-differential-testing)
   - [Edge Case Testing](#edge-case-testing)
   - [Test Vector Schemas](#test-vector-schemas)
   - [Platform-Specific Validation](#platform-specific-validation)
10. [Reproducibility and Audit Standards](#10-reproducibility-and-audit-standards)
11. [Artifact Standards](#11-artifact-standards)
12. [Liveness and Recovery Standards](#12-liveness-and-recovery-standards)

---

## 1. Cryptographic Security Model Standards

### Algebraic Group Model (AGM)

- **Source**: (Fuchsbauer, Kiltz, & Loss, 2018; Bauer, Fuchsbauer, & Loss, 2020)
- **Scope**: Framework for stating hardness assumptions in pairing-based protocols
- **Application**: Bilinear hardness claims should be framed as AGM security proofs rather than novel named assumptions
- **Key principle**: Security properties should follow from structural constraints (e.g., absence of specific arms) rather than requiring external validation of new assumptions
- **Guidance**: Novel or proposal-specific assumptions should be analyzed within the AGM rather than introduced as named standalone assumptions

### Generic Bilinear Group Model

- **Scope**: Standard lens for reasoning about "no cross-group exponent transfer" in pairing-based cryptography
- **Application**: Understanding when attackers cannot compute target elements without specific handles (e.g., missing CRS elements)
 

---

## 2. Pairing-Based Cryptography Standards

### Groth16 SNARK

- **Source**: (Groth, 2016)
- **Scope**: CRS structure, verification equation, element independence
- **Key elements**:
  - CRS components: α₁, β₂, γ₂, δ₂, and query elements
  - Verification via single pairing product equation (PPE)
  - Independence of CRS elements sampled during setup (normative invariants consolidated in §6)
 

### Groth-Sahai Proof Systems

- **Source**: (Groth & Sahai, 2008)
- **Scope**: One-sided vs. two-sided constructions, pairing product equations (PPE)
- **Key distinctions**:
  - Statement-only bases vs. trapdoored CRS
  - One-sided constructions confine prover randomness to single group
  - How commitments interact with public bases
 

---

## 3. Cryptographic Derivation Standards

### Hash-to-Curve

- **Source**: (Faz-Hernández, Scott, Sullivan, Wahby, & Wood, 2023)
- **Scope**: Deterministic derivation of elliptic curve points with domain separation
- **Application**: Deriving public bases in a way that prevents adversarial bias
- **Key requirement**: Domain-separated tags to ensure independence of different base sets
 

---

## 3A. Cryptographic Primitive Standards

### Poseidon2 (SNARK-Friendly Hash)

- **Source**: (Grassi, Khovratovich, Lüftenegger, Rechberger, Schofnegger, & Walch, 2023)
- **Scope**: SNARK-friendly hashing for in-circuit use and protocol KDFs/commitments
- **Application**:
  - Use clear domain separation for distinct contexts
  - When used in KDFs or commitments, ensure arguments and encodings are unambiguous
- **Key properties**:
  - Domain-separated with ASCII tags
  - Nonce-free mode for key commitment
  - SNARK-friendly (efficient in-circuit representation)
 

### BLS12-381 Curve Family

- **Source**: (Boneh, Drijvers, & Neven, 2022)
- **Scope**: Widely used pairing curve family for production systems
- **Key properties**:
  - Prime-order subgroups (r ≈ 2^255)
  - Efficient Type-3 pairings (e: G₁ × G₂ → G_T)
  - G_T serialization: 576 bytes (12×48B Fp limbs, little-endian)
  - Performance (96 pairings): typically ~50–100ms on broadly "modern" hardware; high-end CPUs (e.g., Apple M1+/Intel 13th-gen) may achieve ~10–50ms. Figures are hardware/library-dependent (see implementation benchmarks in common libraries, e.g., blst/MIRACL, and IETF BLS draft guidance)
- **Application**: Pairing operations, subgroup checks, canonical encodings
- **Validation requirement**: Implementations MUST perform subgroup checks and reject non-canonical encodings

### SHA-256

- **Source**: (National Institute of Standards and Technology, 2015)
- **Scope**: All byte-level context hashing (H_bytes operations)
- **Guidance**: Use consistent byte-level hashing for context binding
- **Application**:
  - Context commitment derivation from well-defined components
  - Public input binding (e.g., hashing of public inputs and parameters)
  - Domain tag construction for distinct context layers
- **Validation requirement**: All H_bytes operations MUST use SHA-256, not other hash functions

---

## 4. Bitcoin Protocol Standards

### BIP-340 (Schnorr Signatures)

- **Source**: (Wuille, Poelstra, & Nick, 2020)
- **Scope**: Schnorr signature specification for Bitcoin
- **Key requirements**:
  - Nonce derivation with strong entropy
  - x-only public keys with even-Y normalization
  - Unique nonces (never reuse)
  - Challenge derivation via tagged hash (per §5A Domain Separation): `c = tagged_hash("BIP0340/challenge", R_x || P_x || m)`
- **Validation requirement**: Canonical encodings and signature normalization per §8 (Canonical Encoding Enforcement)

### BIP-327 (MuSig2)

- **Source**: (Nick, Ruffing, & Feist, 2021)
- **Scope**: Multi-signature protocol using Schnorr signatures
- **Key requirements**:
  - Two-point nonces (R₁, R₂) for security against Wagner's attack
  - High-entropy nonce generation (NOT deterministic-only)
  - Nonce aggregation before signing
  - Session state management
- **Critical note**: Deterministic-only nonces are risky in multi-party settings due to partial state compromise vulnerability
- **Validation requirement**: Verify implementations use a CSPRNG for nonce generation and enforce nonce uniqueness across sessions

### BIP-341 (Taproot)

- **Source**: (Wuille, Nick, & Towns, 2020)
- **Scope**: Taproot transaction structure and script-path validation
- **Key elements**:
  - SIGHASH_ALL with `annex_present = 0` (no annex allowed)
  - Tapleaf hash and version binding in sighash computation
  - NUMS key derivation for key-path disabling (RFC 9380 hash-to-curve)
  - Script-path spending via Merkle proof
- **Validation requirement**: All transactions MUST use SIGHASH_ALL with annex absent

### secp256k1 Elliptic Curve

- **Source**: (Certicom Research, 2010)
- **Scope**: Bitcoin signature layer (BIP-340) and related constructions
- **Reference**: See §3 (Hash-to-Curve) for NUMS derivation patterns

---

## 5. Multi-Party Protocol Standards

### Commit-Reveal Protocols

- **Source**: (Blum, 1983)
- **Scope**: Anti-bias mechanism for multi-party randomness generation
- **Standard construction**:
  - Phase 1 (Commit): Each party posts c_i = Hash(tag || value_i || salt_i)
  - Phase 2 (Reveal): Each party reveals value_i, salt_i; others verify against c_i
- **Application**: Prevents adaptive coordination and grinding attacks on collective randomness
- **Invariants**: Uniqueness of (pk_i, salt_i); binding of revealed values to commitments; freshness (reject replay across instances)
- **Acceptance tests**: See §9 (Acceptance Test Patterns)

### RANDAO (Modern Commit-Reveal)

- Reference: Ethereum RANDAO as a modern example (Ethereum Foundation, n.d.); consider penalties for non-reveal, reveal-order leakage, and incentive alignment

### Ceremony Enforcement Standards (Temporal Ordering)

- **Purpose**: Enforce structural independence through temporal ordering and binding
- **Application**: Multi-phase protocols where later parameters must not influence earlier ones

**Temporal Ordering Requirements**
1. **Phase 1**: Commit to public inputs (e.g., verifying key, public inputs) and derive initial parameters deterministically
2. **Phase 2**: Generate primary cryptographic parameters (e.g., CRS elements)
3. **Phase 3**: Generate independent secondary parameters (e.g., additional CRS instances)
4. **Phase 4**: Freeze and bind all digests in a context commitment
5. **Phase 5**: Parties choose randomness only after all freezes complete

**Freeze-and-Bind Digest Pattern**
- **Purpose**: Propagate parameter commitments through layered hashing
- **Reference**: See §5A (Context Binding Architecture) for the canonical binding sequence
- **Property**: Any parameter substitution changes the final context commitment, invalidating prior bindings

**Participant Disjointness**
- **Pattern**: Parameter generation phases use disjoint participant sets (normative requirement in §6)
- **Rationale**: Prevents a single party from correlating parameters across phases
- **Validation**: Ceremony transcripts SHOULD document participant sets per phase

 

---

## 5A. Entropy and Nonce Management Standards

### OS-Level CSPRNG Requirements

- **Sources**: OS-specific entropy APIs
  - **Linux**: `getrandom(2)` syscall with GRND_RANDOM flag
  - **BSD/macOS**: `getentropy(3)` function
  - **Windows**: `BCryptGenRandom` with BCRYPT_USE_SYSTEM_PREFERRED_RNG
- **Application**:
  - Generate per-session randomness using OS CSPRNGs
  - Ensure nonces are unique per session and never reused
  - For multi-party settings, agree on common randomness and verify consistency

### Domain Separation Architecture

- **Principle**: All hash/KDF operations SHOULD use unique, context-specific ASCII domain tags
- **Guidance**:
  - Use distinct tags for different layers (e.g., context, KEM/KDF, DEM, protocol transcripts)
  - Follow BIP-340’s tagged hash construction where applicable (Wuille, Poelstra, & Nick, 2020); for non-Bitcoin contexts, use a similar tag-prefix approach
  - Ensure tags are versioned and recorded for auditability

### Context Binding Architecture

- **Scope**: Layered, acyclic commitments that bind protocol parameters and inputs
- **Key properties**:
  - Deterministic computation of context commitments from well-defined components
  - No forward/self-references that introduce circular dependencies
  - Changes to any bound component MUST change the final context commitment
- **Composition note**: See §8 (Formal Proof Standards → IND-CPA + INT-CTXT Composition) for guidance on secure composition; binding and acyclicity support integrity when combined with authenticated contexts
- **Validation requirement**: Test that variations to any bound component result in distinct commitments; ensure no cyclic dependencies

---

## 6. Specification Language Standards

### RFC 2119 (Normative Language)

- **Source**: (Bradner, 1997)
- **Scope**: Precise normative language (MUST, SHOULD, MAY)
- **Application**: Security-critical properties should be stated as normative requirements, not suggestions
- **Key words**:
  - **MUST**: Absolute requirement
  - **MUST NOT**: Absolute prohibition
  - **SHOULD**: Recommended but not absolute
  - **SHOULD NOT**: Not recommended but not prohibited
  - **MAY**: Truly optional

### Explicit Security Invariants

- **Principle**: Security-critical structural properties must be stated explicitly as normative requirements
- **Application**: Properties that emerge from design choices (e.g., "γ₂ MUST NOT be included in armed set") should be documented as MUSTs, not left implicit
- **Examples**:
  - Deterministic derivations MUST be specified for targets derived from public inputs
  - Bases and secondary parameters MUST be fixed before any prover randomness is chosen
  - Trapdoored elements (e.g., γ₂) MUST NOT be included where they would violate independence
  - Disjointness: CRS participant sets MUST be disjoint across phases
- **Validation requirement**: Check that all security assumptions are stated as normative requirements

---

## 7. Issue Classification Standards

### Soundness vs. Liveness vs. Fairness

- **Soundness**: Adversary cannot forge proofs or break cryptographic guarantees
  - Example: Attacker finalizes spend without valid proof
  - Mitigation: Cryptographic hardness assumptions, proof verification
- **Liveness**: Protocol can make progress; no deadlocks or griefing that halt execution
  - Example: Malformed arming package causes decapsulation to always fail
  - Mitigation: PoCE verification, mandatory timeout paths
- **Fairness**: Parties cannot bias outcomes or gain unfair advantages
  - Example: Last armer grinds randomness to weaken KEM
  - Mitigation: Commit-reveal protocols, randomness beacons
- **Application**: Correctly classify issues by impact type; fairness issues require different mitigations than soundness breaks
 
**Severity heuristics**
- Soundness griefing → Critical severity (breaks core guarantees)
- Liveness griefing → High severity (stalls progress without breaks)
- Fairness griefing → Medium severity (bias/incentive issues without breaks)

---

## 7A. DoS Prevention and Resource Limits

### Proof/Attestation Complexity Bounds

- **Scope**: Prevent DoS via oversized proof structures
- **Limits**:
  - m₁ + m₂ ≤ 96 (total G₁ + G₂ commitments)
  - Typical Groth16→GS encodings: 20-40 pairings
- **Rationale**: Balance security (sufficient expressiveness) with DoS resistance (bounded computation)
- **Application**: Reject attestations exceeding bounds before pairing computation
- **Performance note**: See §3A (BLS12-381 Curve Family) for expected pairing performance
- **Validation requirement**: Test that implementations reject attestations with 97+ commitments

### Griefing Prevention

- **Scope**: Liveness protection against malicious armers
- **Key checks**:
  - Per-share validation: Reject identity/invalid elements early (e.g., points at infinity)
  - Aggregated validation: Reject aggregate values equal to identity that bypass intended checks
  - Mandatory timeout/abort path: Include a production timeout path to ensure liveness
- **Rationale**: If armer posts malformed artifacts passing PoK but failing PoCE-B at decap, timeout prevents indefinite lock
- **Validation requirement**: Exercise the edge cases cataloged in [§9, Edge Case Testing](#edge-case-testing), with emphasis on null-share detection, collusive cancellation, and decapsulation failure paths

---

## 8. Implementation Security Standards

### Attack Construction Standards (Adversarial Analysis)

- **Purpose**: Systematic methodology for constructing concrete attacks during security review
- **Application**: Security reviewers should demonstrate concrete attack algorithms, not just abstract vulnerabilities

**Attack Anatomy Template (3-Phase)**
1. **Setup Phase**: Adversary prepares malicious artifacts that pass public checks
2. **Execute Phase**: Honest parties proceed using adversary's artifacts
3. **Failure Phase**: Attack manifests at a later protocol stage (e.g., decapsulation fails after expensive computation)

**Coalition Strategy Patterns**
- **Definition**: Multi-party coordination to achieve outcomes impossible for a single adversary
- **Example**: Collusive randomness cancellation (e.g., multiplicative inverses yielding neutral aggregate)
- **Detection Challenge**: Per-party checks pass but aggregate invariants are violated
- **Mitigation Pattern**: Commit-reveal protocols enforcing independence before parameter revelation

**Extraction Algorithm Templates**
- **Key Recovery**: O(1) extraction from protocol transcript analysis
  - Example: MuSig2 nonce reuse yields `x = (s₁ - s₂) · (c₁ - c₂)⁻¹ mod n`
  - Requirement: Two signatures with identical R over different messages
- **Witness Leakage**: Information-theoretic bounds on side-channel leakage
  - Example: Timing leakage `I(W; T_decap) ≈ log₂(#pairing-configs)`
  - Quantification: 4560 valid (m₁, m₂) pairs under m₁ + m₂ ≤ 96

**Griefing Attack Taxonomy**
- See §7 (Soundness vs. Liveness vs. Fairness) for definitions and severity guidance

**Validation Requirement**: Security reports MUST include:
- Adversary capabilities (honest-but-curious, malicious, coalition size k-of-n)
- Success probability ε and computational cost (group operations, pairings)
- Attack detection methods (statistical tests, binding failures)
- Concrete attack algorithm pseudocode

### Formal Proof Standards (Cryptographic Reductions)

- **Purpose**: Standardize reduction proofs for cryptographic security claims
- **Application**: Cryptographic security claims should include reductions to well-studied assumptions

**Hybrid Reduction Methodology**
- **Pattern**: Embed a single-instance challenge into a random index j among n
  - Simulator leverages random oracle queries to extract unknown component
  - Security amplification: ε′ ≤ n·ε (linear advantage loss)
- **Requirement**: Specify challenge distribution, simulator strategy, and advantage bound

**Telescoping Lemma for PPE**
- **Statement**: For construction c = Γu, prove ∏_ℓ e(C_ℓ, U_ℓ) = e(A, B)
- **Proof sketch**:
  1. Expand products in exponent space
  2. Sum over ℓ: Σ_ℓ c_ℓ Γ_{ℓj} = (Γ^T Γ u)_j
  3. Telescoping yields u_j for all j, thus e(A, Σ_j u_j Y_j) = e(A, B)
- **Application**: Binding DLREP proofs to PPE verification

**Algebraic Independence Formalization**
- **Span notation**: e(G_G16) ∉ span_Zr{e(e(V_k, U_j))}
- **Purpose**: Checkable independence property for formal proofs
- **Implication**: If e(G_G16) lies in GS pairing span, masks suffice to compute M without proof

**IND-CPA + INT-CTXT Composition**
- **Theorem** (Bellare-Namprempre): IND-CPA + INT-CTXT ⇒ IND-CCA in Random Oracle Model
- **Requirements**:
  - IND-CPA: Pseudorandom keystream with single-use K_i
  - INT-CTXT: Tag forgery requires inverting PRF (negligible success)

### Constant-Time Execution

- **Sources**: (Almeida, Barbosa, Barthe, Dupressoir, & Emmi, 2016); CROCS (n.d.)
- **Scope**: Extended beyond standard crypto to protocol-specific operations
- **Mandatory constant-time operations**:
  - **Decapsulation (KEM)**: No early returns; verify all equations then branch on final result
  - **Pairing loops**: GS-PPE product computation must not leak witness structure
  - **Scalar operations**: All field arithmetic and group operations
  - **DEM decryption**: Constant-time even for incorrect keys (no early abort on auth failure)
- **Multi-instance caution**: Avoid cache-tuneable table leakage across contexts; table lookups must not use secret-dependent indices
- **Key tools**:
  - **dudect**: Statistical timing leak detection
  - **ctgrind**: Valgrind-based constant-time verification
  - **ct-verif**: Formal verification of constant-time properties
- **Test vectors**: Should include:
  - Fixed vs. variable witness sizes
  - Different verifying keys (VK variations)
  - Multiple protocol contexts/commitments (cross-context timing)
- **Validation requirement**: Run dudect on decapsulation with varying inputs; confirm no timing correlation

### Canonical Encoding Enforcement

- **Sources**: (Wuille, Poelstra, & Nick, 2020); (Boneh, Drijvers, & Neven, 2022)
- **Cross-reference**: See [§3A, BLS12-381 Curve Family](#bls12-381-curve-family) for serialization formats, subgroup checks, and normalization rules
- **Test focus**:
  - Exercise rejection paths for non-canonical encodings and wrong-subgroup elements
  - Verify rejection sampling is in place where scalars derive from hashes
  - Confirm aggregate nonce normalization is enforced prior to signing
- **Validation requirement**: Implementations MUST include automated tests that cover each rejection path above

---

## 9. Verification and Testing Standards

### Testing Best Practices

#### Acceptance Test Patterns

- Valid/Invalid/Replay trinity for protocol artifacts:
  - Valid: Correct artifacts pass verification
  - Invalid: Malformed artifacts rejected at the correct stage
  - Replay: Reused artifacts detected and rejected
 - Example: Commit–reveal tests (see §5 Commit‑Reveal Protocols)

#### Advanced Statistical Validation

- Purpose: Rigorous statistical methods for validating security properties

**TOST (Two One-Sided Tests) for Timing Equivalence**
- Replaces: Welch's t-test (tests difference, not equivalence)
- Purpose: Statistically validate constant-time execution
- Method:
  1. Define equivalence margin δ (e.g., 5% of mean timing)
  2. Test H₀: |μ₁ - μ₂| ≥ δ vs. H₁: |μ₁ - μ₂| < δ
  3. Reject null ⇒ timings are statistically equivalent
- Application: Validate decapsulation timing across witness sizes, verifying keys, and context commitments
- Acceptance: p < 0.05 for both one-sided tests

**Statistical Significance Thresholds**
- Uniqueness Tests: 1000+ iterations to detect collisions with high confidence
  - Example: MuSig2 nonce uniqueness across contexts
  - Requirement: Zero collisions in 10³-10⁴ samples
- Timing Precision Assumptions: δ_time << σ_pairing/√m
  - Measurement precision must be much smaller than timing variance
  - Typical: δ_time < 100ns for ~10ms pairing operations

#### Bit-Exact Reproducibility

- Requirement: Test vectors with bit-exact expected outputs
- Purpose: Detect implementation drift, ensure cross-compatibility
- Coverage:
  - KDF outputs (from specified inputs including context commitments/instance digests)
  - DEM encryption (ciphertext and tags from plaintext, key, and associated data)
  - Serialization (ser_G_T must produce identical byte strings)
- Validation: Implementations differing on any test vector are non-compliant

#### Fuzzing and Differential Testing

- Sources: (Glénaz, 2024); Quarkslab (n.d.)
- Scope: Automated vulnerability detection in cryptographic primitives
- Key tools:
  - Cryptofuzz: Differential testing across multiple crypto libraries
  - AFL++: Coverage-guided fuzzing
- Application: Test edge cases like malformed inputs, degenerate values, multi-instance scenarios
- Coverage areas:
  - Ciphertext validation
  - Nonce generation
  - Pairing computations
  - GS attestation parsing
- Validation requirement: Achieve >95% code coverage in security-critical paths

### Edge Case Testing

- **Scope**: Systematic testing of boundary conditions and degenerate inputs
- **Key categories**:
  - **Degenerate inputs**: R=1 (identity in G_T), identity elements in G₁/G₂, zero values
  - **Malformed data**: Invalid ciphertexts, corrupted proofs, truncated messages
  - **Multi-instance scenarios**: Hash collisions, nonce reuse, context commitment overlap
  - **Boundary values**: Maximum field elements, curve order boundaries, field characteristic
- **Application**: Test vectors should cover both valid and invalid inputs with expected outcomes
- **Examples**:
  - Degenerate targets (e.g., identity elements)
  - Null shares or aggregate identities in multi-party computations
  - Session randomness reuse
  - Exceeding attestation complexity bounds

### Test Vector Schemas

- Scope: Structured, parseable test data with documented format
- Requirements:
  - JSON schema notation at top of file
  - Clear input/output specification
  - Expected outcomes for both valid and invalid cases
  - Edge case annotations
- Application: All test vectors should be machine-parseable and human-readable
- Examples:
  - Valid/invalid ciphertext or package structures (with/without correct proofs)
  - Valid/invalid proof/attestation encodings (various circuit sizes)
  - Edge cases (degenerate values, null shares, nonce reuse)


### Platform-Specific Validation

- **Scope**: Verification across deployment environments with varying constraints
- **Key considerations**:
  - **Hardware wallets**: Memory limits (typically <100KB RAM), computational constraints
  - **Mobile devices**: Timing consistency, power constraints, thermal throttling
  - **Embedded systems**: ARM architectures, limited entropy sources, no OS CSPRNG
- **Application**: Implementation notes should document platform-specific requirements and limitations
- **Concerns**:
  - Pairing computation on resource-constrained devices
  - CSPRNG availability and quality of entropy sources
  - Constant-time execution on ARM (cache architecture differences)

---

## 10. Reproducibility and Audit Standards

### Reproducible Builds

- **Source**: (Open Technology Fund, 2025)
- **Scope**: Enabling independent verification without accessing confidential inputs
- **Key principles**:
  - Outputs-only artifacts (no raw confidential data)
  - Traceability per §5A (Context Hashing Architecture) and SHA256 of reviewed sources
  - Stable naming and versioning
  - Minimal, self-contained artifact sets
- **Application**: All security claims should be independently verifiable through public artifacts
 

### Security Audit Best Practices

- **Source**: (a16z crypto, 2025)
- **Scope**: Comprehensive cryptographic security review methodology
- **Key elements**:
  - Formal verification tools integration
  - Automated test generation
  - Schema validation for test vectors
  - Clear acceptance criteria mapping
  - Link/anchor validation for cross-references
- **Application**: Validation workflows should include automated checks and manual review gates
 

---

## 11. Artifact Standards

 

### Automation and Tooling

- **Scope**: Scripted artifact generation and validation
- **Key components**:
  - Folder initialization scripts
  - Hash computation automation
  - Index generation and updates
  - Link/anchor validation
- **Application**: Minimize manual errors through automation while maintaining auditability

---

## 12. Liveness and Recovery Standards

- **Reference**: Mandatory timeout paths and CSV guidance are defined in [§7A, Griefing Prevention](#griefing-prevention)
- **Future scope**: Additional liveness controls will be added here as the protocol matures

---

## References

Almeida, J. B., Barbosa, M., Barthe, G., Dupressoir, F., & Emmi, M. (2016). Verifying constant-time implementations. In *25th USENIX Security Symposium* (pp. 53-70). USENIX Association.

a16z crypto. (2025, May 8). Security audit secrets to success. a16z crypto. https://a16zcrypto.com/posts/article/security-audit-best-practices/

Bauer, B., Fuchsbauer, G., & Loss, J. (2020). A classification of computational assumptions in the algebraic group model. In *Advances in Cryptology – CRYPTO 2020* (pp. 121-151). Springer. https://doi.org/10.1007/978-3-030-56784-2_5

Bellare, M., & Namprempre, C. (2000). Authenticated encryption: Relations among notions and analysis of the generic composition paradigm. In *Advances in Cryptology — ASIACRYPT 2000* (pp. 531-545). Springer. https://doi.org/10.1007/3-540-44448-3_41

Blum, M. (1983). Coin flipping by telephone: A protocol for solving impossible problems. *ACM SIGACT News, 15*(1), 23-27. https://doi.org/10.1145/1008908.1008911

Boneh, D., Drijvers, M., & Neven, G. (2022). *BLS signature scheme* (draft-irtf-cfrg-bls-signature-05). Internet Engineering Task Force. https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-bls-signature-05

Bossuat, A., Goudarzi, A., Loayza-Meneses, L., & Quiniou, R. (2024, September 24). crypto-condor: A test suite for cryptographic primitives. Quarkslab's blog. https://blog.quarkslab.com/crypto-condor-a-test-suite-for-cryptographic-primitives.html

Bradner, S. (1997). *Key words for use in RFCs to indicate requirement levels* (RFC 2119). Internet Engineering Task Force. https://doi.org/10.17487/RFC2119

Certicom Research. (2010). *SEC 2: Recommended elliptic curve domain parameters* (Version 2.0). Standards for Efficient Cryptography Group. https://www.secg.org/sec2-v2.pdf

CROCS. (n.d.). Constant-time tools. Masaryk University. Retrieved October 28, 2025, from https://crocs-muni.github.io/ct-tools/

Ethereum Foundation. (n.d.). RANDAO. Ethereum.org. Retrieved October 28, 2025, from https://ethereum.org/en/glossary/#randao

Faz-Hernández, A., Scott, S., Sullivan, N., Wahby, R. S., & Wood, C. A. (2023). *Hashing to elliptic curves* (RFC 9380). Internet Engineering Task Force. https://doi.org/10.17487/RFC9380

Fuchsbauer, G., Kiltz, E., & Loss, J. (2018). The algebraic group model and its applications. In *Advances in Cryptology – CRYPTO 2018* (pp. 33-62). Springer. https://doi.org/10.1007/978-3-319-96881-0_2

Glénaz, C. (2024, October 3). Differential fuzzing for cryptography. Quarkslab's blog. https://blog.quarkslab.com/differential-fuzzing-for-cryptography.html

Grassi, L., Khovratovich, D., Lüftenegger, R., Rechberger, C., Schofnegger, M., & Walch, R. (2023). Poseidon2: A faster version of the Poseidon hash function. *IACR Cryptology ePrint Archive, 2023*/323. https://eprint.iacr.org/2023/323

Groth, J. (2016). On the size of pairing-based non-interactive arguments. In *Advances in Cryptology – EUROCRYPT 2016* (pp. 305-326). Springer. https://doi.org/10.1007/978-3-662-49890-3_12

Groth, J., & Sahai, A. (2008). Efficient non-interactive proof systems for bilinear groups. In *Advances in Cryptology – EUROCRYPT 2008* (pp. 415-432). Springer. https://doi.org/10.1007/978-3-540-78967-3_24

National Institute of Standards and Technology. (2015). *Secure hash standard (SHS)* (FIPS PUB 180-4). U.S. Department of Commerce. https://doi.org/10.6028/NIST.FIPS.180-4

Nick, J., Ruffing, T., & Feist, Y. (2021). MuSig2: Simple two-round Schnorr multi-signatures (BIP 327). Bitcoin Improvement Proposals. https://bips.dev/327/

Open Technology Fund. (2025, January 14). Reproducible builds baseline security assurance. https://www.opentech.fund/wp-content/uploads/2025/05/SRL-reproducible_builds-baseline_assurance-report-final.pdf

Quarkslab. (n.d.). Crypto-Condor: A test suite for cryptographic primitives. Retrieved October 28, 2025, from https://github.com/quarkslab/crypto-condor

Wuille, P., Nick, J., & Towns, A. (2020). Taproot: SegWit version 1 spending rules (BIP 341). Bitcoin Improvement Proposals. https://bips.dev/341/

Wuille, P., Poelstra, A., & Nick, J. (2020). Schnorr signatures for secp256k1 (BIP 340). Bitcoin Improvement Proposals. https://bips.dev/340/

---

## License

This document is licensed under CC-BY 4.0.
