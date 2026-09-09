# 17 Design choice ledger (all knobs + why + how to test)

Format per choice: name / why / suggested / experiment.

## A. Model family

- Purpose: encoding long tactic states + emitting short tactic
  strings. Encoder-decoder (paper) matches this asymmetry: full
  context encoding, cheap decoding.
- Alternative: decoder-only (R1/GPT-style). Trade: no dense shared
  context state: each token re-reads everything (cost); but simpler.
- Suggestion: keep encoder-decoder as paper, start from an open
  code+math pretrained model (AlphaGemma-1B/ROMAP).
- Test: 20 E14 (same corpus, same eval).

## B. Value output

- Options: scalar regression (MSE on $d$), categorical (paper,
  MuZero ref 72), Gumbel-sigmoid.
- Why categorical: heavy skewed tail (many states are close to
  done, few near hopeless), and better calibrated miscalibration
  handling; scalar MSE understates long-proof states.
- Suggestion: categorical, $B = 512$ buckets, $D_{\max} = 512$,
  linear spacing.
- Test: E5.

## C. Q-form and $\gamma$

- Paper formula $Q = \gamma^{-V-1} = \gamma^{d-1}$; alternative:
  min-max normalize sibling values (AlphaZero-style).
- Why choose exponential: scale-invariant under constant value
  errors; sibling normalization can collapse when only one child is
  bad; no static min/max.
- Suggestion: $\gamma = 0.99$.
- Test: E1 (γ), E2c (Q-form A/B).

## D. Prior in PUCT: raw probabilities vs temperature power

- Paper: $\pi^{1/\tau}$: temperature is a prior-flattener.
- Suggestion: $\tau = 1.0$ initially; 0.5/2 in E4; options: no
  temperature (raw $\pi$), τ<1 for trust, τ>1 for spread.
- Test: E4.

## E. Sampling temperature of generation (K tactic draws)

- Options: greedy (K top), sampling τ = 0.7-1.0, top-p 0.9-0.98.
- Why: candidate diversity vs quality. High τ during early state
  exploration, low when near proof end (curriculum).
- Suggestion: $\tau_{\mathrm{gen}} = 0.8$, top-p 0.95, dual-phase.
- Test: E4.

## F. Expansion candidates count $K$

- Options: 16 to 128. Impact: each expansion costs executing K
  tactics; K=64 default (AlphaProof-like); higher K helps when
  prior is weak.
- Suggestion: 64.
- Test: E4 (K sweep).

## G. Progressive sampling constants (C, α)

- Options: C ∈ {0.1, 0.5, 1.0, 2.0}, α ∈ {0.25, 0.5, 0.75};
  identity: n(s) ≤ C N(s)^α.
- Why: AlphaProof's main divergence from plain UCT: opens the
  candidate pool dynamically on nodes along the current path.
- Suggestion: C = 1.0, α = 0.5.
- Test: E3.

## H. AND-node multiplier c_AND

- Options: {0.5, 1, 2}. Low = lean on values; high = explore.
- Suggestion: 1.0.
- Test: E3b.

## I. Budget allocation in matchmaker (B0, m, cap)

- Suggestion: B0 (per attempt) = 500 & grow ×2 per successive
  failure to cap 15k; window 25; trust_count 5.
- Alternatives: fixed budget, exponential per-fail, or
  allocate-on-progress.
- Test: E6 (matchmaker variants) — measure: solved-per-TPU-day,
  disproof rate, curriculum completion.

## J. Training data mix (SFT:replay 1:9)

- Options: pure replay; 1:9 (paper); 5:1; plus "student-teacher
  stability". Case: replay-only can drift to memorizing search
  idioms; SFT keeps system prompt of robust tactics (linarith etc.).
- Suggestion: 1:9.
- Test: E7.

## K. Value target: exact MC (λ=1) vs λ-returns/Bootstrap

- Observations: exact returns deterministic in proof-trace: λ=1
  performs as "exact labels" and no bias; but slower convergence vs
  TD(0), needs full-horizon consistency.
- Suggestion: λ = 1.0 (then 0.9 in E6-δ).
- Test: E6.

## L. Pretraining corpus choice

- Options: code-only; math-only (arXiv + textbooks); mixed 1:1:1
  (S).
- Why: encoder must read code-like **and** math grammar.
- Suggestion: mixed, 300B tokens if budget; else reuse open model.
- Test: indirect (E14: open vs trained-from-scratch).

## M. Value head bucket spacing (linear vs log)

- Options: linear (equal spacing over d), log (more resolution near
  d=0: near-proof states). From MuZero for Atari scores ±300 uses
  uniform; but error curves: uniform works.
- Suggestion: linear; test log in E5.
- Test: E5.

## N. Per-attempt budget scaling & search repetitions

- Options: one long search (AlphaProof: single search); many
  short searches (classic "rollout with relabel"); hybrid.
- The paper: single search per attempt, no commits — best use of a
  fixed total B. Short-window restarts lose tree-level context.
- Test: E8 (reuse vs restart).

## O. TTRL focus time / curriculum

- Options: running TTRL only until target solved (paper) vs
  continuing for margin; variant quality hard vs easy (curve).
- Suggestion: stop at solved (budget-efficient); hard-variant
  curriculum; see 18.
- Test: E11 (difficulty schedule), E11b (continue-after-solved).

## P. Tolerance to disproofs

- Options: train on disproof-branch data (paper: yes, mixed) vs
  proof-only. Disproofs carry negation-relevant path structure.
- Suggestion: yes (both); consider classifying loss weights
  1:0.1 proof:disproof? (S: equal, monitor).
- Test: E13.

## Q. Exploration noise for early-shape search

- Options: root Dirichlet ε = 0.25 α = 0.3 (S), progressive
  sampling; no noise.
- Test: E10 (root noise on/off).
