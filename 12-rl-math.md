# 12 Reinforcement learning mathematics (Bellman, MC, TD, PG)

## 1. MDP

$(\mathcal{S}, \mathcal{A}, P, R, \gamma)$, episodic, finite horizon
$T_{\max}$ (S: 1024). Policy $\pi(a \mid s)$.

Approximate each problem statement by the MDP of the Lean
environment (01): deterministic transitions
$P(s' \mid s, a) \in \{0,1\}$ (tactic execution is deterministic;
only the network sampling is stochastic).

Reward: $-1$ per tactic. Terminal success: goal-free state.
We define returns

$$
G_t = \sum_{k \ge t} r_k, \quad
V^{\pi}(s) = \mathbb{E}_{\pi}\big[ G_t \mid s_t = s \big]
$$

(no discount in the return computation itself; the discount $\gamma$
enters only via the $Q$-transform in search, 03/14.)

## 2. Bellman equations (for the exact objects)

Bellman expectation operator (policy evaluation):

$$
(T_{\pi} V)(s) = \sum_a \pi(a \mid s)\, \sum_{s'}
P(s' \mid s,a)\big( r(s,a,s') + \gamma V(s') \big)
$$

Bellman optimality operator (control):

$$
(T^{\ast} V)(s) = \max_a \sum_{s'} P(s' \mid s,a)\big( r + \gamma V(s') \big)
$$

**Theorem (contraction).** $T_{\pi}$ and $T^{\ast}$ are
$\gamma$-contractions in $\|\cdot\|_{\infty}$: for all $V, U$,

$$
\| T^\ast V - T^\ast U \|_\infty \le \gamma \| V - U \|_\infty
$$

Proof: for a fixed $a$ the expectation is a convex combination;
$\big| \max_a X_a - \max_a Y_a | \le \max_a |X_a - Y_a|$; recursively
obtain $\gamma$ per horizon step. Banach's fixed point theorem then
gives the unique solution (value function $V^{\ast}$) and geometric
convergence of value iteration:

$$
\| V_k - V^{\ast} \|_\infty \le \gamma^{k} \| V_0 - V^{\ast} \|_\infty .
$$

## 3. The special structure of proof search

Deterministic single-player, AND/OR decomposition. On a single-goal
DAG the Bellman optimality equation reads:

$$
V^{\ast}(s) = \max_a \big\{ -1 + V^{\ast}(s') \big\} \quad
\text{with} \; V^{\ast}(\text{goal}) = 0
$$

hence $V^{\ast}(s) = -(\text{min steps to a proof from } s)$, and
value iteration converges in $T_{\max}$ steps. The catch: the state
space is astronomically large (countable but ~$10^{100+}$), so we
approximate $V$ with a neural network and only evaluate on states
reached by a guided stream of Monte-Carlo rollouts; the same
Bellmanity structures the AND nodes (min over subgoals):

$$
V^{\ast}(s_{\mathrm{AND}}) = \min_j V^{\ast}\big(s^{(j)}\big)
$$

**This is why tree search + function approximation exists**:
exact DP is infeasible ($b^d$ states; even visiting $b^d/2$ is
hopeless); Monte-Carlo + bootstrap trade variance for bias.

## 4. Monte-Carlo vs TD

MC estimate of $V^\pi(s)$: mean of sampled returns (unbiased, high
variance, no bootstrapping). TD(0):

$$
\delta_t = r_t + \gamma V_{\theta}(s_{t+1}) - V_{\theta}(s_t), \qquad
V_{\theta}(s_t) \leftarrow V_{\theta}(s_t) + \alpha \delta_t
$$

biased (bootstrap uses $\hat{V}$) but lower variance. $\lambda$-return
combines them:

$$
G^{\lambda}_t = \sum_{k=0}^{\infty} (\gamma\lambda)^{k} \big( r_{t+k} + \gamma V(s_{t+k+1}) \big) \cdot \text{...}
$$

more precisely, GAE for advantages:

$$
\delta_t := r_t + \gamma V(s_{t+1}) - V(s_t), \qquad
A^{\mathrm{GAE}(\gamma, \lambda)}_t = \sum_{l \ge 0} (\gamma \lambda)^{l} \delta_{t+l}
$$

$\lambda = 0$: TD(0) advantage; $\lambda = 1$: MC advantage
(AlphaProof's value targets are equivalent to Monte-Carlo returns
from search — Section 14 & 16 discussions).

## 5. Policy gradient theorem

Objective: $J(\theta) = \mathbb{E}[\sum_t r_t]$ with
$\tau \sim \pi_\theta$. Score-function trick:
$\nabla_\theta \log \pi_\theta(a \mid s)$:

$$
\nabla_{\theta} J(\theta) =
\mathbb{E}_{\tau} \left[
\sum_t \nabla_\theta \log \pi_\theta(a_t \mid s_t)\, G_t
\right]
$$

Derivation chain: 
$\nabla J = \sum_\tau \nabla \pi(\tau) R(\tau)$
$\displaystyle = \sum_\tau \pi(\tau) R(\tau) \nabla \log \pi(\tau)$
with $\log \pi(\tau) = \sum_t \log \pi(a_t \mid s_t) + \log P(s_{t+1}\mid s_t, a_t)$
and the environment terms vanish (they do not depend on $\theta$).
Baseline $b(s_t)$ (function of $s_t$ only) is unbiased because

$$
\sum_a \pi_\theta(a \mid s)\, \nabla_\theta \log \pi_\theta(a \mid s) = 0
\ \Rightarrow\ \mathbb{E}\big[ \nabla \log \pi(a\mid s) \, b(s) \big] = 0
$$

REINFORCE-with-baseline:

$$
\nabla J = \mathbb{E}\Big[ \sum_t \nabla \log \pi_\theta(a_t \mid s_t)\,
\big( G_t - b(s_t) \big) \Big]
$$

## 6. Actor-critic

Replace $b$ by $V_{\phi}(s)$ (critic), then we get the GAE advantage
estimates (Section 4). PPO's clipped objective used over these
advantages:

$$
L^{\mathrm{CLIP}}(\theta) = \mathbb{E}_{t}\Big[
\min\!\big( \rho_t A_t,\ \operatorname{clip}(\rho_t, 1-\varepsilon,
1+\varepsilon)\, A_t \big) \Big], \quad \rho_t = \frac{\pi_\theta(a_t \mid s_t)}{\pi_{\theta_{\mathrm{old}}}(a_t \mid s_t)}
$$

## 7. AlphaProof's construction in this taxonomy

- Rewards: verifier-grounded ($r_t = -1$ per tactic; terminal
  reward implicit in reaching the proof). No learned reward model.
- Value: categorical head over $-d$ (steps remaining). Trained on
  exact search MC returns of successful/deduped search paths
  (decentralized), 10% SFT mix.
- Policy: cross-entropy to the action the search "chose" (the
  actions along successful proof/disproof traces), i.e. the gradient

$$
\nabla_\theta \mathcal{L} = -\nabla_\theta \log \pi_\theta(a^{\ast} \mid s),
$$

  i.e. **expert iteration**: the improved policy (search) is treated
  as the supervisor. No clipping, no group-relative advantages: the
  trust region is implicit, because search always follows the current
  policy, so on-policy data (see 16 §5 for the GRPO contrast).

- All of this is exactly AlphaZero's structure with states = tactic
  states, actions = tactic strings, rewards = $-1$, opponent = none,
  and a *distribution over start states* (the curriculum).

## 8. Why the dynamics of the main RL is sound, at least monotonically

The search output is a verified success or nothing. Consider the
per-state action distribution $\sigma = \mathrm{search}(s; \theta)$
and define $\pi_{\mathrm{imp}}(a \mid s) \propto \mathbf{1}\{\text{on a success path}\}$.
Training CE to $\pi_{\mathrm{imp}}$ is policy
improvement whenever search finds at least one success; every
successful state provides a supervised (state → good-action) signal.
Failures contribute nothing (they are pruned). This is the standard
"bootstrapping proof data" scheme of refs. 13/24 (and is why the
paper claims: later agents solve more with far fewer sims).
