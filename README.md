# Fingerprinting the Black Box

**Identifying and optimising eight unknown functions under a tight query budget.**

Imperial College Business School · Professional Certificate in ML and AI · BBO Capstone

## Documentation

- [Datasheet](DATASHEET.md) — the query dataset: motivation, composition, collection, uses.
- [Model card](MODEL_CARD.md) — the optimisation approach: intended use, strategy, performance, limitations.

## Non technical explanation

Imagine eight hidden machines, each with a few dials. Turn the dials and a machine
returns a single score, but you never see inside it and every test costs you. The goal
is to find the dial settings that score highest for each machine, using as few tests as
possible. This project does that with Bayesian optimisation: after each test it builds a
statistical guess of how each machine behaves, then picks the most promising setting to
try next. It also tries to recognise each machine, because if one turns out to be a
known problem, the answer can be looked up rather than searched for. Over weekly rounds
it climbs toward the best settings while tracking progress for all eight.

## Inputs and outputs

Each function takes one point in the unit hypercube, every coordinate between zero and
one, submitted as a hyphen separated six decimal string (for example
0.201690-0.150011-0.476874 for a 3D function). The output is a single scalar. All eight
functions are maximisation problems; where the real objective is a minimisation it is
negated so higher is better.

## Objective and constraints

Maximise each function using as few queries as possible. The function structure is
unknown, only ten initial observations are provided per function, one query per function
is allowed per week, and the challenge runs over thirteen rounds. Progress is scored by
how far the best value climbs from the starting point toward the true maximum.

## Technical approach

Two layers: fingerprint, then optimise.

The fingerprinting layer compares each function against known optimisation benchmarks.
This paid off on F7, which matched Hartmann 6D almost exactly, letting its known global
optimum be submitted directly to reach the true maximum in one query.

Where a function resists identification, a Gaussian Process ensemble drives the search:
three kernels (Matern 2.5, Matern 1.5, RBF) averaged, with disagreement folded into the
uncertainty, scoring Latin Hypercube candidates by a hybrid of Expected Improvement,
Probability of Improvement and Upper Confidence Bound. The exploration weight anneals
down as the budget is spent. A duplicate guard prevents re proposing an already tested
point.

Two functions need bespoke handling. F5 climbs monotonically to the domain corner, so
its maximum sits at all ones. F1 is a sharp radiation source, not a smooth surface; it
is handled by a physics style decay model that triangulates the source from the
strongest readings, after an early misread was corrected by re ranking the full dataset
by output value.

## Results so far (Round 10)

| Function | Dim | Status | Best |
|----------|-----|--------|------|
| F1 | 2D | Active hunt, climbing after correction | ~2.3e-12 |
| F2 | 2D | Banked, narrow peak | 0.7234 |
| F3 | 3D | Plateaued near zero | -0.0081 |
| F4 | 4D | Banked, confirmed peak | 0.3496 |
| F5 | 4D | Solved: 100% of true max | 8662.48 |
| F6 | 5D | Banked | -0.1585 |
| F7 | 6D | Solved: 100% of true max | 3.3224 |
| F8 | 8D | Banked | 9.9177 |

The key lesson: recognising a function's identity can beat brute force search, and a
transparent, logged process is what let a wrong early assumption on F1 be caught and
corrected.

## Setup

```
pip install numpy scipy scikit-learn matplotlib pandas
```

The initial_data folder is not included (data is proprietary to the challenge). Create
it with one subfolder per function, each holding initial_inputs.npy and
initial_outputs.npy, then set the data path at the top of the notebook.

## Contact

Akash Saraswate
