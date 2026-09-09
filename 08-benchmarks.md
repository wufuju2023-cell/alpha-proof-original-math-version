# 08 Benchmarks, protocols, measured scaling

## 1. Benchmark inventory

- miniF2F (corrected by DeepMind): high-school competition, public
  `valid` / `test` splits; corrections fix misformalizations
  (disprovable statements, contradictory hypotheses).
- formal-imo: 258 historical (non-geometry) IMO problems,
  expert-formalized: 107 algebra, 77 combinatorics, 74 number theory.
- PutnamBench: undergraduate Putnam; held-out PutnamBench-test = 189
  problems, even years 1990+ (multi-label subjects: algebra 78,
  analysis 57, number theory 33, geometry 20, linear algebra 18;
  sum > 189 because problems carry several subject tags);
  PutnamBench-train = the rest, TTRL not run on it.
- IMO 2024 live: paper's own protocol, §4 below.

Data separation rule (applies to all): pretraining corpus excludes the
Lean code/benchmark proofs; SFT data = Mathlib only (no benchmarks);
main-RL curriculum = autoformal + 3.5K human, with autoformalized
documents similar to any eval problem removed; IMO-2024 run frozen
before the competition released problems.

## 2. Result datapoints (from Table 1 / Extended Data Table 2)

Per-problem-mean compute budgets (search compute, excluding amortized
training):

2 TPU-minutes (≈ 2 min on 1 TPU):

- miniF2F-valid 96.0%, miniF2F-test 96.3%, formal-imo 33.2%,
  PutnamBench-train 35.4%, PutnamBench-test 27.9%.

12 TPU-hours:

- miniF2F-valid 97.1%, miniF2F-test 97.7%, formal-imo 43.7%,
  PutnamBench-train 48.7%, PutnamBench-test 39.4%.

TTRL 50 TPU-days:

- miniF2F-valid 99.6%, miniF2F-test 97.5%, formal-imo 53.9%,
  PutnamBench-test 45.5%.

TTRL 500 TPU-days:

- miniF2F-valid 100.0%, miniF2F-test 99.6%, formal-imo 58.3%,
  PutnamBench-test 56.1%.

Prior state of the art (same metric families, different dataset
editions — compare with care):

- GPT-F expert iteration: miniF2F-test 36.6%.
- Hypertree proof search: 41.0%.
- InternLM2-Math-Plus-7B: 43.4%.
- Kimina-Prover Preview: minF2F-test 80.7%, PutnamBench-test 1.6%.
- DeepSeek-Prover-V2: minF2F-test 88.9%, PutnamBench-test 5.3%.

Subject breakdown (TTRL, formal-imo): number theory 75.7%, algebra
72.6%, combinatorics 20.3%.

## 3. Scaling laws in the measurements

Solve rate as a function of budget $B$ (sims or TPU-hours), per
checkpoint $c$:

$$
S_c(B) = \frac{1}{|\mathrm{Bench}|} \sum_{p \in \mathrm{Bench}}
\mathbf{1}\big[\text{proved with } \le B \text{ sims}\big]
$$

Observed (Fig. 3c / Fig. 4a):

- $S_c$ strictly increases in $B$; later checkpoints dominate earlier
  ones at *every* budget (both power and efficiency):
  $S_{\mathrm{final}}(300\,\mathrm{sims}) \approx 0.30$ vs previous
  checkpoints unreachable;
- search scaling: 2 TPU-min → 12 TPU-h lifts formal-imo +10.5pp,
  PutnamBench-test +11.5pp;
- $S$ saturates: e.g. search-only plateau at ≈ 43.7% (formal-imo) and
  39.4% (Putnam) is what TTRL then breaks by +15pp.

## 4. IMO 2024 evaluation protocol (numbers)

1. Pre-release freeze: main RL stopped, hyperparams fixed before
   problem release.
2. Expert Lean formalization of the 5 non-geometry problems (P5 was
   too hard to formalize within day 1; the auto-formalization system
   as a side experiment succeeded on P1, P2, P6).
3. Answer-first problems (P1, P2, P5, P6): statements formalized with
   `sorry` placeholders; answer generator = Gemini 1.5 Pro + Python
   tooling, few-shot; $k = 500$ candidate answers per problem →
   correct answer present for all four.
4. Refutation sieve, 10-minute tree search per candidate (low-compute
   mode): disproved 99% of P1 candidates, 98% of P2 candidates, 7% of
   P6 candidates (P6's near-constant value makes candidates sit close
   to the truth: hard to refute, hence deeper TTRL needed).
5. TTRL on the surviving candidates: 2–3 days each → proofs for
   P1 (algebra), P2 (number theory), P6 (algebra).
6. P4 (geometry) → AlphaGeometry 2; P3, P5 (combinatorics) unsolved.
7. Judges (T. Gowers, J. Myers) score full points per solved problem:
   total 28/42 = silver medallist level; gold cutoff = 29.
8. Human reference: P1 413/609 solved by students, P2 156/609,
   P6 5/609.

## 5. Other protocol data

Auto-formalization evaluation A/B: 50 IMO + 50 Putnam statements,
panel ≈ 3 experts (metrics in 05 §5).

TTRL ablations: quantity ↑ → solve rate ↑; generator strength (2.0
Flash S) → harder curriculum → better target solve rate (07 §4).
