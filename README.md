# Distance Scoring Metrics

# Our version of MAD
I am currently implementing a version of mad that limits to std for a symmetric gaussian distribution.
Instead of using Q75 and Q25, I am using Q86 and Q14, to better approximate 1 sigma of a gaussian distribution

# Hybrid Tests
v1 corresponds to improved consistent probability with mean and std
v2 corresponds to improved consistent probability with median and mad
v3 corresponds to improved consistent probability with median and std
v4 corresponds to current consistent probability with median and mad

# Scoring Results
Check out/jsd_uniform folder for images of the test_pdf, base_pdf and their final scores.
The out/entropy_uniform was just some testing with a different way to calculate how similar
test_pdf is to a uniform distribution, but it did not give good results.

# Standard Deviation Problem
Currently, there is a problem where if the optical distance PDF is very broad, then 
we don't know much about the function, so it should maintain a high score (near 1). On the other hand, for a function that is well-known and far from the mean, it should have a lower score. Therefore, the score calculated by relative entropy or whatever method should be multiplied by some standard deviation factor. This factor should be bounded between 0 and 1, and approach 1 for very low and very high standard deviation, and be approximately zero for anything in between (possibly skewed to lower standard deviations)
Consider calculating relative entropy between GW PDF and Optical PDF, and consider the Optical PDF entropy (distance from uniform distribution). If it is very close to a completely uniform distribution, then it should get a high score because of lack of information (This should be weighted by the standard deviation). If it is very close to the target distribution, then it should get a high score because of actual precision (so this should be weighted by 1/std). 
This actually could be a good way to do it, provided that we actual have an efficient way to calculate entropy, and additionally, that we have relatively good amount of information about the PDFs themselves, since that is required for analytic determinations of relative entropy and entropy.

# Current Final Score Calculation
- JSD between base_pdf and test_pdf
- JSD between test_pdf and uniform distribution (Potentially Reconsider this)
- Wasserstein Distance between test_pdf and base_pdf

The JSD between base_pdf and test_pdf is an information theory metric to measure how similar the probability distributions are aligned. This is useful to make sure that the test_pdf and base_pdf have similar shapes, but this does not account for alignment, and therefore it would penalize a delta_function which is well localized at the mean of the base_pdf, simply because it doesn't have the same shape as the base_pdf. 
The JSD score is muliplied by the uniform distribution score. This allows it so that a very well-localized peak in a PDF has a final score that depends a lot on its JSD value. However, a very poorly-localized peak in a PDF has a final score that is also very close to 1, but this is because we do not have enough information about the PDF. 
The information theory metrics should be prioritized less than the actual alignment score, which is determined by the Wasserstein Distance. Currently, the weights are 0.4 (information theory metrics) and 0.6 (alignment), however this can be adjusted.