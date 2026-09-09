# 20 Experiment suite (protocol for every choice)

## 0. Common methodology

- **Metric**: solve rate $S(B)$ at budget $B$ (sims per problem) or
  TPU-days per problem; auxiliary: sims-per-solve, mean proved length
  in tactics, disproof rate, wall-clock ops.
- **Budget**: always measure at matched *total compute* (wall-clock
  plus GPU time), not just matched sims.
- **Seeds**: ≥ 3 independent seeds per condition.
- **Statistics**: paired per-problem data; report mean ± 95%
  bootstrap CI:

$$
\mathrm{CI}_{95} = [ \widehat{\theta}^\ast_{0.025},\;
\widehat{\theta}^\ast_{0.975} ] \quad \text{(percentile over 10k resamples)}
$$

  significance test: permutation-based paired difference on per-problem
  solve vectors (p < 0.05, pre-registered).
- **Data**: open, fixed splits (miniF2F-valid, formal-imo,
  PutnamBench-test at 189), from the redaction rules (08 §1).
- Files templated: `src/experiments/e<NN>_*.py`, conf JSON, logs to
  `artifacts/`.
- **Decision rule** (each experiment): adopt option B if
  point estimate improves and lower 95% CI > 0; otherwise keep
  default; prefer simplicity.

## E1. Search discount γ

- Question: the γ in $Q = \gamma^{-V-1}$ (only paper formula).
- Setup: SFT checkpoint + 200k-step RLO; miniF2F-valid; 4k sims/attempt.
- Grid: γ ∈ {0.9, 0.99, 0.999, 0.9999}, all else constant.
- Metric: S(4k) + mean proved length.
- Expected pattern: small γ finds short proofs fast (high S at low
  budget, saturates); large γ captures longer proof paths; pick the
  γ with the best S(4k); for TTRL-style long proofs, re-check with
  γ=0.999 config.

## E2. Q-form & exploration factor

- (a) Q-form: exponential (paper) vs sibling min-max normalization.
- (b) c_init × c_base grid: c_init ∈ {0.1, 0.5, 1.25, 3},
  c_base ∈ {10, 10^2, 10^4, 19652}.
- Metric: S(4k) at matched sims; log learning-rate of policy
  entropy vs sims.
- Expectation: exponential Q-form becomes more robust as value-gap
  increases; c_init too high wastes sims on junk branches (slower S),
  too low loses proof coverage (asymptotic low S).

## E3. Progressive sampling (C, α)

- Condition check: n(s) ≤ C·N(s)^α triggers K new tactics.
- Grid: C ∈ {0.1, 0.5, 1.0, 2.0}, α ∈ {0.25, 0.5, 0.75}.
- Measure: S(4k), and *novel tactic* hits per state (fresh-edges
  expansion rate) for validation of the trigger's behavior.
- Hypothesis: optimal (C, α) stronger with low-quality value net
  (SFT), weaker after training, confirming the paper's focus on
  critical paths.

## E4. K and τ_prior / τ_gen

- K ∈ {16, 32, 64, 128}; τ_prior ∈ {0.5, 1.0, 2.0};
  τ_gen ∈ {0.6, 0.8, 1.0} with top-p ∈ {0.9, 0.95, 0.98}.
- Do one full factorial (with spot reduction at seeds=2), then
  refine.
- Metric: per-state candidate validity rate, S(4k), wall-clock.

## E5. Value head: buckets, D_max, spacing

- B ∈ {64, 256, 512, 1024}; D_max ∈ {256, 512, 1024, 2048};
  spacing linear vs log.
- Evaluate: value-error proxy (correlation net V vs search V),
  S(4k).
- Expectation: log spacing improves early stopping (near-done
  states) but might degrade TTRL precision; if performance equal,
  keep linear (simpler).

## E6. Value target λ and matchmaker details

- λ ∈ {0.0, 0.5, 0.9, 1.0} (exact-MC vs TD).
- N-window ∈ {10, 25, 50} (interestingness), trust_count ∈ {2, 5,
  10}, B-cap schedule ∈ {fixed, ×2, ×3}.
- Metrics: S(4k) and curriculum progress (proved/disproved/timed-out
  fraction vs TPU-days).

## E7. SFT:replay ratio

- {0:10, 1:9, 1:4, 1:1} (the paper: 1:9 default).
- Watch drift of policy entropy and ECF-grade `linarith` usage rate;
  and recall degress on miniF2F.

## E8. Single-search vs restarts

- Compare one full-B search vs B/4-restarts × 4 (with full reset);
- metrics: S at equal total sims.

## E9. RL objective family (the GRPO question)

- Conditions: (a) expert iteration (paper; CE to search-chosen
  actions), (b) PPO+critic with same value, (c) GRPO (group of 8
  rollouts per statement), (d) REINFORCE w/ GAE, (e) DPO on
  (good vs prior sampled) contrastive pairs.
- Setup: identical statement data and clock budget; generate
  experience with a frozen mid-training snapshot each 50k steps so
  all conditions see the same rollouts; then each trains its updated
  policy on the shared sample.
- Metrics: S(4k) curve, sims-per-solve, value-correlation, sample
  efficiency per GPU-day.
- Expected (prior evidence): (a) best or tied-best; (c) competitive
  when rollout cheap; (e) fast but needs good negative samples.
  Decision rule: if (a) not clearly best, switch to the winner at
  the same simplicity threshold.

## E10. Root Dirichlet noise on/off (ε, α)

- {off, ε=0.15, ε=0.25, ε=0.4}, α ∈ {0.1, 0.3, 1.0}; and note
  interaction with progressive sampling.
- Metric: S(4k) distinctly from cold-start (first-branch) success.

## E11. TTRL curriculum schedule

- Variant difficulty ordering: easy-first / hard-first / random;
  variant set size: {10, 100, 1000, 100k (Top-10 vs Top-100k in
  paper's Fig 3 ablations)}; generator LLM: 1.5 vs 2.0-ish power
  classes (try two open models).
- Metrics: fraction of variants solved in the TTRL training set;
  target solve time (TPU-days to solve T); final S(T).
- Expected: hard-first wins (paper's Extended Data Fig. 3d), larger
  set wins (quantity monotone Fig 3b). Freeze the best.

## E12. Tokenizer

- Vocab {32K, 64K, 128K, 256K}; BPE (greedy vs minBPE) vs unigram
  (SentencePiece); model fixed, evaluate CR (11 §4) and S(4k) after
  100k RL steps with the tokenizer swapped.

## E13. Disproof data use

- Train on (a) proof-only, (b) proof+disproof (paper), (c)
  proof+disproof with 0.1 disproof weight (S).
- Metric: S(4k), disproof-rate and solved rate (on curriculum), and
  step correlation.

## E14. Network family

- Encoder-decoder vs decoder-only at matched ~1B params: same env,
  same data; token budget, S(4k).
- Hybrid answer: keep the paper's structure if encoder-decoder wins
  throughput (it will, long-state): the winner decides.

## E15. Compute-normalized eval for the "I am later" user

- Every experiment re-run at fixed GPU-day budget; final report as a
  stats-blob `perf_summary.json` with the raw numbers to be
  automatically updated by you later.

The highest-value first three experiments (order for your hands):
E1 γ, E4 (K, τ), E6 λ — all cheap, unambiguous signal, and flip a
single knob each.
