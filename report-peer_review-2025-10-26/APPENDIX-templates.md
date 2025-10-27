# Appendix Templates

This file contains reusable templates for the peer review appendix system.

---

## Entry Template

Use this template when adding new entries to `APPENDIX-issue-debates.md`.

```markdown
<a id="aII-rnn"></a>
### AII.RNN — Short Descriptive Title

Role: M1 | M2 | CR | LR

Context (optional): one line to orient the reader; no links to original debate files.

Spec refs (optional): [`<spec-file>.md §N Section Title`](../<spec-file>.md#section-anchor), other spec/report anchors as needed.

Summary
- Key point 1
- Key point 2
- Key point 3 (as needed)

Key Arguments/Findings
- Evidence-backed statement with brief rationale.
- Optional short quote (<= 1–2 lines) if wording is critical.
- Keep to essentials; prefer bullets.
```

**Instructions:**
- Replace `II` with two-digit issue number (01..99)
- Replace `R` with role code: M1/M2/... (Mathematician), CR (Crypto Reviewer), LR (Lead Reviewer)
- Replace `NN` with two-digit sequential index per issue (01..99)
- Replace `<spec-file>` and `#section-anchor` with actual spec file and markdown anchor
- Keep entries atomic (200–400 words); no links back to original `issue-XX-...md` files
- Ensure anchor `id="aII-rnn"` is lowercase and hyphenated (e.g., `id="a04-m101"`)

---

## Redirect Stub Template

Use this template when replacing original `issue-XX-debate-...md` files with redirect stubs.

```markdown
# Moved

This content has moved to the Appendix:

- Appendix: `APPENDIX-issue-debates.md#aII-rnn`
- Context: Issue II — <Short Title>, Code AII.RNN
```

**Instructions:**
- Replace `II` with two-digit issue number
- Replace `rnn` with role code and sequential number (lowercase, hyphenated)
- Replace `RNN` with role code and sequential number (uppercase, e.g., M101, CR02)
- Replace `<Short Title>` with brief description of the issue
- Keep the original filename unchanged to preserve external links
- Optionally include a one-line summary below the context line

---

## Notes

- **Atomicity:** Appendix entries must be self-contained and not rely on original debate files.
- **Compactness:** Target 200–400 words per entry; use bullets over prose.
- **Spec refs:** Always use relative paths from git root (e.g., `../PVUGC-2025-10-20.md`).
- **Anchors:** Use lowercase, hyphenated format for anchor IDs (e.g., `#a04-m101`).
