# 13 MCTS: why it is used (bandits, UCT, theorems)

## 1. MCTS as approximate dynamic programming

Monte-Carlo tree search is a randomized, asynchronous version of
**policy evaluation + policy improvement** (backwards induction done
forwardly with rollouts). Each simulation is one Monte-Carlo sample
of a subtree: it (a) evaluates: averages a return estimate into
$Q(s,a)$ along the traversed path; (b) improves: the selection policy
(barring exploration) reads $\operatorname*{argmax}_a Q(s,a)$, one
component of a greedy policy improvement step.

Formally, MCTS implements the Bellman operator via Monte-Carlo:
$Q_{\mathrm{mcts}}(s,a) \approx$ sample-average of
$\big(r + \gamma Q_{\mathrm{mcts}}(s', a')\big)$; with tree search
values computed the same way the fixed point of the operator is the
Bellman-optimal $Q^{\ast}$, because the environment is deterministic
and the tree records exact transitions. The value network at the
leaves is a learned `V` substitute for an a-priori-unknown
bootstrapped term.

## 2. Bandit abstraction (per state a multi-armed bandit)

At node $s$ with candidate actions $A(s)$, each sims pulls one arm.
**UCB1** plays arm $i$ with index

$$
I_i(t) = \bar{x}_i + \sqrt{\frac{2\ln n}{n_i}}, \qquad
n_i = \#\text{pulls of } i, \; n = \sum_i n_i
$$

The key property — Hoeffding tail bound:
$\Pr\big[\mu_i > \bar{x}_i + \sqrt{2\ln n / n_i}\big] \le n^{-4}$
(exponential exp of the residual). So with probability
tightly 1: the true best arm's index lies above its sample mean and
each gap is eventually punished.

**Theorem (UCB1 regret bound, Auer et al. 2002).** Against any
reward distribution in [0,1], for suboptimal arm $i$ with gap
$\Delta_i = \mu^\ast - \mu_i > 0$:

$$
\mathbb{E}\big[ T_i(n) \big] \le \frac{8 \ln n}{\Delta_i^2} + 1 + \frac{\pi^2}{3}
$$

Consequence: after $n$ pulls the expected *regret* is
$O(K \ln n / \min_i \Delta_i)$ — i.e. what UCB pays to find the best
arm is **logarithmic** in $n$ rather than linear; the "sample
complexity" is $O(\ln n / \Delta^2)$, so the algorithm spends
essentially all its pulls on the best arm.

## 3. UCT (Kocsis & Szepesvári 2006)

UCT = run bandit UCB at every node of the tree with the payoff being
the rollout return of the child subtree. Facts (the standard
statements):

- UCT allocations are consistent: with probability 1, each node
  explores each child, and as $KT \to \infty$, the root's
  estimated value converges to $V^{\ast}$ (in probability; the root
  action distribution converges to concentrate on optimal actions);
- the graph is a tree of bandits and the per-level bandit bound of
  Section 2 applies level by level: the number of sims a *suboptimal*
  subtree gets (over-all bad pulls) is
  $O\big(D \ln KT / \Delta^2\big)$ where $\Delta$ is the gap of the
  child value at that node and $D$ the horizon (all rewards in
  [0,1]).

They key transformation: the *computational complexity* of deciding
the root move goes from $b^D$ (exhaustive) to
$\mathrm{poly}(b, D / \Delta^2)$ — exponential in $D$ removes,
replaced by binom — plus a variance term from Monte-Carlo rollouts.

## 4. The variance-bootstrapping step (why a value network)

Rollouts with random continuations have huge variance: the return of
a random walk in tactic space is essentially $-T_{\max}$. Putting a
learned $V$ at leaves replaces stochastic rollout *tails* by a
deterministic one-step bootstrap (this is MuZero-style). Formally:
value-of-leaf = $\widehat{V}_{\mathrm{net}}(s_L)$ with noise
$\varepsilon_L$; when $\varepsilon_L$ is small (good value net), the
variance term vanishes and the bandit analysis holds with
deterministic subtree returns. AlphaProof is exactly this:
evaluate leaf with the network's categorical value head; no random
rollouts.

## 5. Cost model derived from ordering (why values + priors matter)

Assumption: the search discovers a proof of length $d$ if it
visits the `good` children (those on a witness path) in node order.
Define the *rank* $r_j$ of the good child among the explored
candidates at level $j$: the expansion cost of finding the whole
proof is

$$
\sum_{j \le d} r_j \approx d\, \mathbb{E}[r]
$$

- blind search: $\mathbb{E}[r] = (b+1)/2 \Rightarrow$ cost
  $\approx d\, b/2$ but with *depth* overflow: overstates: each good
  child must be visited only $O(1)$ times, but $b$ can be huge and
  we must also keep all ancestors alive. Real costs grow with
  $b^{\text{depth}}$ only when no values are used;
- with an accurate value function, the good child's rank
  concentrates at 1: $\mathbb{E}[r] \approx 1$, search cost
  $\approx d$ visiting calls, i.e. the value discussion made
  explicit: this is the reason "30% at 300 sims" in the paper works.
  Value noise inflates $\mathbb{E}[r]$; PUCT's bandit allocation
  adapts, paying the UCB1 cost of Section 2 per child instead of
  switching by estimate.

## 6. Why not simple depth-first / greedy rollout

The compounding-failure count: at every state, the "good set" of
actions (on any proof) has probability mass $p$ under the current
policy. One random rollout succeeds with probability at most
$p^d \approx e^{-d(1-p)}$ — catastrophically small at depth 50 with
$p = 0.1$ ($10^{-50}$). Sampling at the tree: $K$ per node,
so the per-node failure is $(1-p)^K$ and the overall success
probability along the tree is about

$$
\Pr[\text{found}] \approx \Big[ 1 - (1-p)^{K} \Big]^{d}
$$

with $p = 0.1$, $K = 64$, $d = 50$: $(1 - 0.9^{64}) \approx 1$ —
this is the quantitative case for tree search. MCTS additionally
concentrates the budget on the path regions that the value prior
identifies as promising (Section 5).

## 7. When MCTS is the wrong tool (limits)

- If value function is noise: values randomly ranked: MCTS collapses
  to near-random scan, best-first with prioritization is equivalent
  and cheaper (see 15).
- If $t_{\mathrm{exec}}$ per tactic is high relative to compute:
  search is dominated by env calls, so a narrower, deeper beam may
  dominate.
- If per-attempt sims are so small that the tree has no rows:
  pure sampling ($K$-best) is competitive.
