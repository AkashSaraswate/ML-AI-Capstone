# Model Card: BBO Optimisation Approach

## Overview

**Name:** Fingerprinting the Black Box, a Bayesian optimisation strategy with
benchmark identification and per function strategy routing.
**Type:** Sequential model based optimisation (Gaussian Process surrogate plus hybrid
acquisition), with bespoke strategies for functions whose behaviour breaks the GP
assumptions.
**Version:** v2, round 10 of a 13 round challenge.

## Intended use

**Suitable for:** Maximising expensive, low to moderate dimensional black box functions
under a tight evaluation budget, where each evaluation is costly and only a few samples
can be collected. This mirrors hyperparameter tuning and experimental design.

**Use cases to avoid:** Very high dimensional problems where a Gaussian Process covers
the space too sparsely to be reliable, functions with extremely sharp near
discontinuous peaks where a smooth surrogate misleads (unless handled by a bespoke
model, as F1 required), large batch queries, and any setting needing a calibrated
global model of the function rather than just its maximum.

## Details: strategy across the rounds

The approach has two layers: fingerprint each function, then optimise. Where a function
matched a known benchmark it was solved directly; otherwise a Gaussian Process ensemble
drove the search. Each function was routed to one of a small set of strategies based on
its observed signature: solved lock, narrow peak bank, exploit, explore, or bespoke.

- Round 1: broad, near space filling sampling to learn each surface from the initial ten
  points.
- Round 2: moved to a three kernel GP ensemble (Matern nu=2.5, Matern nu=1.5, RBF, each
  with a WhiteKernel noise term and per dimension length scales) scoring Latin Hypercube
  candidates by a normalised hybrid of Expected Improvement, Probability of Improvement
  and Upper Confidence Bound, after a single GP proved over confident.
- Round 3: added heuristic overrides, lowering the UCB exploration weight to exploit when
  a function declined for two rounds.
- Round 4: trained a neural network surrogate (MLP) and read its numerical gradients to
  rank which dimensions drive the output; added a logistic regression classifier that
  binarises outputs at the 70th percentile to confirm the influential dimensions.
- Rounds 5 to 7: added a duplicate guard excluding candidates within a set radius of any
  past query, after a wasted repeat query; tightened the local search radius on functions
  whose peaks proved sharp and spiky (F4, F6).
- Rounds 8 to 10: an endgame of banking confirmed peaks and concentrating the remaining
  budget on the one function still improving. The key correction came on the radiation
  function: re ranking its full dataset by output value revealed the strongest reading
  sat in a region the earlier search had ignored, after which an inverse square decay
  model triangulated toward the source.

## Performance

Measured by the challenge's normalised score,
score = (f(x*) - f_initial_best) / (f_true_max - f_initial_best), how far each function's
best value has climbed from its initial best toward its true maximum. Results after round
10, with the absolute gain over the initial best in brackets:

| Function | Dim | Status | Initial best | Current best | Note |
|----------|-----|--------|-------------|-------------|------|
| F1 | 2D | Active hunt, recovering | ~0 | ~2.3e-12 | Climbed from 1e-59 to 1e-12 over three rounds after a correction |
| F2 | 2D | Banked, narrow peak | 0.611 | 0.723 | +0.112 |
| F3 | 3D | Plateaued near zero | -0.035 | -0.008 | +0.027 |
| F4 | 4D | Banked, confirmed peak | -4.026 | 0.350 | +4.375, crossed into positive |
| F5 | 4D | Solved: 100% of ceiling | 1088.9 | 8662.5 | Monotonic climb to the domain corner |
| F6 | 5D | Banked | -0.714 | -0.159 | +0.556 |
| F7 | 6D | Solved: 100% of ceiling | 1.365 | 3.3224 | Hartmann 6D known optimum |
| F8 | 8D | Banked | 9.599 | 9.918 | +0.319 |

Two functions (F5, F7) reached 100% of their true maximum. F4 recovered the most in
absolute terms. Four of eight improved on their initial best in the first week, and the
endgame protected those gains while F1 continued to climb.

## Assumptions and limitations

**Assumptions.** Smoothness, that similar inputs give similar outputs, which makes the GP
valid; and determinism, confirmed for several functions by identical re queries returning
identical values.

**Limitations and failure modes.** The smoothness assumption fails on sharp peaked
functions, where the surrogate is well fit to observed points yet systematically wrong
between them, which misled the F1 search for several rounds (the optimizer's curse under
extreme data sparsity). Sampling is biased toward exploited regions, so the dataset is not
an even picture of each function. In eight dimensions the budget is too small for
confident coverage, so F8 should be read as a strong local optimum, not a global one. The
central failure mode is trusting a confident surrogate when the true function violates its
assumptions.

## Ethical considerations

Transparency is the main safeguard. Every query, returned value, strategy label and
reasoning note is logged, so another researcher could reproduce the search and follow its
logic, including its corrections. The F1 misread was caught by re ranking the full
evidence, and it is documented here rather than hidden, because a reproducible process
must record its own mistakes. In real world adaptation, this discipline of checking a
model's suggestions against the full evidence rather than the latest result is what makes
an optimisation approach safe to rely on. The functions are synthetic with no personal or
sensitive data, so the ethical weight falls entirely on honest reporting of uncertainty.
