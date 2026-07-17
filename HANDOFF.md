# Distance Scoring — Project Handoff

> Context document for a fresh Claude session. Everything needed to pick up work on the
> gravitational-wave (GW) host-galaxy **distance scoring** project without prior conversation.
> Written 2026-07-17.

---

## 0. TL;DR

We score how well a candidate host galaxy's **distance estimate** agrees with a GW event's
**distance posterior**, to help vet optical counterpart candidates (kilonovae, sub-solar-mass
events) for the TROVE / SAGUARO follow-up pipeline. The current best metric is
**`hybrid_v3`** (a.k.a. "Hybrid BC/Tophat V3"): a heavy-tailed top-hat branch blended with an
analytic Bhattacharyya-coefficient (BC) branch, weighted by a **logistic function of the
host/GW width ratio** (`r0=1, k=4`), built on **robust median/MAD statistics**. It is
validated on 23 synthetic PDF stress cases and 2112 real hosts from event **S251112cm**.

The immediate deliverable in progress is a **research poster** — see `POSTER.md`.

---

## 1. Scientific motivation

- GW events (BNS mergers, and sub-solar-mass / SSM candidates) are localized in 3D:
  sky area × luminosity **distance**. The distance posterior is roughly Gaussian and
  well-characterized.
- Optical surveys yield hundreds–thousands of transient candidates per event. Each candidate's
  **host galaxy distance** is one of the strongest discriminants — *if scored correctly*.
- **Key tension (the thesis of the whole project):** host distances come in two very
  different flavors that a single naive metric cannot serve:
  - **spec-z** (spectroscopic redshift): near delta-functions, very sharp.
  - **photo-z** (photometric redshift): broad, often asymmetric, sometimes nearly uninformative.
- Right question by regime:
  - Sharp host PDF → *"is the point estimate consistent with the GW posterior?"* (top-hat / consistency)
  - Broad host PDF → *"how much do the two distributions overlap?"* (BC)
- The distance score is **one term** in a larger candidate score (sky position, photometric
  evolution, etc.) from Noah's distance-scoring work.

## 2. Design requirements for a "good" distance score

1. Bounded in [0, 1]; identical PDFs → 1; disjoint PDFs → 0.
2. A spec-z galaxy at exactly the right distance must score **~1**, even though its PDF shape
   looks nothing like the GW posterior. **This is precisely where plain BC fails.**
3. A very broad / uninformative photo-z should score **moderately** — neither rewarded nor excluded.
4. Score should **decline monotonically with offset** and **preserve ranking** (a 0.5σ candidate
   must beat a 1.5σ candidate — a hard box loses this).
5. Robust to **asymmetric/skewed** photo-z PDFs and outliers → median/MAD, not mean/std.
6. Fast & vectorizable — must run on thousands of candidates during live follow-up.

## 3. The central failure mode (the motivating result)

The **Bhattacharyya coefficient**, `BC = ∫ √(p_GW · p_host) dD`, penalizes pure *width*
mismatch even when the distance is exactly right. For a host **perfectly centered** on the GW
mean with width ratio `r = σ_cand/σ_gw`:

    BC_centered(r) = √(2r / (1 + r²))

So a correct sharp spec-z galaxy at `r = 0.2` scores **0.62** — a ~38% penalty *for being
correct*. Requirement #2 is violated. Everything downstream exists to fix this while keeping
BC's good behavior in the broad-PDF regime.

`BC_centered(r) ≥ 0.9` only holds for `r ∈ [0.51, 1.96]` — this "BC trust window" is what sets
the logistic weighting parameters below.

## 4. Iteration history (what was tried, in order)

| # | Method | Idea | Verdict |
|---|--------|------|---------|
| 0 | **Plain BC** | overlap integral | Fails req #2 (0.62 on centered spec-z) |
| 1 | **Information-theory** (JSD to GW × JSD-to-uniform, + Wasserstein) | separate shape-agreement from information content; weights 0.4/0.6 | Right instincts, wrong machinery: ad-hoc weights, costly entropy integrals, real-data failure — anomalous high scores (~0.8–1.0) for photo-z hosts at 2000–10000 Mpc |
| 2 | **Consistency probability** | erfc of scaled offset; "is point estimate consistent?" | Correct 1.0 on centered spec-z, but ignores host shape → underuses informative photo-z |
| 3 | **Hybrid** (top-hat ⊕ BC, weight = width ratio) | ask each question in its regime | The winning structure |
| 4 | **Robust stats** (median/MAD vs mean/std), v1–v4 | tolerate skew/outliers | v2 tightens spec-z scores hugely; introduced a real-data correlation regression (see below) |
| 5 | **Noah's latest hybrid** | production baseline | The "before" for V3 |
| 6 | **`hybrid_v3` (mine)** | new top-hat + logistic weighting | **Current best** |

