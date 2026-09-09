# 01 The Lean RL environment (formal definition)

Lean 4/CIC: a statement is a type $\Gamma \vdash P$, a proof is a term
of that type. We cast tactic-mode proving as a Markov decision process.

## 1. States, actions, transitions

State (normalized):

$$
s = (\Gamma, \Delta), \quad \Gamma = [x_1 : A_1, \ldots, x_m : A_m], \quad
\Delta = [g_1, \ldots, g_N]
$$

$\Gamma$: sorted, $\alpha$-renamed hypothesis list; $\Delta$: open goals.
The observation fed to the network is the pretty-printed string
$\mathrm{pp}(s)$, truncated at $L_{\max}$ tokens (S: 4096).

Action $a$: one Lean tactic string. Transition:

$$
\mathrm{exec}(s, a) = s' \quad \text{or} \quad \mathrm{invalid}
$$

Validity conditions for $a$ at $s$:

1. $a$ executes without error;
2. the generated proof term contains no `sorry`;
3. the closed term type-checks: goals of $\Delta$ not addressed by $a$
   are closed via the private axiom

$$
\mathrm{internalSorry} : \forall\,\beta : \mathrm{Prop},\, \beta
$$

so the kernel only has to verify the branch $a$ actually touched.
Soundness is preserved: the final closed proof is still kernel-checked.

## 2. Rewards, returns, value function

Ever applied tactic costs:

$$
r_t = -1, \qquad
G_t = \sum_{k \ge t} r_k \ \text{until termination}
$$

Single-goal chain of total length $\ell$ (tactics): $G_0 = -\ell$.
Hence

$$
V(s) = \mathbb{E}[G_t \mid s_t = s] = -d(s),
$$

$d(s)$ = expected remaining tactic steps.

**AND states.** If $a$ splits the goal into $N > 1$ independent
subgoals, the environment builds $N$ child states $s^{(j)}$, each with
one open goal $g_j$, all others $\mathrm{internalSorry}$-closed. The
state is then an AND node, and the return is the minimum over children:

$$
G(s) = \min_{j \le N} G\big(s^{(j)}\big)
$$

Instead of the natural sum. Semantic: the value of a state is
$-T_{\mathrm{steps}}$ where $T_{\mathrm{steps}}$ is the number of
tactics in the **longest** branch needed to discharge all subgoals.

**Balancing incentive (derivation).** Let a state split into $k$
subgoals with branch lengths $d_j$ and total work
$W = \sum_j d_j$. Because branches are solved independently,

$$
\max_j d_j \ge \frac{W}{k}, \quad \text{equality iff all } d_j = W/k .
$$

Minimizing $\max_j d_j$ (equivalently maximizing the min-return value)
drives $d_j \to W/k$: subgoals of balanced difficulty are rewarded.
A sum-return objective, $\sum_j d_j = W$, is insensitive to imbalance
and permits one dominating long branch. This explains the paper's
choice of min-return.

## 3. Disproof operator (negation flip)

Private axiom used by the environment:

$$
\mathrm{internal\_true\_if\_false} : (\alpha \to \mathrm{False}) \to \alpha
$$

To disprove goal $(\Gamma \vdash P)$: revert all local hypotheses,
apply the axiom, clean up negated quantifiers. Resulting new context:

$$
\Big(\cdot \vdash \lnot \big(\forall \bar{x}, P(\bar{x})\big)\Big)
$$

i.e. the environment re-targets the fully quantified original goal.
The final disproof term is still kernel-checked, so its conclusion
corresponds to a genuine classical proof of the negation as encoded
by the axiom; in practice this is the only place non-Lean axioms enter
(see also the final Axiom check, §6).

## 4. Environment operations (engineering semantics)

- Parallel execution: multiple threads, many states at once; states
  are save/restore/unique-id-able (snapshot = serialized context).
- Batch queries to the proof network: gather $M$ states, one forward.
- Cost model per tactic: execution time $t_{\mathrm{exec}}(a)$
  (wall-clock cap, S: 10 s), plus Lean's internal heartbeat limit.
- Lean `checkSystem` called more often in critical paths: early abort.
- Integer cap: Lean's arbitrary-precision integers hard-capped so huge
  numerals cannot cause runaway computation.
- Tactic compilation: hot tactics (e.g. `linarith`) compiled to C,
  ~$6\times$ faster term generation.

## 5. Episode protocol

1. Initial state $s_0$ from a formal problem statement $p$.
2. Loop: network proposes candidates $\to$ search applies/validates
   (see 03-search.md).
3. Terminate with `proved` when a goal-free, kernel-checked term for
   $p$ exists; `disproved` when the negated goal is closed;
   otherwise `timeout` at budget exhaustion.

## 6. Final independent verification (soundness gate)

Every claimed proof/disproof is replayed as:

```
lean <file>.lean
```

with a custom command checking the term depends only on the three
standard Lean axioms: propositional extensionality, classical
choice, `Quot.sound`. No other axioms allowed.
