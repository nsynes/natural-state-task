# Bird challenge: from BirdNET predictions to observations

**Contents**
- [Bird challenge 1: observation labels for each prediction](#bird-challenge-1-observation-labels-for-each-prediction)
  - [Summary](#summary) · [Data](#data) · [Method](#method) · [Results and decisions](#results-and-decisions)
- [Bird challenge 2: hand-off to the Tech team](#bird-challenge-2-hand-off-to-the-tech-team)

## Bird challenge 1: observation labels for each prediction

### Summary
- Following Wood & Kahl (2024), I fitted a logistic regression per species relating BirdNET's
  (logit-transformed) confidence score to whether an ornithologist judged the prediction correct,
  and set the threshold where the fitted probability of a correct prediction reaches **0.99**.
- Only **Abyssinian Nightjar** produced a reliable threshold: **BirdNET score ≥ 0.667**, labelling
  **3,147 of 10,187** predictions (31%) as observations.
- The other three species get **no observations**: the validation data can't yet demonstrate 99%
  precision for them. Oriole and Plover have too few incorrect clips to fit a curve, but many
  low-scoring predictions are available to validate, so a threshold may become possible.
  For Firefinch, BirdNET is mostly wrong and there are almost no high-scoring predictions to
  validate, so a reliable threshold is unlikely.

Outputs: `data/processed/birdnet_observations.csv` (all 29,491 predictions with a new `observation`
field) and `data/processed/species_thresholds.csv` (one row per species: fit, threshold and status).
Analysis: `bird-data-investigation.ipynb`.

### Data
| Species | Predictions | Validated | Correct | Incorrect |
|---|---|---|---|---|
| Abyssinian Nightjar | 10,187 | 150 | 117 | 33 |
| African Black-headed Oriole | 18,054 | 150 | 150 | 0 |
| Red-billed Firefinch | 235 | 101 | 6 | 95 |
| Three-banded Plover | 1,015 | 150 | 149 | 1 |

Validated clips are spread across the full score range, but this differs from the predictions:
for Nightjar, Oriole and Plover, approximately half of all predictions score below 0.3, while the validated clips
over-represent high scores (see the score-distribution plot in the notebook). Spreading validation across the range
suits curve fitting, but it means relatively few validated clips sit where most predictions,
and most errors, are likely to be.

### Method
1. **Transform the score.** `logit(confidence)`, with scores of 1.0 clipped to 1 − 1e-6 to avoid
   infinite values.
2. **Fit per species.** Binomial GLM `outcome ~ logit_score`.
3. **Solve for the threshold.** Set the fitted probability of a correct prediction to 0.99 and solve
   for the score: `threshold_logit = (ln(0.99 / (1 − 0.99)) − b0) / b1`, where `b0` is the intercept
   and `b1` the slope. Convert back to a BirdNET score with `threshold = 1 / (1 + e^(−threshold_logit))`.
4. **Check the threshold is trustworthy before using it** (this is the key decision, below).
5. **Label.** `observation` = species name where the species' threshold is usable and
   `confidence ≥ threshold`; otherwise empty.

The notebook plots the first-pass fits for all species, before the reliability checks: validated clips
(1 = correct), the fitted probability of a correct prediction with its 95% CI, and the 0.99 line.
Plover's threshold appears there but is rejected (see below). A final plot shows thresholds only where
they passed the checks.

### Results and decisions
| Species | Slope (95% CI) | Fitted threshold | Status | Observations |
|---|---|---|---|---|
| Abyssinian Nightjar | 2.07 (1.10 to 3.04) | 0.667 | Usable | 3,147 |
| African Black-headed Oriole | — | — | No incorrect clips | 0 |
| Red-billed Firefinch | 0.94 (0.07 to 1.82) | 0.998 | Above validated scores | 0 |
| Three-banded Plover | 0.55 (−0.96 to 2.05) | 0.259 | Unreliable fit | 0 |

A threshold is only used if it passes three checks:

- **Both correct and incorrect clips exist.** *African Black-headed Oriole* had 150/150 correct.
  The curve can't be fitted without errors to learn from: the model is undefined, not just uncertain.
- **The slope is clearly positive.** The method assumes higher scores mean more likely correct.
  For *Three-banded Plover* the slope's 95% CI includes zero, so the data can't show that relationship.
  Its single incorrect clip (score 0.264) drives the fit, and since the threshold divides by the
  slope, it isn't reliably estimated.
- **The threshold lies within the validated scores.** For *Red-billed Firefinch* the curve reaches
  0.99 only at a score of 0.998, above every validated clip (max 0.907), so the threshold would be an
  extrapolation. Since 95 of 101 validated Firefinch predictions were wrong, BirdNET is unreliable
  for this species at the scores we have.

**Recommendations**
- *Oriole, Plover:* validate more clips, focusing on low scores. There are plenty available,
and that is where errors are most likely. With enough incorrect clips, a reliable 99% threshold may become possible.
- *Firefinch:* BirdNET is mostly wrong for this species, and there is almost nothing left at high
  scores to validate: 101 of 235 predictions are already validated, and only one scores 0.9 or more.
  A reliable 99% threshold is unlikely to be achievable.

## Bird challenge 2: hand-off to the Tech team
**How it runs.** Two jobs after BirdNET predictions are stored:
1. **Fit thresholds**: run per species (and BirdNET version) whenever validations change.
   Writes a `species_thresholds` table.
2. **Label predictions**: run on every new batch: join predictions to `species_thresholds`
   and compare scores.

**Input requirement.** Validated clips must link to predictions by species code and BirdNET version.
The sample validation file has only common names and filenames, and joining on names is fragile.

**Algorithm**: `fit_threshold()` in the notebook is the reference implementation (Python/statsmodels):
logit-transform the score → binomial GLM → solve for the score at P(correct) = 0.99 →
assign a status. Rules for when *not* to use a threshold:

| Status | Rule |
|---|---|
| `no_negatives` / `no_positives` | All validated clips have the same outcome |
| `unreliable_fit` | Slope's 95% CI includes 0 |
| `threshold_above_validated_scores` | Threshold above the highest validated score |
| `fitted` | Passed all checks → `threshold_usable = True` |

**Suggested next steps**
- Feed species without a usable threshold into a "needs validation" queue for ornithologists,
  using score-stratified sampling.
- Warn when new predictions have scores outside the validated range.
- A dashboard listing each species' status, the observation counts per run, and the distribution
  of confidence scores, as a quick check on pipeline health.
- Move `fit_threshold` into a tested module (one test per status), and consider per-project thresholds,
  since BirdNET's accuracy can vary by site.
