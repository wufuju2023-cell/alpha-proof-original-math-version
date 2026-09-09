# 06 Main RL (AlphaZero-style, with a start-position corpus)

The board-game self-play trick (single universal start state) is
unavailable; instead start positions come from a huge formal problem
dataset. Agent = AlphaZero over a distribution over start positions.

## 1. Start position dataset

$$
\mathcal{D}_{\mathrm{start}} = \mathcal{D}_{\mathrm{AutoForm}}
\cup \mathcal{D}_{\mathrm{human}},
\qquad
|\mathcal{D}_{\mathrm{AutoForm}}| \approx 8\times10^{7},\quad
|\mathcal{D}_{\mathrm{human}}| \approx 3.5\times10^{3}
$$

$\mathcal{D}_{\mathrm{human}}$: miniF2F-curriculum (ref. 24) +
auto-formalization seed set + PutnamBench-train (ref. 21, minus test).

## 2. Matchmaker (orchestration, formula-level)

Per problem $p$: history of last $N$ attempts, outcomes
$\{ \text{proof}, \text{disproof}, \text{timeout} \}$.

**Interestingness** (priority):

$$
I(p) = \mathrm{high}\ \text{iff}\ \big[
n_p = 0 \ \lor\ n_p < \mathrm{TC} \ \lor\
(n_p \ge \mathrm{TC} \land 0 < s_p < 1)
\big]
$$

where $s_p = \frac{\#\text{proofs in last } N}{N}$
("solved in some, but not all, of the last $N$ attempts"), and

$$
I(p) = \mathrm{low} \iff s_p = 0 \ \text{(with } n_p \ge \mathrm{TC}\text{)}
\ \lor\ \text{consecutive proofs} \ge \mathrm{TC}
$$

$s_p = 1$ (mastered) and consistent-failure states → deprioritized;
**disproved statements never retried**.

(S: $\mathrm{TC} = \mathrm{trust\_count} = 5$, window $N = 25$.)

**Adaptive compute budget per attempt**:

$$
B(p) = \min\big(B_{\mathrm{cap}},\ B_0 \cdot m^{\,f_p}\big),
\qquad f_p = \#\mathrm{timeouts}\ (\text{last } N)
$$

(S: $B_0 = 500$ sims, $m = 2$, $B_{\mathrm{cap}} = 15{,}000$.)
Qualitative behavior: easy statements get few sims; oscillating hard
ones get exponentially more; every actor gets a random objective
(prove XOR disprove) and a statement from the prioritized queue.

## 3. Actor experience generation

- tree search (03) guided by current $\theta$, from $s_0(p)$;
- no action commitment: search the full attempt, using the whole
  budget $B(p)$, return `proved` / `disproved` / `timeout`;
- only **verified** outcomes transmitted to the learner; timeouts are
  logged to matchmaker but excluded from training batches.

## 4. Learner

Replay buffer: successful search (state-tactic-second) traces from
proofs **and** disproofs; mix:

$$
\frac{|\mathcal{B}_{\mathrm{SFT}}|}{|\mathcal{B}_{\mathrm{self}}|}
= \frac{1}{9}
$$

(policy batches: 10% Mathlib SFT data, 90% self-generated.)

**Policy loss.** For states along a successful trajectory where tactic
$a$ was applied:

$$
\mathcal{L}_{\mathrm{pol}} =
-\log \pi_\theta(a \mid s)
$$

Only successful attempts contribute (no ghost actions from failed
searches).

**Value loss.** Categorical CE with return target. For a successful
chain $s_0 \to s_1 \to \cdots \to s_T$ (terminal proof), the return
from state $s_t$ is:

$$
G(s_t) = -(T - t)
$$

For a path through AND nodes, use the min over subgoals (01 §2).
Binned target:

$$
\mathcal{L}_{\mathrm{val}} = -\log p_v\big(\mathrm{bin}\big(G(s_t)\big)\mid s_t\big), \quad
\widehat{V}(s) = \mathbb{E}_v\big[G \mid s\big]
$$

(S: $\lambda$-return with $\lambda = 1.0$ i.e. Monte Carlo target; a
small $\lambda$-mix $(0.9)$ is a reasonable alternative.)

**Learning-rate scheme** (S): AdamW, warmup 5K, cosine
$\eta: 3\times10^{-5} \to 3\times10^{-6}$, batch 1024 pairs, max
batch tokens $10^{6}$; $\sim 10^{6}$ update steps total (paper:
"approximately 1 million training steps").

## 5. Why this is (asymptotically) a policy-improvement loop

Search with budget $B$ defines an improved policy $\pi_B$ on reachable
states; the actor's *successful* states sample $\pi_B$'s mode region.
This is expert-iteration: policy chases episodes search could actually
finish, values chase exact observed returns, and failures are pruned
so noisy garbage never corrupts targets. Cross-entropy-to-$a$ with 90/10
mix keeps the old SFT behavior as a reference which prevents collapse
on the sparse state space (regularization, S-choice; the paper's ratio
is fixed at 10%/90%).

## 6. Compute

- Main RL: $\approx 8\times10^{4}$ TPU-days (e.g. 4,000 TPUs × 20 days).
- Per-update cost model:

$$
C_{\mathrm{step}} \approx 6\, N_{\theta} \cdot \mathrm{batchTokens}
$$

  over $10^{6}$ steps with $\approx 10^{6}$ tokens per step gives
  $\approx 6\times10^{21}$ FLOPs, plus parallel actor search cost
  (dominant: tactic execution/search budget).
- Search cost dominates wall clock: $B$ sims × K tactics ×
  $t_{\mathrm{exec}}$.

## 7. Measured dynamics (paper, Fig. 3)

- Training-curve fraction: proved ↑, disproved ↑, undecided ↓ with TPU-d.
- Held-out solve rate at 4,000 sims rises monotonically from the
  0-TPU-day SFT checkpoint;
- efficiency: later checkpoints need far fewer sims per solve rate:
  final agent $\approx 30\%$ of historical IMO at 300 sims (earlier
  checkpoints never reach it at any budget);
- data separation: RL curriculum excludes anything similar to eval
  benchmarks (05 §4).
