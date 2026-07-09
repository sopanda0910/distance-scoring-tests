# hybrid_cons_prob v2 vs v4 — side-by-side scores

Both share the same median location, robust MAD-family percentile widths (16/84),
and plateau z-score branch. They differ **only** in the non-plateau branch:

- **v2** — smooth blend of both tails: `Φ(z)·erfc(|d|/σ₋) + (1−Φ(z))·erfc(|d|/σ₊)`
- **v4** — discrete pick of the single tail facing the GW (the original
  `consistency_probability` rule)

GW reference: symmetric Gaussian, mean 0, σ = 5. Scores at `wt = 1`.

| case | loc | unc₋ / unc₊ | median | v2 (blend) | v4 (one tail) | Δ (v2−v4) |
|---|---|---|---|---|---|---|
| base_pdf | 0 | 5 / 5 | -0.1 | 1.0000 | 1.0000 | 0.0000 |
| deltafunc_pdf | 0 | 1 / 1 | -0.1 | 1.0000 | 1.0000 | 0.0000 |
| deltafunc_slightly_offset_pdf | 5 | 1 / 1 | 4.9 | 0.9091 | 0.9091 | 0.0000 |
| deltafunc_offset_pdf | 50 | 1 / 1 | 49.9 | 0.0000 | 0.0000 | 0.0000 |
| wide_pdf | 0 | 10 / 10 | -0.1 | 1.0000 | 1.0000 | 0.0000 |
| wide_slightly_offset_pdf | 5 | 10 / 10 | 4.9 | 0.6547 | 0.6547 | 0.0000 |
| wide_pdf_offset | 15 | 10 / 10 | 14.9 | 0.1797 | 0.1797 | 0.0000 |
| wide_pdf_veryoffset | 50 | 10 / 10 | 49.9 | 0.0000 | 0.0000 | 0.0000 |
| verywide_pdf | 0 | 50 / 50 | -0.1 | 1.0000 | 1.0000 | 0.0000 |
| verywide_slightly_offset_pdf | 5 | 50 / 50 | 4.2 | 0.9272 | 0.9275 | -0.0002 |
| verywide_pdf_offset | 50 | 50 / 50 | 40.0 | 0.3792 | 0.3792 | 0.0000 |
| asymmetric_pdf_deltafunc | 0 | 0.5 / 1 | 0.2 | 0.9942 | 0.9942 | 0.0000 |
| asymmetric_pdf_deltafunc_offset | 50 | 0.5 / 1 | 50.2 | 0.0000 | 0.0000 | 0.0000 |
| **asymmetric_pdf_wide** | 0 | 7.5 / 12.5 | 3.1 | **0.7705** | **0.7618** | **+0.0087** |
| asymmetric_pdf_wide_offset | 50 | 7.5 / 12.5 | 53.1 | 0.0000 | 0.0000 | 0.0000 |
| asymmetric_pdf_wide_offset_neg | -50 | 7.5 / 12.5 | -46.9 | 0.0001 | 0.0001 | 0.0000 |
| asymmetric_pdf_very_skewed | 0 | 50 / 1 | -31.3 | 0.1799 | 0.1799 | 0.0000 |
| asymmetric_pdf_very_skewed_offset | 20 | 50 / 1 | -12.4 | 0.6098 | 0.6090 | +0.0008 |
| asymmetric_pdf_very_skewed_very_offset_pos | 80 | 50 / 1 | 47.0 | 0.2071 | 0.2071 | 0.0000 |
| asymmetric_pdf_very_skewed_very_offset_neg | -80 | 50 / 1 | -89.3 | 0.0000 | 0.0000 | 0.0000 |
| asymmetric_pdf_very_skewed_slightly_offset_neg | -5 | 50 / 1 | -35.8 | 0.1192 | 0.1192 | 0.0000 |
| asymmetric_pdf_very_skewed_moderately_offset_neg | -8 | 50 / 1 | -38.5 | 0.0907 | 0.0907 | 0.0000 |
| **asymmetric_v4_straddle** | -6 | 1 / 15 | 3.2 | **0.7280** | **0.7084** | **+0.0196** |

## Summary

- **21 of 23 cases are identical** to ≤0.001. Only two show a meaningful gap:
  `asymmetric_v4_straddle` (+0.0196, the constructed worst case) and
  `asymmetric_pdf_wide` (+0.0087).
- **Maximum disagreement is ~0.02** (2 percentage points). v4's one-tail
  limitation is real but small on split-normal PDFs.
- **Why the gap stays small:** v2 and v4 only differ in the transition zone —
  median within ~1–2 robust-σ of the GW *and* asymmetric widths. Outside it,
  `Φ(z)` saturates onto the single facing tail, so v2 collapses to v4. And inside
  it, the median is close enough that both erfc scores are already high, capping
  how much the ignored far tail can change the result.
- **When v2 scores higher (positive Δ):** the tail v4 discards is the *wider*
  one, so v2's blend adds a bit of leniency (`asymmetric_v4_straddle`,
  `asymmetric_pdf_wide`). When the discarded tail is narrower, v4 edges slightly
  above v2 (`verywide_slightly_offset_pdf`, −0.0002).

Plots: `hybrid_tests/v2/`, `hybrid_tests/v4/`, and the side-by-side
`hybrid_tests/v2_vs_v4_straddle.png`.
