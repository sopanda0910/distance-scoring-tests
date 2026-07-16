Some intuitive limitations that I can think of from how the current distance scoring works
- Bhattachary Coefficient is not a good distance metric (in distribution space), and therefore consider a more common sqrt(1 - BC**2), or something similar to that to actually compute something like the dot product rather than the distance in distribution space
- The weights consider the average of the tails, and it is an even average, so high asymmetric distributions are treated the same as symmetric distributions. This is fine for the most part, but some weighting considerations could be implemented depending on which side of the mode it is on, similar to the consistentency_probability method almost
- The tophat score essentially checks whether it is within 2 sigma in a smooth box using sigmoid functions (why not consider something similar to the z-score or something like that?)
- Additionally, considering a box, this loses ranking information, so a candidate that is 0.5sigma and 1.5 sigma will get the same score, which is undesirable?


- One of the key differences between linear weighting and using something like logistic is near the weight of 0 (for things like delta functions). A linear relationship would have a weight somewhere near 0.2/0.1, but using a logistic, the weight drops down to 0.03, which is actually desirable.


## Justification for the logistic weighting parameters (r0 = 1, k = 4)

The hybrid score blends the top-hat and BC branches with `w(r) = 1 / (1 + exp(-k(r - r0)))`,
where `r = sigma_gal / sigma_gw` is the width ratio. The parameter choices are not arbitrary:

### Setting r0 = 1

`r0` is the ratio at which we trust both branches equally (w = 0.5). The natural choice is
`r0 = 1`: the galaxy PDF and the GW posterior are equally informative when they have equal
width. Below it, the galaxy PDF is sharp enough that "where is the point estimate?" (top-hat)
is the better question; above it, the PDF is broad enough that "how much do the distributions
overlap?" (BC) is the better question.

### Setting k = 4 — three convergent arguments

**1. The BC's own systematic error defines the handoff window (primary argument).**
For a galaxy *perfectly centered* on the GW mean, the Bhattacharyya coefficient is not 1 but

    BC_centered(r) = sqrt(2r / (1 + r^2))

i.e. it penalizes pure width mismatch even when the distance is exactly right. A correct
spec-z galaxy at r = 0.2 gets BC = 0.62 — a 38% penalty for being *right*. (This is the
pathology the hybrid exists to fix, and matches the notebook: deltafunc_pdf has BC = 0.6202.)

Solving BC_centered(r) >= 0.9 gives the window where BC misjudges a perfect match by
less than 10%:  **r in [0.51, 1.96]**.  The weight should therefore hand off across
exactly this window:

- require w(0.5) <= 0.1  (trust BC <= 10% where it is > 10% wrong)  ->  k >= 2 ln 9  ~ 4.39
- require w(2.0) >= 0.98 (fully trust BC once it is reliable)       ->  k >= ln 49   ~ 3.89

k = 4 is the round number satisfying both. With k = 4: w(0.5) = 0.12 and w(2) = 0.98,
so the logistic's 12–88% transition spans r in [0.5, 1.5], mirroring the BC trust window.

**2. Continuity with the original linear scheme.**
The logistic's maximum slope is k/4 at r0. For k = 4 that slope is exactly 1 — identical
to the original `weight_linear` ramp (w = r) it replaced. So k = 4 is the unique steepness
for which the logistic is the smooth version of the legacy clipped-linear weighting rather
than a differently calibrated scheme; intermediate cases score consistently with v1/v2.

**3. Spec-z anchor.**
w(0) = 1 / (1 + e^(k*r0)) = 1 / (1 + e^4) ~ 0.018, so a spectroscopic delta-function keeps
>= 98% top-hat weight. For comparison, k = 3 would leak 4.7% to the (broken-at-r=0) BC
branch, and k = 4.6 would be needed for 99%. The 2% leakage at k = 4 is consistent with
the 2% tail floor used elsewhere in the top-hat scorer.

### Summary

| r    | w(r), k=4 | regime                                   |
|------|-----------|------------------------------------------|
| 0    | 0.018     | spec-z: essentially pure top-hat          |
| 0.5  | 0.119     | BC starts becoming trustworthy            |
| 1.0  | 0.500     | equal information -> equal trust (r0)     |
| 1.5  | 0.881     | BC dominates                              |
| 2.0  | 0.982     | photo-z: essentially pure BC              | 