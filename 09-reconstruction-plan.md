# 09 Open-source reconstruction plan

Goal: a Lean-based AlphaProof-style prover reproducible by a
small/medium lab. Mapping of paper internals to open components,
with compute/target choices.

## 1. Component map

- Proof assistant: Lean 4 (open) + Mathlib (open). Needed env ops
  (01 §4 all doable via: Lean `mono` protocol, lake projects, custom
  tactic-compile to C, heartbeat/wall-clock patches, int cap patch).
- State ops: snapshot/restore id, parallel evaluation — implement via
  a persistent process pool + lean server; state normalization:
  pretty print + sort hypotheses + canonicalize names.
- Proof network: encoder-decoder from an open code+math pretrained
  model (recommend AlphaGemma-1B or similar encoder-decoder; both
  close to the described pretraining corpus, 1/3.5 of the 3B target).
  Skip the proprietary 300B-token pretraining step; the 40+10 epoch
  pretraining from open corpora is available for scale-up later.
- Policy/value heads: default decoder LM-head + categorical head
  (buckets D_max = 512, 02 §3).
- Auto-formalizer: open instruct LLM (QwQ-32B / Qwen2.5-27B /
  DeepSeek-V3) + CoT + Lean-output schema; STaR loop with
  equivalence checker = *this same prover in low-compute mode*
  (05 §3).
- RL infra: JAX or PyTorch async actor-pool + GPU learner;
  matchmaker in a small service (06 §2).
- TTRL variant generator: same LLM, few-shot Ω ≈ 791 pairs; plus
  programmatic Σ-ops (07 §1).
- Benchmarks: open — Mathlib-parse miniF2F (corrected),
  formal-imo (google-deepmind/formal-imo), PutnamBench commit as in
  paper.

## 2. Scaled parameter table (S, tiered)

Tier M (mid, ~8×H100-GPU-days scale):

- model 1B (d = 1536, enc 24, dec 12),
- state cap $L_{\max} = 2048$, $K = 32$, $\tau = 1.0$,
- budgets $B_0 = 300$, $B_{\mathrm{cap}} = 8{,}000$,
- curriculum: $10^{6}$ autoformalized statements,
- RL steps: $2\times10^{5}$, batch 512 pairs, cosine 2e-5.

Tier L (paper-scale): as §3, using 3B params and paper-derived
numbers (γ = 0.99, K = 64, 80M curriculum, 1M steps).

Tier S (small / canary, 1×4090 24h):

- model 400M, miniF2F only, SFT + 5K RL steps; used to smoke-test the
  entire stack; expected miniF2F-valid ≈ 20% (baseline sanity).

## 3. Phase plan with acceptance criteria

- A. Env (1–2 wk): Lean4 + mathlib builds; tactic exec from Python;
  validity + AND-split + reflect + verification replays; benchmark:
  random-tactic + `linarith`-hammer baseline on miniF2F-valid must
  match published noise floor (≈1%).
- B. Data (1 wk): Mathlib trace extractor → 300K+ (state, tactic)
  pairs; SFT 1B model; accept: miniF2F-valid ≈ 30 % at 1k sims
  (SFT-only, cf. paper's 0-TPU-day point).
- C. Main RL v1 (2–4 wk after B): actors(CPU) + matchmaker + learner;
  curriculum: miniF2F-curriculum + 3.5K human + 100K autoformalized;
  accept: miniF2F-valid ≥ 60 % at 4k sims; S(B) curve shape matches
  Fig. 3c (efficiency gain visible across 2 checkpoints).
- D. Auto-formalization (ongoing): LLM formalizer on 100K→1M natural
  statements from open corpora (MATH, Omni-MATH, olympiad archives);
  STaR-equivalence filter via low-compute prover; accept: pass@1
  ≥ 45% on the paper's 50-IMO/50-Putnam style internal set;
  dedupe/redact vs eval benchmarks.
- E. Scale-up RL with 80M-style curriculum; accept: formal-imo ≥ 20%
  at 12 TPU-h equivalent; PutnamBench-test ≥ 15%.
- F. TTRL: variant generator + focused loop on 20 formal-imo hard
  problems; accept: +8pp on formal-imo at 50 GPU-days equivalent;
  IMO-2024-6 replay: verify P1/P2/P6 proofs fresh (post-hoc,
  curriculum-frozen baseline).

## 4. Compute model for planning

Per RL update (FLOPs): $6 N_\theta \cdot \mathrm{batchTokens}$.
Tactics dominate: search throughput estimate:

$$
\frac{\mathrm{sims}}{\mathrm{s}} \approx
\frac{\#\mathrm{workers}}{K\,(t_{\mathrm{exec}} + t_{\mathrm{nn}})}
$$

Equivalences (paper-scale): main RL 80K TPU-days ≈ 4,000 TPUs × 20 d;
auto-formalization 100K TPU-days; SFT 10 TPU-days; TTRL 50–500
TPU-days per problem; IMO 2024 runs 2–3 d each. Rule of thumb for
budget planning: 1 TPU-v6e-day ≈ 0.6–0.8 H100-BFP16-day (rough
FLOPs equivalency, verify locally).

## 5. Suggestion set (all flagged S) summarized

α = 0.5, C = 1.0, γ = 0.99, c_init = 1.25, c_base = 19652, K = 64,
τ = 1.0, c_AND = 1.0, c_pen = 1.0, μ = 1.0 (cost), T_max = 1024,
tactics wall cap 10 s, TC = 5, N = 25, B_0 = 500, m = 2,
B_cap = 15000, value bins 512, λ-return 1.0, optimizer AdamW
(3e-5 → 3e-6 cosine), SFT:RL mix 1:9.

## 6. Risk register (short)

- Prover performance is dominated by search cost: hyperparameter
  drift (γ, α, c*) shows as orders-of-magnitude sims differences —
  tune on miniF2F-valid first.
- Auto-formalization quality sets the ceiling: with 60% pass@1 never
  substitute noisy statements without the disproof/reprove
  redundancy.
- AND-node semantics: wrong min/max convention silently destroys
  balancing (01 §2, derivation given).
- Value head histogram range D_max must exceed proof lengths; garbage
  for states of length > D_max — clamp + monitor tails.
- Data leakage: any formal problem similar to eval sets in the
  curriculum invalidates the benchmark; enforce dedup at pipeline
  input (08 §1 rule).

## 7. Open questions worth minimal-ablation experiments

1. Best γ for step-domain Q-transform (paper formula given, γ ours).
2. Progressive sampling (C, α) vs budget scaling.
3. Curriculum difficulty curve in TTRL (2.0 Flash-S evidence: prefer
  hard variants).
4. Value head bucketing: log vs linear; report tails.
5. Whether disproof-side states should be included for policy
  training separately from proofs (paper includes both).
