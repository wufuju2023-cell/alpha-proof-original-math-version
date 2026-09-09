# 16 LLM post-training: SFT, RLHF/PPO, GRPO, expert iteration — exact math and comparison

## 1. SFT (supervised fine-tuning)

Dataset $\{(x_i, y_i)\}$ (prompt–response), objective CE on response
tokens:

$$
\mathcal{L}_{\mathrm{SFT}} = -\mathbb{E}_i\Big[ \sum_{t}
\log \pi_\theta(y_{i,t} \mid x_i, y_{i,<t}) \Big]
$$

## 2. RLHF (RL with a reward model)

Steps: (1) RM $r_\phi(x, y)$ trained on preference pairs; (2) RL over
$\pi_\theta$ maximizing $J$ with KL penalty to $\pi_{\mathrm{ref}}$:

$$
J(\theta) = \mathbb{E}_{y \sim \pi_\theta(\cdot \mid x)}
\Big[ r_\phi(x, y) - \beta \log \frac{\pi_\theta(y \mid x)}{\pi_{\mathrm{ref}}(y \mid x)} \Big]
$$

PPO objective (per step $t$ of the rollout, with GAE advantages $A_t$
over token rewards $r_t = r_\phi^{(t)} - \beta \log\rho_t$):

$$
L^{\mathrm{CLIP}} = \mathbb{E}\min\big[ \rho_t A_t,\
\operatorname{clip}(\rho_t, 1-\varepsilon, 1+\varepsilon)\, A_t \big]
$$

Value loss $\mathcal{L}_v = \big(V_\psi(s_t) - G_t^{\mathrm{GAE}}\big)^2$,
entropy bonus $\beta_H H(\pi)$.

## 3. RLVR: RL from *verifier* feedback (rule/checker based)

Reward $r(x, y) = \mathbf{1}\{\mathrm{Verifier}(y) \text{ passes}\}$
(code compiler, unit tests, Lean kernel, ...). Used e.g. R1-style:
GRPO (Zhang et al., 2024/2025):

For each prompt $x$: sample group $y_1, \ldots, y_G$; per-sample
reward $R_i$; group-normalized advantages (per token, same across
tokens of a response):

$$
A_{t}^{(i)} = \frac{R_i - \mu_G}{\sigma_G + \epsilon}, \qquad
\mu_G = \frac{1}{G}\sum_{j} R_j, \qquad
\sigma_G^2 = \frac{1}{G}\sum_j (R_j - \mu_G)^2
$$

loss:

$$
L^{\mathrm{GRPO}} = -\mathbb{E}\Big[\frac{1}{G}\sum_i \frac{1}{T_i}
\sum_t \frac{\pi_\theta(y_{i,t} \mid \cdot)}{\pi_{\theta_{\mathrm{old}}}}\,
A_t^{(i)} - \mathcal{O}\Big]
$$

(with clip and optional KL-in-reward). RLOO variant uses
leave-one-out baseline:

$$
A_i = R_i - \frac{1}{G-1}\sum_{j \ne i} R_j
$$

DPO: offline, closed form, no values:

$$
L^{\mathrm{DPO}} = -\mathbb{E}_{(x,y_w,y_l)}
\log \sigma\Big[ \beta \big( \log\frac{\pi_\theta(y_w \mid x)}{\pi_{\mathrm{ref}}(y_w \mid x)}
- \log\frac{\pi_\theta(y_l \mid x)}{\pi_{\mathrm{ref}}(y_l \mid x)} \big)\Big]
$$

## 4. Where AlphaProof sits (this is the key point)

AlphaProof is **none of the above**: it is *search-augmented expert
iteration* (search-as-trainer). There is no reward model (the Lean
kernel is the verifier), no group-relative advantages (no sampled
group at all — the environment is deterministic), and no
on-policy-to-canvas clipping. Its update formulas (see 06 §4):

- policy: $\nabla = -\nabla \log \pi_\theta(a^{\ast}\mid s)$ where
  $a^{\ast}$ = the action the search actually executed on a
  *successful* branch. This is implicit policy improvement:
  the search policy is *computationally superior*, we imitate it:
  per-state this is equivalent to running the search with the same
  net but *more* sims. (Standard "bootstrapping" / expert-iteration
  theories.)
- value: returns used as *labels* state-by-state:
  $\hat{V}(s_t) = G_t = -\text{steps remaining (longest branch)}.$
  They are *exact* per state along a success, so no advantage
  bias/noise from rollouts. (This is better than MC with a single
  terminal reward because it is a per-state, deterministic target.)

Terminology — the same family:

- "RLHF" = learned reward + PPO + critic;
- "RLVR/GRPO" = binary/sparse verifier reward + group MC advantages
  + no critic, sample $G$ variants of each prompt;
- "expert iteration" = search produces the improved data (this
  paper, also refs. 13-24);
- "AlphaZero/MuZero" = value-backed MCTS + categorical value + a
  **state-value function with per-state advantage semantics** (this
  paper also, but with a distribution over initial states instead of
  an empty board). AlphaProof is literally "AlphaZero for a
  single-player, start-position-distribution MDP with text
  actions".

## 5. Concrete differences you care about

List format:

- Advantage estimator: PPO: GAE + critic; GRPO: group-wise z-scores;
  AlphaProof: search-derived state-value targets (+ implicit 0/1
  advantage at branch-success).
- Value: PPO: scalar critic; GRPO: none; AlphaProof: categorical
  value head over steps-remaining (in the $Q$-transform).
- Trust region: PPO: clipping; GRPO: clipping; AlphaProof: search is
  a *same-policy* sample, and training data is on-policy (search
  always uses the current net; no stale data → no distribution
  shift/clipping needed; there is a 10% SFT mix for forgetting
  prevention).
- Reward sparsity: all share verifier-backed terminal reward;
  exhaustive search handles the penalty better.
- Sample efficiency: GRPO sample-out comes from 8-64 i.i.d.
  *independent* responses; search corresponds to 10³-10⁴ correlated
  but *tree-structured* samples — much better local search around
  good branches.
- Failure handling: single failure → discard: same, but GRPO loses a
  full prompt episode; search keeps the subtree, i.e., partial
  evidence may be re-used if a branch is close to success.

## 6. Other credible choices and tuning protocol

Candidates: (a) PPO with the same verifier and an alinearized
log-π KL-term; (b) GRPO; plus (c) soft expert iteration with
$\pi_{\mathrm{target}} = (1-\alpha)\,\pi_{\mathrm{search}} + \alpha\,\pi_{\mathrm{old}}$
(recommended: α starts at 0.5 and decays to 0); (d) REINFORCE with
GAE on a scalar critic; (e) DPO on search-vs-prior contrastive pairs
($y_{\mathrm{good}}$ drawn from search,
$y_{\mathrm{bad}}$ from current policy): cheap and works when
inference is compute-limited.

How to choose (same-criterion evaluation protocol; see 20 E9):

- Equal compute (fixed GPU-days, fixed sims), 3 seeds;
- metrics: solve rate, sims/solve, mean solved proof length,
  forward-looking growth (rate at step k+Δ vs k) — the learning
  curve;
- never evaluate single-checkpoint numbers; compare *curves*;
  when two curves cross, report both.

Hypothesis ranking (empirical in the literature): expert iteration ≈
AlphaProof's = best for verifier-heavy, expensive-rollout settings;
GRPO when $G$ is small/large, reward cheap, and no search wanted;
PPO when a learned RM or typed-value function is available and the
task is like RLHF.
