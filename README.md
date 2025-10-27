# PVUGC Protocol Security Review

**Author:** Fernando Paredes <fernando@develcuy.com>

**License:** [CC-BY 4.0](https://creativecommons.org/licenses/by/4.0/)

## Quick Start
- Read [`PVUGC-2025-10-20.md §1 Introduction`](PVUGC-2025-10-20.md) for the protocol specification that the reviews target.
- Use [`report-peer_review-2025-10-26/README.md`](report-peer_review-2025-10-26/README.md) for the current executive summary; it supersedes the v2.0 update and preliminary reports where noted.
- Refer to [`report-peer_review-2025-10-26/00-INDEX.md`](report-peer_review-2025-10-26/00-INDEX.md) for live status, severities, and deep links into each peer‑reviewed issue brief.

## Document Map
- [`report-peer_review-2025-10-26/`](report-peer_review-2025-10-26) – Version 3.0 peer review package (current). Each PVUGC-00X.md file contains the peer‑reviewed issue report; see [00-INDEX.md](report-peer_review-2025-10-26/00-INDEX.md) for status.
- [`report-update-2025-10-07/`](report-update-2025-10-07) – Version 2.0 review package (superseded by v3.0). Each PVUGC-00X-*.md file revisits a specific issue, tracking whether it remains open, improved, or resolved.
- [`report-preliminary-2025-10-07/`](report-preliminary-2025-10-07) – Original assessment set. Compare with the update to trace how each flaw evolved from initial discovery.
- [`report-peer_review-2025-10-26/appendix_mathematical_considerations_v2.md`](report-peer_review-2025-10-26/appendix_mathematical_considerations_v2.md) – Peer-reviewed companion with Mathematician #2's validations and enhancements.
- [`report-update-2025-10-07/appendix_mathematical_considerations.md`](report-update-2025-10-07/appendix_mathematical_considerations.md) – v2.0 appendix (superseded; still useful for historical derivations).
- [`preliminary-peer_review-plan.md`](report-peer_review-2025-10-26/preliminary-peer_review-plan.md) – Proposed external review cadence, roles, and deliverables.
- [`report-peer_review-2025-10-26/preliminary-peer_review-report.md`](report-peer_review-2025-10-26/preliminary-peer_review-report.md) – Complete Round 2 synthesis covering validations, refutations, enhancements, and novel findings.
- [`PVUGC-2025-10-05.md`](PVUGC-2025-10-05.md) – Production specification with GT-XPDH assumption, Multi-CRS AND-ing, and normative security requirements (two-sided GS + HKDF/AEAD).

## How to Navigate Findings
- Start with the peer review directory to see the latest status, then consult the matching file in the v2.0 update and preliminary reports to understand the original reasoning.
- Issue codes stay stable across versions (PVUGC-001 through PVUGC-011). Status icons in [`00-INDEX.md`](report-peer_review-2025-10-26/00-INDEX.md) flag whether follow-up is still required.
- When diving deeper on critical items, pair the issue brief with the relevant section in [`appendix_mathematical_considerations_v2.md`](report-peer_review-2025-10-26/appendix_mathematical_considerations_v2.md) for technical obligations and suggested proof strategies.

## Suggested Reading Order
1. [`PVUGC-2025-10-20.md §1 Introduction`](PVUGC-2025-10-20.md) (specification baseline)
2. [`report-peer_review-2025-10-26/README.md`](report-peer_review-2025-10-26/README.md) (current executive summary)
3. Issue briefs in [`report-peer_review-2025-10-26/`](report-peer_review-2025-10-26), prioritising open critical/high items (PVUGC-001–011)
4. Matching files in [`report-update-2025-10-07/`](report-update-2025-10-07) and [`report-preliminary-2025-10-07/`](report-preliminary-2025-10-07) for historical context
5. Supporting materials: [`appendix_mathematical_considerations_v2.md`](report-peer_review-2025-10-26/appendix_mathematical_considerations_v2.md)

---

## References

This review project analyzes and builds upon the following foundational documents by **sidhujag**, sourced from [this HackMD document](https://hackmd.io/@sidhujag/BJeoZ0Ohxe):

- **PVUGC (Mainnet-Today, Prove-to-Sign)** ([`PVUGC-2025-10-05.md`](PVUGC-2025-10-05.md))
- **Proof-Agnostic Verifiable Unique Group Commitments (PVUGC)** ([`PVUGC-2025-10-20.md`](PVUGC-2025-10-20.md))

---

*No AIs were harmed during the production of this work.*
