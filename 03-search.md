# 03 Tree search (Sampled-AlphaZero for proofs)

Nodes = normalized Lean states; edges = validated tactics.
Notation: $V(s,a)$ aggregated search value of taking $a$ at $s$;
$N(s,a)$ visits; $n(s)$ = tactic draws so far at $s$; $d(s)=-V(s)$.

## 1. Q-transform (key formula)

$$
Q(s,a) = \gamma^{-V(s,a)-1}, \qquad V(s,a) = -d(s,a)
$$

so with $d(s,a) = d(s')$ (steps remaining after $a$):

$$
Q(s,a) = \gamma^{\,d(s,a)-1}, \qquad d(s,a) = 1 + d(s')
$$

Counterintuitive Q convention resolved: $\gamma \in (0,1)$, hence
$Q \in (0,1]$, monotone *increasing* in value (decreasing in steps).
This replaces earlier AlphaZero-style $\bar{V}$ min-max normalization.
(S: $\gamma = 0.99$; with $d$ up to ~300 steps, $Q$ ranges
$0.05 \dots 1$, a useful spread without overflow.)

Unvisited edges (no backprop data yet):

$$
d(s,a) = \widehat{d}_{\mathrm{net}}(s) + c_{\mathrm{pen}}, \quad
c_{\mathrm{pen}} = 1.0 \ \ (\mathbf{S})
$$

(paper: "network prediction for the parent node minus a fixed
penalty" — the penalty punishes taking an unexplored step.)

## 2. Selection (PUCT)

At OR node $s$:

$$
a^{\ast} = \operatorname*{argmax}_{a} \Big[ Q(s,a) +
c(s)\, \pi^{1/\tau}(a \mid s)\,
\frac{\sqrt{\sum_b N(s,b)}}{N(s,a) + 1} \Big]
$$

exploration factor:

$$
c(s) = c_{\mathrm{init}} +
\log\Big(\frac{N(s) + c_{\mathrm{base}} + 1}{c_{\mathrm{base}}}\Big)
$$

(S: $c_{\mathrm{init}} = 1.25$, $c_{\mathrm{base}} = 19652$,
$\tau = 1.0$.)

Rationale: $Q$ scores progress (shorter is better); the PUCT bonus
with $c(s)$ decays as $N(s)$ grows, so exploration contracts with
visits, in the standard AlphaZero way.

## 3. Expansion (at leaf $s_L$)

- Draw $K$ tactics $\sim \pi(\cdot \mid s_L)$.
- Validate each in Lean (01 §1); discard invalid.
- Apply valid tactics; produce child states.
- Merge children that are the same Lean state, up to renaming and
  reordering of syntactically identical hypotheses, keeping the one
  minimizing

$$
\mathrm{cost}(a) = \mu \cdot |a|_{\mathrm{tok}} + t_{\mathrm{exec}}(a),
\quad \mu \ \ (\mathbf{S}: \, 1.0)
$$

(paper: "cost function linearly depending on string length and
execution time").
- Leaf value: $V(s_L) = \widehat{V}_{\mathrm{net}}(s_L)$ (network
  estimate, not search aggregate).

## 4. Backpropagation

Along the traversed path: for each (s,a):

$$
N(s,a) \leftarrow N(s,a) + 1, \qquad
d(s,a) \leftarrow \frac{(N-1)\,d(s,a) + d_{\mathrm{child}}}{N}
$$

For OR nodes the state aggregate is the min over children (best
path):

$$
d(s) = \min_a d(s,a)
$$

## 5. AND nodes (multi-goal states)

From 01 §2: goal split into $k$ independent subgoals
$s^{(1)}, \ldots, s^{(k)}$. Actions from an AND node = choose which
subgoal to work on; only **unproven** subgoals are selectable.

Selection:

$$
j^{\ast} = \operatorname*{argmax}_{j} \Big[ \big(1 - Q(s_{\mathrm{AND}}, j)\big)
+ c_{\mathrm{AND}}\, c(s_{\mathrm{AND}})\, \pi(j \mid s_{\mathrm{AND}})\,
\frac{\sqrt{\sum_b N(s_{\mathrm{AND}}, b)}}{N(s_{\mathrm{AND}}, j) + 1} \Big]
$$

- $\pi(j) = 1/k$ uniform over subgoals (S).
- $1 - Q$ replaces $Q$: with $Q \in (0,1]$ the hardest subgoal
  (smallest $Q$, largest $d$) gets the largest selection score, so the
  search attacks the bottleneck. (S: $c_{\mathrm{AND}} = 1.0$.)
- Backprop through an AND node: pass the **min** child value upward
  (equivalently max $d$), matching the min-return definition:

$$
d(s_{\mathrm{AND}}) = \max_j d\big(s^{(j)}\big)
$$

## 6. Progressive sampling (novelty vs standard MCTS)

If on the simulation path we hit a node with

$$
n(s) \le C\, N(s)^{\alpha}
$$

draw $K$ additional tactics from $\pi(\cdot \mid s)$ and add them as
new edges. (S: $C = 1.0$, $\alpha = 0.5$; paper attributes to
Coulom's progressive unfolding.) Interpretation: near-frequent nodes
receive fresh diversity; the additional samples are themselves
validated and cost-ranked as in §3. Early $N$ → cheap diversity;
large $N$ → deepen.

## 7. Single search per attempt (no commits)

For one problem: one tree, expanded until `proved` /
`disproved` / budget $B$. No action commitment, no restarts. Because
the environment is deterministic and perfect, the tree globally
allocates compute over all discovered paths. (Unlike games, no
opponent/unmodeled state.)

## 8. Pseudocode

```
def search(s0, B, model, env):
    tree = new_tree(s0)
    while count_sims < B:
        s = select_root_to_leaf(tree)      # §2, §5
        if leaf not expanded:
            sample K if n(s) <= C*N(s)^α or first visit   # §3, §6
            add edges (valid, dedup, cost-min)
            V_leaf = model.value(s_leaf)
        backprop(V_leaf, path)             # §4
        if env.proved(s0):  return (proved, τ)
        if env.disproved(s0): return (disproved, τ¬)
    return timeout
```

Cost per simulation:

$$
O\big(K\, t_{\mathrm{exec}} + K\, L\, d\big)
$$

with batched network calls (K in-flight tactics per sim, L state
tokens, d model dim).
