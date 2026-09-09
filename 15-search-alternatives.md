# 15 Alternatives to MCTS and how to choose

## 1. Candidate algorithms

- **Greedy depth rollout** (autoregressive, no search): sample a full
  tactic sequence. Cost: $O(d)$ per attempt; fails by compounding
  (13 §6).
- **Beam search**: at each depth keep top-$b$ states by value.
  Cost: $O(d\,b\,K)$; but no *lookahead reallocation*: wrong
  early choices irreversibly cut off the beam.
- **Best-first (frontier) search**: like A*: pop the state
  minimizing $f = g + h$, where $g$ = accumulated steps, $h$ = value
  guess of remaining; expand its $K$ children; never return. Cost
  regulated by frontier. Optimal (w/ admissible $h$) but value-net $h$
  is *not* admissible → the search is heuristic, can chase a long
  dead-end. Memory = frontier, usually a few thousand entries.
- **Iterative deepening / IDA**†: attempt depth-limited DFS with
  growing depth ($N$ revisions) — avoids frontier memory but
  re-expands everything ($b^d$ effective; death for huge $b$).
- **MCTS (this paper's choice)**: see 13.
- **LATS / ToT**: LLM-generation-with-reflection; here the
  environment is deterministic and one-player, so "reflection" is
  just another value estimate (an extra network call, no
  information gain).
- **Program-guided search (HTPS ref 17, DeepSeek-Prover-V2 ref 26)**:
  same MCTS family; the difference is variant-level structure
  (subgoal decomposition, tree-of-subgoals) vs AlphaProof's
  single-tree on the AND/OR-expanded tactic state.
- **Whole-proof generation + repair (Baldur ref 45)**: one generation
  $+$ error-fix loops; error compounding, no per-state adaptation.

## 2. Decision axes (critical comparison)

1. any-time property and budget adaptivity: MCTS yes (bandits
   re-allocate), beam does not (fixed $b$), best-first also yes.
2. robustness to value noise: bandits smooth estimates by sim
   allocation; frontier methods always act on one-shot estimates —
   a single noisy "best" state can lock in a dead end.
3. memory: MCTS ~ states visited ($\approx$ 100k at 10k sims per
   attempt); frontier similar; beam small.
4. environment cost (tactic execution dominates): with
   $t_{\mathrm{exec}} \gg$ inference, minimizing *K per sim* matters:
   MCTS re-uses subtree evaluations; frontier re-evaluates on the
   fly (no pruning of dead children: each child executed once
   anyway).
5. applicability: MCTS gets its strength when the state space is
   enormous yet values are informative. Our setting: yes (branching
   $b \approx$ 1e3-1e5, depth $d \approx$ 1e1-1e3).

## 3. The "why not other search" answers

- Why not BFS/DFS: complexity $b^d$; the only acceptable part is
  the systematic exploration; no estimator to prune.
- Why not A*: no admissible heuristic; the value net is a learned
  guess; admissibility would require a lower bound on steps-minus-1
  (available? of course: proofs ≥ 1 step; but it is uselessly weak:
  $h \ge 0$ everywhere: A* ≈ unweighted BFS). With a *consistent*
  weak $h$ you get optimal solutions; unweighted does not find
  proofs faster.
- Why not pure sampling: math in 13 §6 ($(1-p)^K$ argument). The
  magic of MCTS is not that it samples more: it is that its *value
  estimate* concentrates later sims on the promising left branch
  (Fig. 1c) while still keeping coverage.
- Why not R1-style long-horizon generation (GRPO on proofs): the
  reward is binary and sparse, only terminal; per-step credit
  assignment comes from the value head differently (16), and more
  importantly a wrong-segment rollout wastes its entire budget; a
  search relaxes at every step. The empirical fall-back to "just
  sample many times" is MCTS with K=1.
- Why not use an external prover (SMT/Z3/hammer): 1P3 styles; they
  cover a small fraction of the needed math (e.g. `linarith` is in
  the tactic set and *is* used by search).

## 4. Experiment to choose (see 20, E9)

Design: fixed network checkpoints (SFT-only and post-RL-1M),
same budget (sims, not wall-clock: set both equal since env costing
dominates), same beta-value net. 5 seeds × 3 budgets. The
algorithms: PUCT-MCTS (AlphaProof), greedy best-first frontier
(with $g+h$), beam $b \in \{16, 64\}$, rollout-only (K=1 MCTS
degenerates into pure sampling), and (optionally) HTPS-style
subgoal-tree. Metrics: solve rate vs budget; sims-per-solve; final
solve rate at budget-saturation; wall-clock variance.

Hypothesis: when the value net is good (post-RL): frontier/beam
compete closely; when it is noisy (SFT-only): PUCT wins by a wide
margin (bandit robustness). If confirmed: MCTS is justified; if the
value net is excellent, choosing a much simpler beam is a valid
edge (and the user could simplify).
