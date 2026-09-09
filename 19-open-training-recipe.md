# 19 Detailed open-version training recipe

## 1. Scope

Minimal viable: Lean 4 + Mathlib, 300K Mathlib pairs, open 1B
encoder-decoder (start point AlphaGemma-1B), SFT, then RLO s = main
RL with 100K-1M autoformalized problems, then TTRL. All numbers
below are concrete and tuned per budget budget (S).

## 2. Environment build (week 1-2)

- Lean 4 + Mathlib pinned commit, `mathlib4` from GitHub;
- custom tactics: `linarith` compile-to-C, heartbeat multiplier.
- Env interface (Python, state-keyed):

```
class TacticEnv:
    def init(self, statement: str) -> State   # builds state from file
    def next(self, s: State) -> list[State]   # one-step expansions? no:
    def apply(self, s: State, a: str) -> State | Invalid
    def decompose(s) -> list[State]           # AND-split
    def negate(s) -> State                    # reflect/disprove
    def hash(s) -> str                        # canonical state id
```

- state id must be canonical (renaming + hypothesis ordering) for
  dedup/merge.
- theorem statements obtained from `{problem}.lean` files or Lean
  `#check` API; and goal-state pretty-print for observability.

## 3. Data pipeline (S)

Mathlib trace extraction: instrument the Lean elaborator
(`elab` hooks) to emit `(full_state, tactic)` tuples; filters:
state and tactic token caps; no `sorry`; go over all of Mathlib
(a few hours at full scale). Est.: 300K–500K pairs, 5M tactic
tokens.

Autoformalized curriculum: run fine-tuned LLM (Qwen/QwQ,
vLLM batch) on 1M natural statements → 80M (at m ≈ 80 avg picks);
verify each candidate parses (Lean `#check` single-threaded);
purge statements that are identical/similar to eval (redaction
execution: dedup on AST-hash + ≥80% string overlap to any eval
problem, plus remove any exact form).

## 4. SFT (1 H100 day-equivalent)

- model: 1B (from open E/D checkpoint; encoder + decoder attn),
  new value head: $p_v$ over $B=512$ buckets.
- LR schedule: 5K warmup + cosine to 0, peak 2e-5, weight decay
  0.03? (S: 0.1), clip 1.0, bf16.
- Data: SFT 300K pairs (95/5 split); tokens/batch = 10^5. 3 epochs.
  Epochs don't matter much at these scales (monitor CE + MMLU-like
  purity check: spurious memorization).

## 5. Main RL (facial)

**Actor nodes**: pool of CPU workers, each: Lean env process; get
task (problem, budget B) from matchmaker; run tree search via
Python loop calling the network on the GPU side (batched): every
step:

$$
N_{\mathrm{sims}} \uparrow B, \quad
\mathrm{outcome} = \mathrm{proved/disproved/timeout}
$$

**Matchmaker** (queue): per-problem hashtable; state in
{new, attempted, proven, disproven}; score:

$$
\mathrm{priority}(p) = \mathbf{1}\{\mathrm{unattempted}\}\vee
\mathbb{1}\{n_p < TC\} \vee \mathbb{1}\{0 < s_p < 1\}
$$

$$ \mathrm{budget}(p) = \min\big[ B_{\mathrm{cap}}, B_0 \cdot 2^{n_{\mathrm{fail}}(p)} \big] $$

**Learner** (GPU): 10% SFT + 90% replay (replay buffer = most recent
N_success states-actions-of-successful-paths; size S: 2e6; priority
sampling? no: uniform; add dedup by state hash to prevent repeated
identical gradients).

Training config (S): batch 512 pairs; each sample = (raw state
tokens ≤ 2048? 4096; action ≤ 128), bf16, grad acc 4 to get 2048
effective batch, LR 3e-5 (peak), 3e-6 (end), cosine, 1M updates;
AdamW, β = (0.9, 0.95), wd 0.1, grad clip 1.0; eval on miniF2F +
formal-imo at ever 10k updates with sims 4k.

**Throughput model**:

$$
\mathrm{sims/s} = \frac{\mathrm{actors}}{K \, \bar{t}_{\mathrm{exec}} + t_{\mathrm{net}}},\quad
\bar{t}_{\mathrm{exec}} \approx 0.1\text{–}1 \, \mathrm{s}
$$

Key engineering detail: keep execution time (Lean side) under 1s:
the wall-clock limit in the environment is 10s but the median
should be 0.1-1s.

## 6. TTRL recipe (S)

- After main RL: pick targets T evaluated unsolved at 12 TPU-hours
  equivalent (i.e., the hardest);
- Curate variants as in 18: 1000-100k; validate in Lean;
- Focused run: same as main RL but curriculum = {T} ∪ V_T, small
  learner L (same model), single target at a time; eval every
  10 TPU-days of TTRL: solve-rate at 4k sims;
- Stop on solve.

## 7. Compute accounting (formulas)

- SFT: 1 GPU-day (1B, 100-epoch x 3);
- Main RL: $C_{\mathrm{RL}} \approx \mathrm{sims} \cdot (t_{\mathrm{exec}} + t_{\mathrm{net}}) \cdot \mathrm{actors}$. At 8 × H100: e.g., 6-8 days of 1M sims-cycles on 1M problems... rule: 50-GPU-days per 1M attempts? -> make concrete: expected 1M sims/day with 16 CPU actors (median t_exec = 0.3s, K = 64: tree-sim time ≈ 0.3s × 64 + 0.02 = 20s per sim): 30K sims/day? (S, measure after env build).
- TTRL: 20-40 GPU-days per target at 1B scale.

Milestones (acceptance):

1. miniF2F-valid SFT-only ≈ 30% @ 4k sims (like paper's 0 day pt);
2. main RL 100K steps → miniF2F-valid 70%+;
3. TTRL on formal-imo: +5-10 pp beyond search-only;
4. Recompose: IMO 2024 P1/P2/P6 replay with TTRL (post-frozen).

## 8. What to measure and log (metrics)

- per-search: sims, tactics executed, wall-clock/GPU-days, outcome;
- per-training-loss: pol CE, value CE, entropy;
- per-rollout: attempt success, disproof, timeout ratio;
- per-benchmark-eval: solve rate vs sims (S(B) curve, 3+ budgets);
- latency per environment ops; Lean process crashes (kill/crash
  rate); dedup hit rate.

## 9. Risks (mitigations to pre-install)

- env overflow (Lean state too big) → truncation hook;
- tactic explosion (cancel the `interval` tactic explosion by
  wall-cap);
- server crash: restart-env per actor
  (auto-restart worker pool);
- replay stagnation: trigger "fresh interest" reset by
  matchmaker’s interestingness formula (already built-in);
- state bloating: hash-canonicalization at env level is mandatory
  (measured 5-20% saving in sims — planned E-fix).
