# 28andMe brain states

This project examines whether day-to-day estradiol and progesterone levels are
associated with resting-state brain dynamics in one densely sampled participant.
The current scope is Objective 1 of `Proposal.pdf`: identify recurring
whole-brain activation states and derive session-level measures such as state
occupancy, dwell time, and switching rate.

## Data and current scope

- `data/raw/participants.tsv` contains hormone, cycle, mood, sleep, anxiety,
  stress, and dietary measures for 60 sessions.
- The 60 NumPy files contain parcel time courses. Each has shape
  `(100 cortical regions, 820 TRs)` and must be transposed to `(TRs, regions)`
  for modeling.
- Sessions 01–30 are naturally cycling and are the primary analysis set.
  Sessions 31–60 are oral-contraceptive sessions and are out of scope for now.
- Motion-censored observations appear as whole-TR NaN columns. In sessions
  01–30, 0–13 TRs are missing per session (median 4).

Raw data and the proposal are local inputs and are excluded from version
control.

## Working analysis plan

1. Audit the time courses, missingness, scaling, and hormone trajectories.
2. Standardize parcel time courses consistently and reduce dimensionality with
   PCA before state modeling. Fit preprocessing only on training data whenever
   model selection uses held-out data.
3. Use a shared Gaussian HMM across all 30 sessions as the primary model. A
   shared state vocabulary makes occupancy and transition metrics directly
   comparable across days.
4. Fit session-specific HMMs as a sensitivity analysis. These can assess whether
   sessions differ in complexity, but their state labels and selected state
   counts are not directly aligned across sessions.
5. Compare a prespecified range of state counts using blocked/segment-aware
   held-out likelihood, solution stability across random initializations, and
   basic degeneracy checks (very rare states or implausibly short dwell times).
   Select one state count with the same rule for every session/model comparison;
   do not choose it from the desired hormone result.
6. Derive fractional occupancy, number of occupied states, dwell-time
   distributions, transition count/rate, transition entropy, and related
   diversity measures for each session.
7. Relate those measures to continuous estradiol and progesterone levels, with
   sensitivity checks for time/cycle structure and motion censoring.

The preprocessing audit supports z-scoring every parcel within each session and
using a shared 24-component PCA representation (at least 80% variance retained)
for the primary HMM. PCA will be refit inside each model-selection training fold
and then fit once to all 30 sessions for the final model. The unreduced 100-parcel
representation will be a sensitivity analysis. The candidate state range,
covariance structure, and selection rule remain to be finalized; this keeps
“optimal” explicit rather than equating it with the largest in-sample likelihood.

Initial model selection compares diagonal Gaussian HMMs with 3–15 states using
five session-level folds and ten deterministic initializations per fold and state
count. Matched parcel and PCA pipelines each completed 650 converged fits using
the same folds and seeds. State stability is measured within fold and state count
using all seed pairs, cosine similarity of state means, and Hungarian matching.
Mean matched similarity ranged from about 0.96–1.00 for parcels and 0.69–0.86
for PCA. The plateau rule requires two consecutive mean held-out gains below
0.01 log-likelihood units per valid TR; neither representation reached that rule
within the tested range, and both had their highest mean held-out likelihood at
15 states. No final state count has been selected.

## Missing-TR policy

The primary analysis treats each uninterrupted run of observed TRs as a
separate sequence. HMM fitting therefore estimates no transition across a
motion gap, switching metrics exclude gap boundaries, and dwell periods touching
a gap are treated as censored. This is preferable to assuming either continuity
or a switch when the latent state is unobserved. A sensitivity analysis may
compare the two simpler conventions requested in the project brief.

## Repository layout

```text
data/raw/           Local source data (not tracked)
notebooks/          Reproducible analyses, run in numeric order
results/figures/    Generated figures (not tracked)
results/tables/     Generated tables (not tracked)
environment.yml     Conda environment specification
```

Notebook code should remain simple and direct. Use short comments or compact
Markdown only when they clarify a decision, assumption, or result.

## Environment

A dedicated environment is useful because HMM packages and notebook kernels
are not part of the Python standard library. With Conda installed:

```bash
conda env create -f environment.yml
conda activate brain-states
python -m ipykernel install --user --name brain-states --display-name "Python (brain-states)"
jupyter lab
```

Run Jupyter from the repository root so relative paths in notebooks resolve
consistently. Update the environment explicitly when adding a dependency.

## Project conventions

- Keep the source data immutable; write derived artifacts under `results/`.
- Put analysis work in reproducible notebooks unless a small reusable helper
  clearly reduces duplication.
- Prefer straightforward code over frameworks or premature abstractions.
- Keep prose and comments brief and useful.
- Set and record random seeds for stochastic models.
- Save model-selection diagnostics, not only the selected result.
