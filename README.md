# PVUGC Bitcoin Bridge Security Report

## Quick Start
- Read [`PVUGC-2025-10-07.md`](PVUGC-2025-10-07.md) for the protocol specification that the reviews target.
- Use [`report-update-2025-10-07/README.md`](report-update-2025-10-07/README.md) for the current executive summary; it supersedes the preliminary report where noted.
- Refer to [`report-update-2025-10-07/00-INDEX.md`](report-update-2025-10-07/00-INDEX.md) for live status, severities, and deep links into each updated issue brief.

## Document Map
- [`report-update-2025-10-07/`](report-update-2025-10-07) – Version 2.0 review package. Each [`PVUGC-00X-*.md`](report-update-2025-10-07) file revisits a specific issue, tracking whether it remains open, improved, or resolved.
- [`report-preliminary-2025-10-07/`](report-preliminary-2025-10-07) – Original assessment set. Compare with the update to trace how each flaw evolved from initial discovery.
- [`report-update-2025-10-07/appendix_mathematical_considerations.md`](report-update-2025-10-07/appendix_mathematical_considerations.md) – Extended algebraic notes keyed to the issue codes (ideal for formal proof or counterexample work).
- [`report-peer_review-2025-10-15/appendix_mathematical_considerations_v2.md`](report-peer_review-2025-10-15/appendix_mathematical_considerations_v2.md) – Peer-reviewed companion with Mathematician #2's validations and enhancements.
- [`PVUGC-breakthrough.md`](PVUGC-breakthrough.md) – Narrative overview of why the adaptor-signature approach matters and the breakthroughs claimed.
- [`preliminary-peer_review-plan.md`](report-peer_review-2025-10-15/preliminary-peer_review-plan.md) – Proposed external review cadence, roles, and deliverables.
- [`CLAUDE.md`](CLAUDE.md) – Transcript of an auxiliary model-assisted analysis session; useful for alternative perspectives.
- [`Sys_BTC_Bridge.pdf`](Sys_BTC_Bridge.pdf) – Slide-friendly summary of the bridge concept and threat model (binary PDF).

## How to Navigate Findings
- Start with the update directory to see the latest status, then consult the matching file in the preliminary report to understand the original reasoning.
- Issue codes stay stable across versions (PVUGC-001 through PVUGC-010). Status icons in [`00-INDEX.md`](report-update-2025-10-07/00-INDEX.md) flag whether follow-up is still required.
- When diving deeper on critical items, pair the issue brief with the relevant section in [`appendix_mathematical_considerations.md`](report-update-2025-10-07/appendix_mathematical_considerations.md) for technical obligations and suggested proof strategies.

## Suggested Reading Order
1. [`PVUGC-2025-10-07.md`](PVUGC-2025-10-07.md) (specification baseline)
2. [`report-update-2025-10-07/README.md`](report-update-2025-10-07/README.md) (current executive summary)
3. Issue briefs in [`report-update-2025-10-07/`](report-update-2025-10-07), prioritising open critical/high items
4. Matching files in [`report-preliminary-2025-10-07/`](report-preliminary-2025-10-07) where more narrative context is needed
5. Supporting materials: [`appendix_mathematical_considerations.md`](report-update-2025-10-07/appendix_mathematical_considerations.md)
