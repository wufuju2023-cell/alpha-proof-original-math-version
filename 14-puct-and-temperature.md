# 14 PUCT: derivation, temperature, Dirichlet, virtual loss

## 1. PUCT as UCB1 with a prior

Selection index at state $s$ (paper, Eq. 1):

$$
\mathrm{index}(s,a) = Q(s,a) + c(s)\, \pi^{1/\tau}(a \mid s)\,
\frac{\sqrt{\sum_b N(s,b)}}{N(s,a) + 1}
$$

Recast as UCB1:
- exploration term $\sqrt{\sum_b N(s,b)}/(N(s,a)+1)$ is a smoothed
  version of $\sqrt{2\ln N(s)/N(s,a)}$: with raw counts, the log
  numerator grows much slower than $\sqrt{N(s,a)}$, so the term
  dedicates most sims to the leading arm;
- the prior $\pi(a \mid s)$ multiplies the term: it pre-allocates
  pseudo-visits

$$
\tilde{N}(s,a) = N(s,a) + K_{\mathrm{pseudo}}\,\pi(a \mid s)
$$

and behaves like UCB1 on a distribution with *a prior component*:
arms with high prior get an initial head start (their first $K$
exploration pulls are already "won").

Why UCB-like and not pure greedy: greedy has no exploration
guarantee — proofs live in branches with sometimes low, sometimes
high initial estimates; bandit reasoning picks the correct arm in
$O(\ln n / \Delta^2)$ pulls (13 §2).

## 2. The $Q$-transform (exact)

From 03 §1:

$$
Q(s,a) = \gamma^{\,d(s,a) - 1}, \quad d(s,a) = 1 + d(s'), \quad
\gamma \in (0,1) \;\; (\mathbf{S}: \gamma = 0.99)
$$

Why exponential rather than min-max normalization: with
$d \in [0, D_{\max}]$, $Q \in [\gamma^{D_{\max}}, 1] \subset [0,1]$
as well; the local *value spread* is automatically scaled by $\gamma$
(an $\infty$-step gap maps to $\gamma^{\infty} = 0$ which separates
hopeless subtrees from moderately promising ones), and it is
**scale-invariant w.r.t. the network's typical value errors**: a
constant shift in $\hat{V}$ becomes a constant multiple in $Q$ with
rank preserved, while min-max normalization re-scales relative to
siblings and distorts when only one child is poor.

The $\gamma$ choice controls the trade: smaller $\gamma$
(e.g. $0.9$) aggressively deprioritizes long branches (good for
finding *short* proofs fast), larger $\gamma$ ($0.999$) keeps long
proof candidates alive (good for high-difficulty targets where the
only proof is long). E3-style sweep in 20 (E1).

## 3. Exploration factor $c(s)$

$$
c(s) = c_{\mathrm{init}} + \log \frac{N(s) + c_{\mathrm{base}} + 1}{c_{\mathrm{base}}}
$$

(S: $c_{\mathrm{init}} = 1.25$, $c_{\mathrm{base}} = 19652$, from
AlphaZero defaults.) Interpretation: the log term is
$\log\frac{N + c_{\mathrm{base}} + 1}{c_{\mathrm{base}}}$, i.e. $c$ grows as $\ln$ of visits,
mirroring the UCB1 logarithmic bonus. Growth is extremely slow near
$N = 0$; at $N = 19652$ exactly $c(s) = c_{\mathrm{init}} + \ln 2$.

## 4. Temperature on the prior (three distinct $\tau$ in the system)

- $\tau_{\mathrm{puct}}$: prior power in PUCT. Since
  $\pi^{1/\tau} \propto \exp(z/\tau)$, raising to the power
  $1/\tau$ is exactly softmax-tempering of the *logits*:
  $\tau > 1$ flattens the prior (more exploration), $\tau < 1$
  sharpens (more trust in the net). The main reason it exists: the
  net's prior is often overconfident on states it has seen many
  times (SFT & RL bias towards favorite tactics); the power acts as
  a calibrated temperature.
- $\tau_{\mathrm{gen}}$: expansion sampling: temperature used when
  drawing $K$ tactic strings from the decoder. Affects the *diversity
  of the candidate set* and the chance of finding a good-but-low-prior
  action: with small $\tau$, $K$ draws cluster on the top mode.
  (S: $\tau_{\mathrm{gen}} = 0.8$, top-p 0.95; see 20 E4).
- $\tau_{\mathrm{root}}$: optional root-only strategy: for the very
  first simulation epochs of an attempt, sample the root actions
  from a higher-temperature prior to diversify the opening.
  (S: optional, not in the paper; ablations in E4.)

**Restriction (analysis for tuned recommendation).** For the
prior power version: the induced distribution
$\hat\pi_\tau(a) \propto \pi(a)^{1/\tau}$ has entropy $H(\hat\pi_\tau)$
monotone in $\tau$; recommendation: start $\tau_{\mathrm{puct}} = 1.0$
and try $\{0.5, 2.0\}$ in E4.

## 5. Dirichlet noise at root (S, not in paper)

AlphaZero injects Dirichlet noise at root on the prior
(both for Go and chess, with smaller α and e.g. ε=0.25). Suggested:

$$
\hat{\pi}(a \mid s_0) = (1 - \varepsilon)\,\pi(a \mid s_0) +
\varepsilon\, \mathrm{Dir}(\alpha)\, \text{with } \alpha = 0.3,\ \varepsilon = 0.25
$$

Purpose: for an unexplored problem the net's root prior can be
highly concentrated; noise ensures diversified roots. No restarts
(data-amortized) → cheap. Use of a different approach for
alternative positions: progressive sampling (03 §6) does the same at
any node — noise is a cheap low-B substitute. Default: pick
progressive sampling (paper) + ε=0.2 root noise (S).

## 6. Virtual loss for parallel sims (S)

To avoid duplicate work when several sims run concurrently through
the same node, reservation: decrement each visited node's
$N$ by $v_{\mathrm{virt}} = 1$, implying an artificially bad return
of $-1$ stored in $Q$. Condition: tasks launched simultaneously
are typically in different parts of the tree, so $v$ at the frontier
is rare.

## 7. AND-node multipliers in the same analysis

AND node selection:

$$
\mathrm{index}_{\mathrm{AND}} = (1-Q(s_{\mathrm{AND}}, a)) + c_{\mathrm{AND}}\, c(s)\, \pi^{\mathrm{unif}}(a) \frac{\sqrt{\sum_b N}}{\,N(a)+1}
$$

with $\pi^{\mathrm{unif}} = 1/k$. Since $Q \in (0,1]$ (1 is best),
$1-Q$ is the "difficulty", and the bandit at an AND node optimizes
(1) the gap size: always deprioritize solved subgoal; (2) the delay:
the exploration factor is multiplied by $c_{\mathrm{AND}}$
(S: 1.0; sweep 0.5–2 in E3b). The min-max of the backprop (min
$V$, max $d$) keeps the same semantics as the UCT leaf backup.
