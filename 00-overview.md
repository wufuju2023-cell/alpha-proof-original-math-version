# 00 Overview — AlphaProof reconstruction dossier

Source of truth: Hubert et al., "Olympiad-level formal mathematical
reasoning with reinforcement learning", Nature, 2025
(doi:10.1038/s41586-025-09833-y).

This dossier restates every formal object of the AlphaProof pipeline in
mathematical notation, and (re)derives the training/search dynamics
from these objects. Where the paper leaves hyperparameters to
"Supplementary Tables", this dossier fixes explicit values, marked
**S** (= suggested choice), consistent with references [2,18,72].

## 1. Notion summary

- $s$: Lean tactic state. $\Gamma$: hypothesis context; $\Delta$: goals.
- $a$: tactic, text string. $\pi_{\theta}(a \mid s)$: policy.
- $r_t = -1$ per applied tactic. $G_t$: return, $V(s) = \mathbb{E}[G_t \mid s_t = s]$.
- $d(s) = -V(s) \ge 0$: expected remaining steps (longest branch for AND states).
- $N(s,a)$: visit count; $n(s)$: number of action samples at $s$;
  $c(s)$: exploration factor; $\tau$: prior temperature.
- $K$: tactics sampled per expansion. $B$: simulation budget.
- $\gamma$: discount used in $Q$-transform ($\gamma < 1$).
- $T_{\text{steps}}$: tactics in the longest branch of a proof.

## 2. Module map

1. Lean environment — $(s, a, r, G)$ semantics, AND-state split,
   disproof operator, verification. See 01-lean-env.md.
2. Proof network — 3B encoder–decoder; policy + categorical value. See 02-proof-network.md.
3. Tree search — Sampled-AlphaZero PUCT, AND nodes, progressive
   sampling, single-search. See 03-search.md.
4. Pretraining + SFT — 300B-token corpus, span corruption, 300K
   state-tactic pairs. See 04-pretrain-sft.md.
5. Auto-formalization — Gemini fine-tune, STaR refinement,
   80M-curriculum construction. See 05-autoformalize.md.
6. Main RL — matchmaker/actors/learner loop, losses, budgets. See 06-main-rl.md.
7. TTRL — variant generation, evolutionary curriculum, focused RL. See 07-ttrl.md.
8. Benchmarks and measured scaling. See 08-benchmarks.md.
9. Reconstruction plan for an open-source clone. See 09-reconstruction-plan.md.

Deep-dive modules (full math, deriver, alternatives, experiments):

10-transformers.md — attention/encoder/decoder/cross-attn math, RoPE,
   RMSNorm, SwiGLU, Flash-softmax, cost formulas, sampling.
11-tokenizer.md — BPE/minBPE math, vocab design, Lean-aware tokens.
12-rl-math.md — MDP, Bellman contraction proof, MC/TD/GAE, policy
   gradient theorem, why AlphaProof is expert-iteration.
13-mcts-theory.md — bandit abstraction, UCB1 regret theorem, UCT
   consistency, value-bootstrap, ranked cost model, rollout-failure
   math (compounding).
14-puct-and-temperature.md — PUCT = UCB1 + prior, Q-transform
   analysis, c(s) derivation, three temperatures (prior/root/
   generation), Dirichlet root noise, virtual loss.
15-search-alternatives.md — beam/best-first/A*/LATS/whole-proof vs
   MCTS; axis comparison; experiment to decide.
16-llm-posttraining-choices.md — SFT/RLHF-PPO/GRPO/RLOO/DPO exact
   objectives; why AlphaProof is none of them; when to choose which.
17-design-choices.md — ledger of every design knob (why + suggested
   value + experiment id).
18-ttrl-advanced.md — variant algebra, evolutionary expansion,
   difficulty scheduling, stopping rules, compute allocation.
19-open-training-recipe.md — open-source build: env interface, data
   pipeline, SFT/RL/TTRL numbers, throughput formulas, metrics.
20-experiment-suite.md — E1-E15 experimental protocols (y-sweeks,
   metrics, bootstrap CI, decision rules, 3 quick wins list).

## 3. Global constants used in this dossier (all **S**)

- $\gamma = 0.99$ (Q-transform discount; see 03-search.md).
- $K = 64$ tactics sampled at expansion.
- Prior temperature $\tau = 1.0$.
- $c_{\mathrm{init}} = 1.25$, $c_{\mathrm{base}} = 19652$.
- $C = 1.0$, $\alpha = 0.5$ (progressive sampling).
- $c_{\mathrm{pen}} = 1.0$ (unvisited edge penalty, in steps).
- $c_{\mathrm{AND}} = 1.0$ (AND-node exploration multiplier).
- Horizon $T_{\max} = 1024$ tactics per episode (wall-clock also capped).
- Simulation budgets: $B_0 = 500$, growth factor $2$, cap $15{,}000$,
  recent-window $N = 25$, $\mathrm{trust\_count} = 5$.

All constants flagged (S) are ours; anything without flag is from the paper.