### v1–v4 naming (from `README.md` / `v2_vs_v4.md`)
- **v1** = improved consistency probability, mean + std
- **v2** = median + MAD (smooth blend of both tails)
- **v3** = median + std
- **v4** = current consistency-probability rule, median + MAD (discrete single facing tail)
- MAD variant uses **Q86/Q14** (not Q75/Q25) so it limits to σ for a Gaussian.
- v2 vs v4 differ **only** in the non-plateau branch; **max disagreement across 23 cases ≈ 0.02**
  (the constructed worst case `asymmetric_v4_straddle`). The simpler v4 rule is safe.

## 5. `hybrid_v3` — the current method (my contribution)

Two changes over Noah's hybrid:

### (a) New heavy-tailed top-hat (`tophat_score`)
The old `smooth_tophat_score` (product of two sigmoids) crashes below 1e-4 past ~4σ, so a 5σ
and a 10σ candidate become indistinguishable → **ranking destroyed** (violates req #4). The new
`tophat_score` is a sum of three terms in `z = (galaxy_dist − gw_mean)/gw_std`, `u = |z|`:

- **tilt** `exp(alpha·u²)`, `alpha = ln(box_edge_score)/nsigma²` — gentle in-box gradient that
  rewards *exact centering* over "anywhere in the box".
- **core** `exp(−(u/cliff)^(2·cliff_steepness))` — flat plateau + steep cliff (super-Gaussian).
- **tail** `1/(1 + (u/tail_scale)²)` — heavy tail so far candidates stay **rankable** (~1e-2–1e-3
  floor) while still clearly disfavored.
- Combined: `0.98·tilt·core + 0.02·tail`. Defaults: `nsigma=2, cliff=2.5, cliff_steepness=4,
  box_edge_score=0.95, tail_scale=6.0`.

### (b) Logistic branch weighting (`weight_logistic`)
`w(r) = 1 / (1 + exp(−k(r − r0)))` with **r0 = 1, k = 4**, `r = σ_cand/σ_gw` via `sigma_ratio`
(naive mean of the two tails / gw_std). Final score = `(1−w)·tophat + w·BC`.

Three convergent arguments for **k = 4** (full derivation in `NOTES.md`):
1. **BC trust window** `[0.51, 1.96]`: k=4 puts the logistic's 12–88% transition exactly across
   it — `w(0.5)=0.12`, `w(2)=0.98`.
2. **Continuity** with the legacy linear ramp: max slope `k/4 = 1` matches the old `w = r`.
3. **Spec-z anchor**: `w(0) = 1/(1+e⁴) ≈ 0.018` → a delta-function keeps ≥98% top-hat weight
   (linear weighting would leak 10–20% into the broken-at-r=0 BC branch).

Weight-scheme numeric evidence: centered spec-z scores **0.924 (linear) → 0.985 (logistic)**.
Also tested tanh and Hill; logistic chosen for the three arguments above.

## 6. Real-data validation (S251112cm, 2112 hosts)

- **Matches expected distribution:** scores rise/fall with the actual GW distance PDF shape,
  not an arbitrary function of distance.
- **Timing** (40 real records, 20 repeats each): active metrics cost tens of µs/call —
  Hybrid BC/Tophat 47.9 µs, **V3 34.4 µs** mean. Negligible vs the ~7–10 s per-candidate
  network/DB wait. The old numerical-integration metrics `bc_slow` (335 µs) and `bc_norm`
  (11.8 ms, 100k-point grid) are **not** in the pipeline; `bc (analytic)` (15.7 µs) is the
  closed-form replacement used internally.
- **V3 vs previous hybrid:** agree (~0) almost everywhere; in the **20–200 Mpc transition zone**
  V3 is more consistent for well-measured hosts near the true distance (**diff up to +0.6**), at
  the cost of slightly lower scores (down to −0.16) for well-measured hosts that are close but
  off-center — a deliberate trade (the tilt term rewards exact centering).
- **Uncertainty–score correlation (open question):** current Spearman ρ(uncertainty, score) by
  distance bin dips toward ~0 near the GW peak (~120 Mpc) and rises to ~0.9+ far from it. This is
  arguably *intended* (uncertainty matters less when a host is obviously right). NOTE the
  **v2 robust-stats regression**: v2's correlation went *negative* at 30–150 Mpc — a "fixed one
  thing, broke another" cautionary example, since resolved.

## 7. Repository layout

**This repo:** `/home/sopanda25/distance_scoring/` (research/prototyping + poster)

| Path | What |
|------|------|
| `POSTER.md` | **Primary active deliverable** — full poster plan with embedded figures, research question, outcomes |
| `HANDOFF.md` | This file |
| `NOTES.md` | Design rationale; full `r0=1, k=4` justification |
| `README.md` | v1–v4 naming, MAD-approximation note, older scoring-result notes |
| `v2_vs_v4.md` | Side-by-side v2 vs v4 score table (max Δ ≈ 0.02) |
| `hybrid_new.ipynb` | **Latest notebook** — hybrid_v3, new top-hat, weight schemes |
| `hybrid.ipynb`, `dev.ipynb` | Earlier prototyping notebooks |
| `hybrid_new/` | `v1–v4.png` (23-case score grids), `weights/{linear,logistic,tanh,hill}.png`, `diagnostics/tophat_and_weighting.png` |
| `hybrid_tests/` | `v1–v4/` per-case plots, `v2_vs_v4_straddle.png` |
| `pres/` | Per-method per-case plots: `bc/`, `bc_norm/`, `consistency_prob/`, `cons_prob_3/`, `hybrid/` |
| `out/` | `jsd_uniform/`, `hellinger_dist/`, `hybrid/`, `normalization/`, `params_sweep/` (nσ 0.5–3.0 + `scores.csv`) |
| `plots/` | The synthetic test PDFs themselves |
| `real_data/` | **S251112cm figures copied from trove** for the poster (7 PNGs) |

Plot convention: **red = GW posterior, gray = host galaxy PDF**. Same ~23 synthetic cases
(delta / wide / very-wide / offset / skewed) reused across every method for honest comparison.

**Production code lives in the trove repo, NOT here:**
`/home/sopanda25/trove/scoring/dist_scoring_helpers.py` — the authoritative implementation of
every metric: `bc`, `consistency_probability`, `hybrid_cons_prob`, `hybrid_cons_prob_v2`,
`hybrid`, `tophat_score`, `smooth_tophat_score`, `sigma_ratio`, `weight_logistic`,
`hybrid_v3`, plus the analytic BC pieces (`normalization_prefactor`, `bc_integral_neg/pos`)
and info-theory helpers (`jsd`, `information_metric`, `conditional_scoring`, `robust_stats`).
Related: `scoring/scoring.py`, `scoring/management/commands/{time_distance_metrics,
check_distance_scores,recompute_cached_metrics}.py`. The real-data figures were generated into
`/home/sopanda25/trove/out/` (source markdown: `/home/sopanda25/trove/POSTER.md`, which is the
report the poster PDF was exported from).

## 8. Environment

- Use conda env **`env`** (`/home/sopanda25/miniconda3/envs/env/bin/python`) — **not** base
  (base lacks numpy). Other envs exist: `ligo_env`, `myenv` (has PyMuPDF), `trove-env`, `t-env`.
- Metrics depend on numpy + scipy (`scipy.stats.norm`, `erfc`).

## 9. Open threads / possible next steps

- **Generate the missing figure:** a clean `w(r)` plot, linear vs logistic, with the BC-trust
  window `[0.51, 1.96]` shaded and anchor points (r = 0, 0.5, 1, 1.5, 2) annotated. This is the
  only poster figure that doesn't yet exist and it carries the k=4/r0=1 story by itself.
- Decide whether the **near-peak dip** in the uncertainty–score correlation is a real problem or
  expected behavior (Section 6).
- Photo-z PDFs are modeled as **asymmetric (split-normal) Gaussians**; real p(z) can be
  **multimodal** — median/MAD helps but doesn't capture multiple peaks. Possible future work.
- Peculiar-velocity / redshift→distance conversion uncertainty for very nearby events.
- Validate on a catalog with **known hosts** (past events with confirmed counterparts, or
  injected signals).
- From `NOTES.md`: consider `sqrt(1 − BC²)`-style true distance metric; asymmetric-tail-aware
  weighting depending on which side of the mode the GW mean sits.

## 10. Key numbers to remember

- Centered spec-z under plain BC: **0.62** (→ ~1.0 with hybrid_v3).
- BC trust window: **r ∈ [0.51, 1.96]**.
- Logistic weights: `w(0)=0.018, w(0.5)=0.12, w(1)=0.5, w(1.5)=0.88, w(2)=0.98`.
- v2 vs v4 max disagreement: **~0.02** across 23 cases.
- Real data: **2112 hosts**, V3 mean **34 µs/call**, transition-zone gain **up to +0.6**.
