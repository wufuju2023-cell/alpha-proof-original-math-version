# 07 Test-time reinforcement learning (TTRL)

Purpose: problems where large tree search still fails. Re-apply the
full RL machinery to a **bespoke curriculum of synthetic variants** of
the one hard target $T$.

## 1. Variant generation

Generator $\mathcal{G}$: Gemini LLM (few-shot, curated set
$\Omega$ of $791$ (problem, variant) Lean pairs). Prompt strategy
drawn per call: single variant, or a correlated set (problem
decomposition, simulated proof steps).

Heuristic classes encoded in the generator (Pólya-style, ref. 75–80):

- Simplification: transformation with $A' \subset A$, constants
  substituted, bounds reduced:

$$
\big[\forall x\!\in\!A, P(x)\big] \mapsto
\big[\forall x\!\in\!A', P'(x)\big]
$$

- Generalization: free variables lifted, weaker conclusions kept.
- Lemma proposal: extract subclaim $C$ with $C \to P$ plausible.
- Analogy: swap the algebraic structure
  $\lfloor \cdot \rfloor \to \lceil \cdot \rceil$,
  $S \to -S$, index shift $k \to k+1$, domain
  $\mathbb{R} \to \mathbb{Q}$.
- Reformulation: equality $\leftrightarrow$ mutual inclusion,
  $\forall \leftrightarrow$ finite-set conditions.
- Proof steps: emit intermediate goals of a known skeleton.
- Decomposition: split conjunction / conditional into separate
  statements.

Programmatic operator set $\Sigma$ (scheduled deterministically):
hypothesis/goal mutations: numeric perturbation ($0 \to 0.6$),
negated sums, strengthened hypotheses, $\exists$ vs $\forall$
switches, subset vs equality, domain restriction.

## 2. Validation and evolutionary expansion

Every candidate validated to be *syntactically valid Lean*
(parses, type-checks as an open statement). Deduplicated by
normalized syntax.

Evolutionary loop (seeds → richer curriculum):

$$
V_0 = \mathrm{valid}(\mathcal{G}(\Omega, T)), \qquad
V_{i+1} = V_i \,\cup\, \mathcal{G}\big(\mathrm{top}\text{-}k \text{ by }
\mathrm{sim}(T, v)_{v \in V_i}\big), \qquad i < N_{\mathrm{evo}}
$$

($N_{\mathrm{evo}} = 15$, paper) with string similarity
$\mathrm{sim}$ (token Jaccard / edit distance, S). Result:

$$
|V_T| \approx 10^{5}\text{–}10^{6}\ \text{unique valid variants}
$$

(paper: "hundreds of thousands"). Examples structured as shown in
Extended Data Fig. 5 (IMO 2024 P1: floor→integer set, rational
restriction, ceiling reformulations, sum negation, strengthened
$\exists$–lemma forms, etc.).

## 3. Focused RL

Initialize specialist $\theta_T \leftarrow \theta^{\ast}_{\mathrm{mainRL}}$.
Start-position dataset for the matchmaker:

$$
\mathcal{D}_T = \{T\} \cup V_T
$$

(identical procedural machinery as 06 §2–§4: same matchmaker rules,
same 90/10 mix, same losses).

Termination rule: once $T$ itself is proved, matchmaker stops
assigning {T} ∪ V_T (solved target ⇒ the loop's purpose achieved by
definition $T$ ok). Further variants keep training while an
unsolved $T$ remains open.

Batch TTRL: concurrent targets $T_1, \ldots, T_m$ → shared start set
$\mathcal{D} = \bigcup_j \mathcal{D}_{T_j}$ with one matchmaker.

## 4. What TTRL buys: scaling law presentation

For target $T$ with curriculum $V_T$, define:

$$
\mathrm{solve}(c) = \#\{T_j : \text{proved within } c\ \text{TPU-days}\}
$$

Measurements (paper, Fig. 4b):

- initial fast phase: many new solves within first 50 TPU-days;
- sustained growth well past tree-search saturation;
- +15 absolute percentage points vs 12 TPU-h search on formal-imo
  and PutnamBench-test at 500 TPU-days;
- eval at 4,000 sims per problem.

Ablation math (Extended Data Fig. 3):

- curriculum size $|V_T|$: Top-10 → Top-100k: monotone improvement of
  both (a) solved-variant proportion in TTRL train set, (b) target
  prove rate;
- generator strength: Gemini 2.0 Flash S variants solve *worse*
  inside the curriculum but *teaching* = more effective: harder
  training curriculum ⇒ better final target solve rate
  (bottom-line chosen: 2.0 Flash S for generation). Signifies a
  curriculum-difficulty optimum, not simply easier-is-better.

## 5. Why it works (structural argument)

Every variant $v$ shares the *mathematical skeleton* of $T$ (the
lemmas, bounds, invariants), so weight updates from
$\{\,s \mid s \in \text{successful}(\mathcal{D}_T)\}$ refine the
specialist around $T$'s local proof structure; the min-return value
signal (06 §4) makes search budgets focus on the hardest branches,
which are precisely the branches relevant to $T$. This is the formal
version of "practice on related, simpler problems" (Pólya).

## 6. Costs

- Variant generation: $(N_{\mathrm{evo}} + 1)$ rounds × prompts +
  Lean validation — small relative to RL.
- Focused RL: 50–500 TPU-days per target family; IMO 2024 runs:
  2–3 days per problem at the reported scale.
