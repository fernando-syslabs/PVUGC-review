# Appendix — Consolidated Debate References

Purpose: Compact, atomic references for debate materials cited by PVUGC peer review reports. Entries are self-contained, with stable codes and anchors, and do not link back to original debate files.

---

## Conventions

- Atomic entries: self-contained; no links to original `issue-XX-...md` files.
- Compactness: 200–400 words per entry; minimal quotes; prefer bullets.
- Coding: `AII.RNN` where `II`=issue (01..99), `R`=role code, `NN`=sequential per issue (01..99).
  - Role codes: `M1`/`M2`/... = Mathematician #n; `CR` = Crypto Reviewer; `LR` = Lead Reviewer (extend with concise two-letter tags as needed).
- Anchors: issue section `#issue-xx`; entry anchor `#aII-rnn` (lowercase, hyphenated).
- Citations (from reports): “See Appendix A04.M101 (APPENDIX-issue-debates.md#a04-m101).”

---

## Table of Contents (Issues)

- [Issue 02](#issue-02)
- [Issue 03](#issue-03)
- [Issue 04](#issue-04)
- [Issue 05](#issue-05)
- [Issue 06](#issue-06)
- [Issue 07](#issue-07)
- [Issue 08](#issue-08)
- [Issue 09](#issue-09)
- [Issue 10](#issue-10)
- [Issue 11](#issue-11)

---

<a id="issue-02"></a>
## Issue 02

Add entries as A02.RNN (e.g., A02.M101, A02.CR02). Keep each entry atomic and concise.

<a id="a02-m101"></a>
### A02.M101 — Binding CRS + Multi-CRS validation (Mathematician)

Role: M1

Context: Validation that a binding GS CRS removes commitment malleability and that Multi-CRS AND-ing adds defense-in-depth for the attestation layer.

Spec refs (indicative): [`PVUGC-2025-10-05.md §10 Binding CRS Requirements (SXDH/DLIN)`](../PVUGC-2025-10-05.md#10-binding-crs-requirements-sxdh/dlin) (§6 WE/KEM; §7 security; Production Profile multi-CRS); [`report-update-2025-10-07/PVUGC-002-multi-crs-anding.md`](../report-update-2025-10-07/PVUGC-002-multi-crs-anding.md)

Summary
- Binding CRS enforces unique openings for GS commitments, blocking "product forcing" attacks.
- Multi-CRS AND-ing strengthens security by requiring simultaneous validity across independent CRS instances.
- Soundness relies on GS binding (e.g., SXDH/DLIN) and the PVUGC verifier’s PPE structure.

Key Arguments/Findings
- Binding CRS: With a computationally binding CRS, any accepting GS proof implies a unique committed witness; commitments cannot be reinterpreted to satisfy the PPE for false statements.
- AND-of-n CRS: Independent transcripts and separate mask derivations ensure an adversary cannot exploit structure in a single CRS; success would require breaking all instances. Formal bounds give linear advantage loss in standard reductions.
- Practical outcome: The v2.0 normative requirements (binding CRS + AND-ing) close the v1.0 malleability gap and provide robust, auditable conditions for implementers.

<a id="a02-cr03"></a>
### A02.CR03 — DLREP_B variable‑portion coverage (Crypto Reviewer)

Role: CR

Context: DLREP_B subtracts β₂ and the first query Y₀ so the proof binds only the variable portion of B.

Summary
- Proof soundness requires that fixed parts (β₂, Y₀) are handled outside DLREP_B; responses must correspond to the variable coefficients b_j.
- Prevents malleability via reinterpreting fixed contributions as variable terms.

Key Arguments/Findings
- Verifier reconstructs B’s fixed components and rejects if DLREP_B responses don’t match.
- Tie proof enforces ∑C_ℓ alignment with A to bind aggregate coefficients.

<a id="a02-cr02"></a>
### A02.CR02 — Multi-CRS digest pinning and context binding (Crypto Reviewer)

Role: CR

Context: Interplay between GS_instance_digest (binding CRS digests) and layered context binding (ctx_core → arming_pkg_hash → ctx_hash).

Spec refs (indicative): [`PVUGC-2025-10-05.md §10 Binding CRS Requirements (SXDH/DLIN)`](../PVUGC-2025-10-05.md#10-binding-crs-requirements-sxdh/dlin) (§3 context binding; §6 KDF usage of ctx_hash; production profile multi-CRS).

Summary
- Pinning multiple independent GS CRS digests inside GS_instance_digest and header_meta ensures CRS substitution changes arming_pkg_hash and thus ctx_hash.
- ctx_hash inclusion in KDF makes ciphertext keys context-specific; cross-context reuse fails decryption.
- Multi-CRS AND-ing ensures that even if one CRS is compromised, substitution cannot preserve ctx_hash without detection.

Key Arguments/Findings
- Binding propagation: GS_instance_digest → header_meta → arming_pkg_hash → ctx_hash produces a one-way dependency chain; any CRS change invalidates downstream hashes.
- KDF salting with H_bytes(ctx_hash) prevents cross-context replay of valid artifacts.
- Operational note: publish and verify both CRS digests; auditors should recompute ctx_* values from artifacts to detect tampering.

<a id="issue-03"></a>
## Issue 03

Add entries as A03.RNN (e.g., A03.M101, A03.CR02). Keep each entry atomic and concise.

<a id="a03-m201"></a>
### A03.M201 — Independence formalization and ceremony rationale (Mathematician)

Role: M2

Context: Formalizing algebraic independence of the KEM target from GS pairing spans; arguing why a mere “MUST” clause is insufficient and proposing a Normative Setup Ceremony as enforcement.

Spec refs (indicative): [`PVUGC-2025-10-20.md §1 Introduction`](../PVUGC-2025-10-20.md#1-introduction) (§6, §8); [Appendix A11.M201](APPENDIX-issue-debates.md#a11-m201) (Formalization); [Appendix A11.CR02](APPENDIX-issue-debates.md#a11-cr02) (Catalog); [Appendix A11.M203](APPENDIX-issue-debates.md#a11-m203) (Commit‑reveal mitigation)

Summary
- Independence must be a mathematical invariant, not just a procedural requirement; a “MUST” clause does not enforce algebraic independence.
- Formalization: let e(G_G16) be the exponent vector; independence requires e(G_G16) ∉ span_Zr{ e(e(V_k, U_j)) }.
- Failure implies computability of M_i = G_G16^{ρ_i} from masks, breaking no‑proof‑spend.
- Normative Setup Ceremony with temporally ordered commitments enforces independence in practice.

Key Arguments/Findings
- Span definition makes the property checkable in proofs and clarifies attack surface.
- If e(G_G16) lies in the GS pairing span, masks suffice to compute M without any proof.
- Ceremony: (1) commit to x, (2) Groth16 CRS, (3) independent GS CRS instances, (4) freeze and bind all digests in context; armers choose ρ only after freezes.
- Generic Group Model reasoning: independently sampled CRS and precommitted x make structural correlations negligibly likely.

<a id="a03-m202"></a>
### A03.M202 — Multi‑CRS amplification and independence obligations (Mathematician)

Role: M2

Context: Security amplification for AND‑of‑n independent CRS and formal obligations for independence proofs and ceremony verification.

Spec refs (indicative): [`PVUGC-2025-10-05.md §10 Binding CRS Requirements (SXDH/DLIN)`](../PVUGC-2025-10-05.md#10-binding-crs-requirements-sxdh/dlin) (production profile: multi‑CRS AND‑ing; context binding)

Summary
- Multi‑CRS AND‑ing yields linear security amplification: adversary advantage ε′ ≤ n·ε in a standard hybrid reduction.
- Formal obligations: define independence in exponent space; specify ceremony transcripts and bindings; verify independent CRS and frozen parameters in context.
- Practical audit: bind CRS digests and commit_x into ctx_core/header_meta; publish public transcripts for third‑party verification.

Key Arguments/Findings
- Reduction: embed a single‑instance challenge into a random index j among n; simulator leverages RO queries to extract the unknown component; yields ε′ ≤ n·ε.
- Independence obligations: (i) algebraic span definition, (ii) temporal ordering and participant disjointness, (iii) domain separation and binding, (iv) subgroup/serialization hygiene.
- Defense‑in‑depth: AND‑of‑n neutralizes single‑instance structure; ceremony reduces correlation risk to negligible under standard models.

<a id="issue-04"></a>
## Issue 04

Add entries as A04.RNN (e.g., A04.M101, A04.CR02). Keep each entry atomic and concise.

<a id="a04-m201"></a>
### A04.M201 — Liveness griefing formalization (Mathematician)

Role: M2

Context: Formalizing the liveness griefing attack against a non-hardened PoCE-A circuit that proves mask/commitment linkage but does not bind ciphertext to the derived KEM key.

Spec refs (indicative): [`PVUGC-2025-10-20.md §9 Arming Ceremony Checks`](../PVUGC-2025-10-20.md#9-arming-ceremony-checks) (Ceremony rule - pre-signing gates)

Summary
- Non-hardened PoCE-A verifies mask correctness and adaptor share linkage but omits ciphertext binding, enabling costless publication of garbage ciphertexts.
- Attack outcome: all arm-time checks pass; later, decapsulation fails silently after expensive proving, causing liveness failure and griefing.
- Hardened PoCE-A fixes: derive K in-circuit from M_i and bind (ct, τ) with Poseidon2 over AD_core.

Key Arguments/Findings
- Attack anatomy: publish valid D1/D2/T_i and a PoCE-A proof; set ct, τ as random; verification passes at arm time; decapper derives K_i later and fails tag check.
- Security requirement: PoCE-A must prove K derivation and DEM correctness in-circuit to make encryption publicly verifiable at arm time.
- Practical guidance: treat (ct, τ) as public inputs to PoCE-A; use the same domain separation and ser_GT as KDF; enforce ρ_link and T_i consistency.

<a id="a04-m202"></a>
### A04.M202 — Hardened PoCE-A refinements (Mathematician)

Role: M2

Context: Review and refinement pass over the hardened PoCE-A, with emphasis on correctness constraints, timing, and implementation guidance.

Spec refs (indicative): [`PVUGC-2025-10-20.md §1 Introduction`](../PVUGC-2025-10-20.md) (§8 hardened PoCE-A, constraints 6–8, lines ~220–224).

Summary
- Confirms hardened PoCE-A eliminates the liveness griefing vector by binding K and DEM outputs in-circuit.
- Recommends explicit checks (e.g., ρ_i ≠ 0, canonical ser_GT, per-share T_i ≠ O) and timing (verify PoCE before pre-signing).
- Notes linear security model and implementation verification needs (tests ensuring ct/τ binding and tag checks match external decap).

Key Arguments/Findings
- Constraint listing: (i) K_i = Poseidon2(ser_GT(M_i)||H_bytes(ctx_hash)||GS_instance_digest), (ii) ct_i = (s_i|h_i) ⊕ Poseidon2(K_i, AD_core), (iii) τ_i = Poseidon2(K_i, AD_core, ct_i).
- Public verifiability: with hardened constraints, arm-time verification suffices to exclude malformed ciphertexts before costly proving.
- Implementation notes: enforce constant-time DEM/KDF; unify domain tags; publish test vectors to prevent drift.

<a id="a04-cr03"></a>
### A04.CR03 — Anchorless arms (leakage prevention) (Crypto Reviewer)

Role: CR

Context: Arming publishes no “anchor” base (e.g., T₀). Anchorless design avoids structural leakage that could weaken gating/PoCE analyses.

Summary
- Statement‑only G2 bases (β₂ and query elements), aggregated via Γ; no extra anchor base is introduced or published.
- Prevents attackers from leveraging a fixed public anchor in KEM algebra.

Key Arguments/Findings
- Aligns with one‑sided design: randomness lives on G1 commitments; G2 side remains fixed to VK‑derived bases.
- Tests and API ensure no anchor term appears in arms; documentation should state this normatively.

<a id="a04-m203"></a>
### A04.M203 — Telescoping lemma for PPE (Mathematician)

Role: M2

Context: Formal statement that ∏_ℓ e(C_ℓ, U_ℓ) = e(A, B) under the PVUGC construction of C_ℓ and U_ℓ.

Statement
- Let A ∈ G1, and let Y = (Y_0 = β₂, Y_1, …, Y_m) ⊂ G2 be the Groth16 B‑query bases.
- Let Γ ∈ {−1,0,1}^{L×(m+1)} be the FS‑derived matrix and define row bases U_ℓ = ∑_{j=0}^m Γ_{ℓj}·Y_j.
- Let u ∈ Z_r^{m+1} be the coefficient vector (u_0 = 1 for β₂, u_1 = 1 for Y_0, and u_{j>1} = b_{j−1}). Define commitments C_ℓ = (Γ u)_ℓ · A.
- Then ∏_{ℓ=1}^L e(C_ℓ, U_ℓ) = e(A, ∑_{j=0}^m u_j Y_j) = e(A, B).

Proof sketch
- Expand products in exponent space: e(c_ℓ A, ∑_j Γ_{ℓj} Y_j) = e(A, ∑_j c_ℓ Γ_{ℓj} Y_j).
- Sum over ℓ: ∑_ℓ c_ℓ Γ_{ℓj} = (Γ^T c)_j = (Γ^T Γ u)_j.
- With construction c = Γ u (component‑wise), telescoping yields ∑_ℓ c_ℓ Γ_{ℓj} = u_j for all j, so the product equals e(A, ∑_j u_j Y_j) = e(A, B).
- The DLREP_B and tie proofs enforce that the committed c and aggregate ∑ C_ℓ are consistent with A and B, making the construction binding.

<a id="issue-05"></a>
## Issue 05

Add entries as A05.RNN (e.g., A05.M101, A05.CR02). Keep each entry atomic and concise.

<a id="a05-m201"></a>
### A05.M201 — Layered context binding validation (Mathematician)

Role: M2

Context: Formal validation that the three-layer binding (ctx_core, arming_pkg_hash, presig_pkg_hash) with domain tags prevents CRS substitution and cross-context reuse.

Spec refs (indicative): [`PVUGC-2025-10-20.md §1 Introduction`](../PVUGC-2025-10-20.md) (§3 context binding; §6 KDF uses ctx_hash).

Summary
- The layered hash design propagates bindings from artifacts (D1/D2/T_i/ct/τ) and CRS digests to ctx_hash, making tampering detectable.
- Domain separation for ctx_* eliminates unintended collisions across namespaces.
- Replay across contexts is prevented by txid_template’s inclusion in ctx_core and KDF salting with ctx_hash.

Key Arguments/Findings
- header_meta includes GS_instance_digest, which binds CRS; changing CRS modifies header_meta → arming_pkg_hash → ctx_hash.
- presig_pkg_hash binds MuSig2 artifacts (m, T, R, signer_set, coeffs) to the same context.
- With SHA-256 for H_bytes, practical collision resistance is sufficient for operational security.

<a id="a05-m202"></a>
### A05.M202 — Epoch nonce and context uniqueness (Mathematician)

Role: M2

Context: Recommendation to add epoch_nonce to ctx_core to elevate from practical uniqueness to clean formal provability.

Spec refs (indicative): [`PVUGC-2025-10-05.md §10 Binding CRS Requirements (SXDH/DLIN)`](../PVUGC-2025-10-05.md#10-binding-crs-requirements-sxdh/dlin) (§3 ctx_core fields; domain tags).

Summary
- Even with txid_template binding, explicit epoch_nonce simplifies uniqueness arguments and avoids reliance on transaction-structure assumptions.
- Improves formal proofs of Context Uniqueness and simplifies testing and auditability.

Key Arguments/Findings
- Adding nonce to ctx_core preserves acyclicity and domain separation while ensuring per-instance uniqueness.
- Minimal overhead, significant clarity gains for formal verification.

<a id="a05-m203"></a>
### A05.M203 — Appendix notes on ctx_core and nonce (Mathematician)

Role: M2

Context: Compact extraction of mathematical appendix guidance (ctx_core bindings, nonce rationale).

Spec refs (indicative): Appendix lines ~94–104.

Summary
- ctx_core should bind vk_hash, H(x), tapleaf hash and version, txid_template, and path_tag; adding nonce yields explicit uniqueness.
- The nonce is orthogonal to tx template and does not introduce cycles.

Key Arguments/Findings
- Formal proofs prefer explicit uniqueness fields; a nonce provides it without weakening bindings.


<a id="issue-06"></a>
## Issue 06

Add entries as A06.RNN (e.g., A06.M101, A06.CR02). Keep each entry atomic and concise.

<a id="a06-m201"></a>
### A06.M201 — First‑order validation and collusive discovery (Mathematician)

Role: M2

Context: Confirms v2.0 first‑order guards and introduces the second‑order vulnerability: collusive randomness cancellation.

Spec refs (indicative): [`PVUGC-2025-10-05.md §7 GS PPE Verification (96 pairings)`](../PVUGC-2025-10-05.md#7-gs-ppe-verification-96-pairings) (§6 degenerate guards; §8 assertion G_G16 ≠ 1).

Summary
- Validated: G_G16 ≠ 1 assertion (arm-time + PoCE), per‑share and aggregate T ≠ O, ρ ≠ 0, canonical ser_GT, GS size bounds, subgroup checks.
- Discovered: coalitions can choose ρ_i such that ∏ ρ_i ≡ 1 (mod r), passing per‑share checks but collapsing aggregate randomness.
- Proposed mitigation: commit‑reveal scheme for ρ_i to enforce independence and prevent targeted products.

Key Arguments/Findings
- First‑order guards are necessary and sufficient for single‑party degeneracies.
- Aggregate degeneracy requires coordination and is not captured by per‑share checks; protocol brittleness motivates commit‑reveal.

<a id="a06-m203"></a>
### A06.M203 — Appendix notes on degenerate values (Mathematician)

Role: M2

Context: Compact extraction of appendix guidance on Issue 6 (degenerate checks) and connectors to Issue 11 (collusion).

Spec refs (indicative): Appendix Issue 6 lines ~106–112; Issue 11 lines ~174–180.

Summary
- Enumerates degenerate checks (identity/subgroup, T ≠ O, ρ constraints, serialization, size bounds).
- Notes on collusive randomness cancellation and commit‑reveal proposal.

Key Arguments/Findings
- Appendix entries centralize actionable guidance; full appendix remains for extended proofs.

<a id="a06-m204"></a>
### A06.M204 — PoCE‑Across‑Arms soundness (Mathematician)

Role: M2

Context: Multi‑base Schnorr proof showing that all arms share the same exponent ρ.

Relation
- Public inputs: bases {U_ℓ} ∪ {δ₂} and their exponentiations {U_ℓ^ρ} ∪ {δ₂^ρ}.
- PoCE transcript: commitments {T_ℓ = k·U_ℓ}, challenge c = H({U_ℓ}, {U_ℓ^ρ}, {T_ℓ}), responses z = k + cρ.

Claims
- Completeness: Honest prover with ρ passes verification: z·U_ℓ ?= T_ℓ + c·U_ℓ^ρ for all ℓ.
- Soundness (proof of knowledge of common ρ): If an adversary produces (T_ℓ, z) accepting for all ℓ with non‑negligible probability, then (under standard Schnorr assumptions in the random oracle model) we can extract ρ by rewinding on c and solving from two valid transcripts.
- Unforgeability across inconsistent exponents: If any arm used ρ′ ≠ ρ, the corresponding equation fails with overwhelming probability.

Implication
- Ensures deposit‑time arms are exponent‑consistent, preventing mixed‑ρ attacks; complements degenerate‑value guards.

<a id="issue-07"></a>
## Issue 07

Add entries as A07.RNN (e.g., A07.M101, A07.CR02). Keep each entry atomic and concise.

<a id="a07-m201"></a>
### A07.M201 — Timing leakage quantification and mitigations (Mathematician)

Role: M2

Context: Quantifies timing side-channel leakage and proposes constant-time execution and protocol state machine with timeouts.

Spec refs (indicative): [`PVUGC-2025-10-05.md §7 GS PPE Verification (96 pairings)`](../PVUGC-2025-10-05.md#7-gs-ppe-verification-96-pairings) (§6 bounds; decapsulation loop; §12 proposed mitigations) ; Appendix §7.

Summary
- Mutual information framework yields I(W; T_decap) ≈ log₂(#pairing-configs) with 4560 valid (m₁, m₂) pairs under m₁ + m₂ ≤ 96.
- Constant-time requirement: perform fixed 96 pairings (pad with identities) and constant-time DEM decryption (no early abort on tag mismatch).
- Liveness: state machine with mandatory timeouts eliminates deadlocks.

Key Arguments/Findings
- Timing leakage violates zero-knowledge if pairing counts correlate with witness structure.
- Fixed-work decapsulation and explicit timeouts provide cryptographic and operational guarantees.

<a id="a07-cr02"></a>
### A07.CR02 — Race conditions, front-running, invariants (Crypto Reviewer)

Role: CR

Context: Attack algorithms for race conditions and front-running; formal invariants; normative text for §12.

Spec refs (indicative): [`PVUGC-2025-10-20.md §1 Introduction`](../PVUGC-2025-10-20.md#1-introduction) (script, ctx binding, decap loop; proposed §12.1–§12.3).

Summary
- Formal deadlock scenarios and tie-breaking; front-running mitigation options (encrypted mempool, direct miner submission).
- Drafts spec-ready text for constant-time, state machine, and front-running guidance.

Key Arguments/Findings
- Without timeouts and deterministic resolution, trivial DoS persists.
- Front-running analysis ties to transaction template binding and broadcast strategy.

<a id="a07-m202"></a>
### A07.M202 — Revisions and corrections (Mathematician)

Role: M2

Context: Round 3 corrections and additions: worst-vs-average, timing precision, TOST, cache-timing.

Spec refs (indicative): [`PVUGC-2025-10-20.md §1 Introduction`](../PVUGC-2025-10-20.md#1-introduction); Appendix §7.

Summary
- Corrects arithmetic (4560 pairs); clarifies timing precision assumptions (δ_time << σ_pairing/√m).
- Replaces Welch’s t-test with TOST; adds cache-timing considerations.

Key Arguments/Findings
- Strengthens statistical validity and closes remaining analysis gaps.

<a id="a07-m203"></a>
### A07.M203 — Appendix §7 timing notes (Mathematician)

Role: M2

Context: Compact extraction of appendix timing formalization relevant to PVUGC-007.

Spec refs (indicative): [`appendix_mathematical_considerations.md §7`](../report-update-2025-10-07/appendix_mathematical_considerations.md#7).

Summary
- Defines timing leakage model and connects to decapsulation loop structure.
- Supports the need for constant-time execution and measurement methodology.

Key Arguments/Findings
- Consolidates key appendix statements for quick reference by implementers and auditors.

<a id="issue-09"></a>
## Issue 09

Add entries as A09.RNN (e.g., A09.M101, A09.CR02). Keep each entry atomic and concise.

<a id="a09-m201"></a>
### A09.M201 — DEM profile validation (Mathematician)

Role: M2

Context: Formal validation that the single mandatory DEM profile ("PVUGC/DEM-P2-v1") achieves IND-CCA security in the Random Oracle Model, is key‑committing, and removes interoperability risks.

Spec refs (indicative): [`PVUGC-2025-10-20.md §5 DEM Construction (Poseidon2 DEM-P2)`](../PVUGC-2025-10-20.md#5-dem-construction-poseidon2-dem-p2) (§8 DEM construction, lines 217–248; domain tags 79–84; production profile 88–94).

Summary
- Encrypt‑then‑MAC with Poseidon2 over (K_i, AD_core, counter) for keystream and (K_i, AD_core, ct_i) for tag yields IND‑CPA + INT‑CTXT, composing to IND‑CCA.
- Mandatory single profile eliminates format bifurcation; consistent serialization prevents cross‑implementation failure.
- Tag construction is key‑committing: no two different keys validate the same (ct_i, τ_i).

Key Arguments/Findings
- IND‑CPA: With single‑use K_i, Poseidon2 in ROM gives pseudorandom keystream; XOR provides perfect secrecy per instance.
- INT‑CTXT: Forging τ without K_i requires inverting/guessing PRF input; negligible success.
- Composition: Bellare‑Namprempre theorem—IND‑CPA + INT‑CTXT ⇒ IND‑CCA (ROM).
- Interoperability: One profile (“PVUGC/DEM‑P2‑v1”) removes ambiguity; strict ser_GT and domain tags ensure compatibility across implementations.

<a id="a09-cr02"></a>
### A09.CR02 — DEM profile analysis (Crypto Reviewer)

Role: CR

Context: Key‑committing DEM profile and serialization interplay relevant to PVUGC-006’s canonical ser_GT requirement.

Spec refs (indicative): [`PVUGC-2025-10-20.md §5 DEM Construction (Poseidon2 DEM-P2)`](../PVUGC-2025-10-20.md#5-dem-construction-poseidon2-dem-p2) (DEM profile; KDF serialization inputs).

Summary
- Key‑committing DEM with Poseidon2 over (K, AD_core, ct) prevents malleability.
- Consistent ser_GT(M) across implementations is critical; mismatches produce decryption failures.

Key Arguments/Findings
- DEM correctness is enforced in hardened PoCE-A; implementations must align serialization to avoid cross‑compatibility issues.

<a id="a09-m203"></a>
### A09.M203 — Revisions: parameters and test vectors (Mathematician)

Role: M2

Context: Specification precision and testing requirements to make the DEM profile deployable (counter‑mode keystream, explicit tags, Poseidon2 parameters, test vector schema).

Summary / Recommendations
- Use counter‑mode to derive 64‑byte keystream blocks: concatenate two Poseidon2 outputs under an explicit DEM domain tag with counters 0/1.
- Always prepend explicit domain tags for KDF/DEM/TAG calls; specify ASCII encoding.
- Fix Poseidon2 parameters for BLS12‑381 (t=3, R_f=8, R_p=56); document security rationale and MDS/constants provenance.
- Publish normative test vectors (KDF, encryption, decryption, failure cases) with strict encoding rules.

Acceptance
- Counter‑mode keystream specified and tests included.
- Domain tags normative and encoding fixed.
- Poseidon2 parameters normative with references and constants.
- Test vectors published with bit‑exact outputs and failure cases.

<a id="issue-10"></a>
## Issue 10

Add entries as A10.RNN (e.g., A10.M101, A10.CR02). Keep each entry atomic and concise.

<a id="a10-m101"></a>
### A10.M101 — Binding CRS verification essentials (Mathematician)

Role: M1

Context: Canonical checks for binding CRS required by context binding and attestation soundness.

Spec refs (indicative): [`PVUGC-2025-10-05.md §10 Binding CRS Requirements (SXDH/DLIN)`](../PVUGC-2025-10-05.md#10-binding-crs-requirements-sxdh/dlin) (§6 requirement for binding CRS; PVUGC-010 update report for validation procedure).

Summary
- Binding CRS is prerequisite for GS commitment soundness; validators must check subgroup membership, encoding, and parameter consistency.
- Publishing CRS digests and verification transcripts enables third-party audit.

Key Arguments/Findings
- Without binding CRS, attestation malleability and substitution risks reappear.
- Validation should be deterministic and reproducible across implementations.

<a id="a10-cr01"></a>
### A10.CR01 — VK subgroup validation gap (Crypto Reviewer)

Role: CR

Context: `validate_pvugc_vk_subgroups()` is currently a placeholder; production must enforce prime‑order checks.

Summary
- Risk: small‑order components in β₂, δ₂, or G2 query points could undermine GS soundness and binding.
- Recommendation: Verify prime‑order subgroup membership for all VK G2 elements (type‑level guarantees or explicit checks) and fail hard on violation.

Acceptance
- Deterministic subgroup validation during VK import/verification; tests include malformed inputs.

<a id="issue-11"></a>
## Issue 11

Add entries as A11.RNN (e.g., A11.M101, A11.CR02). Keep each entry atomic and concise.

<a id="a11-m201"></a>
### A11.M201 — Collusive randomness cancellation formalization (Mathematician)

Role: M2

Context: Discovery and formalization of coalition strategies choosing {ρ_i} with multiplicative constraints (e.g., ∏ρ_i ≡ 1 mod r) that pass per‑share checks yet coordinate aggregate behavior.

Summary
- Per‑share validation ρ_i ≠ 0 is insufficient; coalitions can enforce ∏ρ_i ≡ c (mod r) while each ρ_i appears uniformly random.
- Ceremony ordering prevents adapting to honest values; grinding is limited to coalition‑local targets (e.g., T_coalition_x).
- Vulnerability does not reveal α directly but enables fairness violations and griefing via repeated abort/restart.

Key Arguments/Findings
- Construction: two‑party ρ₁ = x, ρ₂ = x⁻¹; k‑party generalization with ρ_k = (∏_{i<k} ρ_i)⁻¹.
- Detection gap: No aggregate randomness test; statistical indistinguishability at per‑share level.
- Scope: Non‑grindable targets include final T, presig_pkg_hash, and final signature s due to ordering constraints.

<a id="a11-cr02"></a>
### A11.CR02 — Collusive randomness cancellation catalog (Crypto Reviewer)

Role: CR

Context: Comprehensive catalog of collusive randomness strategies, security games, and the commit‑reveal mitigation.

Spec refs (indicative): [`PVUGC-2025-10-20.md §1 Introduction`](../PVUGC-2025-10-20.md#1-introduction) (§6 guard interactions); Appendix A06.M201 discovery notes.

Summary
- Collusion strategies target aggregate ρ products (e.g., ρ_1 = x, ρ_2 = x^{-1}).
- Commit‑reveal for ρ_i enforces independence and prevents targeted cancellation.

Key Arguments/Findings
- Second‑order vulnerabilities demand protocol‑level randomness coordination, not only per‑share checks.

<a id="a11-m203"></a>
### A11.M203 — Commit‑reveal mitigation and acceptance (Mathematician)

Role: M2

Context: Protocol‑level mitigation to enforce independence of ρ_i via a lightweight commit‑reveal before arming; testable invariants and acceptance.

Summary / Recommendations
- Add KEM Randomness Commitment: each armer commits C_i = H_bytes("PVUGC/RHO-COMMIT/v1" || pk_i || salt_i || ser(ρ_i)).
- Reveal phase before arming: publish (ρ_i, salt_i); verifier checks binding to C_i.
- Invariants: uniqueness of (pk_i, salt_i); fixed window between commit and reveal to reduce adaptive behavior.
- Add validation that all commits present before any reveal; abort on mismatch.

Acceptance
- Commit‑reveal integrated with explicit domain tag and encoding; acceptance tests include valid/invalid reveals and replay.
- Updated timing/state machine specified; no signing/arming until all valid reveals.

---

<a id="issue-08"></a>
## Issue 08

Add entries as A08.RNN (e.g., A08.M101, A08.CR02). Keep each entry atomic and concise.

<a id="a08-m201"></a>
### A08.M201 — Nonce reuse refutation and extraction (Mathematician)

Role: M2

Context: Refutation of “MUST” sufficiency for MuSig2 nonce uniqueness; formal O(1) private‑key extraction when the same R is reused across two signatures.

Summary
- Two Schnorr signatures with identical aggregate R over different messages leak the aggregate private key x: x = (s₁ − s₂) · (c₁ − c₂)⁻¹ (mod n).
- Behavioral “never reuse” clauses are insufficient; cryptographic enforcement is required.
- Severity: High — complete key compromise; bypasses gating and WE entirely.

Key Statements
- Equations: sᵢ = r + cᵢ·x (mod n) with cᵢ = H(BIP0340/challenge, Rₓ || Pₓ || mᵢ). If R repeats, subtract to eliminate r.
- Deterministic attack: modular subtraction + inversion; success probability ~1 when m₁ ≠ m₂.
- Implication: Protocol must specify a deterministic nonce derivation mechanism bound to context.

<a id="a08-cr02"></a>
### A08.CR02 — Deterministic HKDF nonce derivation (Crypto Reviewer)

Role: CR

Context: Sketch of a normative, deterministic nonce derivation scheme for MuSig2 to enforce uniqueness and context binding.

Specification sketch
- Derivation uses HKDF (RFC 5869), versioned domain tags, and produces two nonces (BIP‑327 style):
  - prk = HKDF-Extract(salt = H_bytes(ctx_hash), ikm = skᵢ)
  - rᵢ,1 = bytes_to_scalar(HKDF-Expand(prk, info = "PVUGC/MuSig2-Nonce/v1" || 0x00, 32))
  - rᵢ,2 = bytes_to_scalar(HKDF-Expand(prk, info = "PVUGC/MuSig2-Nonce/v1" || 0x01, 32))
  - Reject 0 or ≥ n; otherwise use (rᵢ,1, rᵢ,2)
- Inputs exclude R to avoid circularity; perform xonly R normalization before computing the BIP‑340 challenge.
- Context binding via ctx_hash; signer_set and musig_coeffs are captured upstream; no direct inclusion of R here.

Properties / Acceptance
- Uniqueness per context; resistance to reuse; reproducibility for testing.
- BIP‑327 compliance; deterministic test vectors; 1000+ uniqueness tests; tie‑breaking for simultaneous timeout.

Notes
- Nonce commitment/proof is optional; deterministic derivation suffices to prevent accidental reuse.

<a id="a08-m203"></a>
### A08.M203 — Revisions: worst‑vs‑average, TOST, cache timing (Mathematician)

Role: M2

Context: Corrections and methodology improvements for MuSig2 timing/nonce analysis and tests.

Summary / Recommendations
- Distinguish worst‑case from average‑case analyses; specify timing precision assumptions.
- Replace Welch’s t‑test with TOST for equivalence testing in timing invariance.
- Add cache‑timing considerations and invariants to the state machine; specify R normalization timing.

Acceptance
- Updated tests (TOST‑based), documented invariants, and clarified timing assumptions.
