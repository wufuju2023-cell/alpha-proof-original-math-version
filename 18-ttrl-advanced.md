# 18 TTRL in full detail (math of the curriculum)

## 1. The abstract statement

Fix a target formal problem $T$. TTRL builds a local curriculum
$V_T$ around $T$ and runs the full RL machinery on
$\{T\} \cup V_T$:

$$
\mathrm{TTRL}: \quad (T, V_T, N_{\mathrm{evo}}, \mathcal{G}) \to
\theta_{\mathrm{spec}}, \quad \text{stop upon solve}(T)
$$

## 2. Variant operators (transformations of the statement)

Statement = $(\Gamma \vdash P)$ (hypotheses, goal). Operators
$\Sigma$ (each deterministic when scheduled):

- $\mathrm{Simp}$: $\Gamma \vdash P \to \Gamma'' \vdash P''$ with
  $\Gamma'' \supseteq \Gamma$ (dropping hypotheses),
  concrete substitution (constants, arithmetic), type narrowing
  ($\mathbb{R} \to \mathbb{Q}$);
- $\mathrm{Gen}$: generalize $P$ (replace constants by variables),
  weaken conclusion $P \to P_{\mathrm{weaker}}$;
- $\mathrm{Strengthen}$: $P \to P_{\mathrm{stronger}}$ (harder,
  gives reachability clues);
- $\mathrm{Lemma}$: propose $C$ (lemma) and output
  $C$ and $C \to P$ as two targets;
- $\mathrm{ProofStep}$: output intermediate $M$ of a plausible
  chain $P \Rightarrow M \Rightarrow$;
- $\mathrm{Analog}$: swap structural tokens: floor/ceil, sum/neg,
  index shift, add 0.6 arbitrary-values, subset/equality;
- $\mathrm{Reform}$: equality ↔ equivalence, ∀ ↔ specific forms;
- $\mathrm{Restrict}$: restricting to a subset of parameters
  (e.g., $n > 0 \to n > 3$).

The paper's Extended Data Fig. 5 exactly demonstrates these on the
IMO 2024 P1 target: (a) $\mathbb{R} \to \mathbb{Q}$ restriction;
(b) strengthen the property:

$$
(n \mid \sum_i \lfloor i\alpha \rfloor) \wedge
(n \mid \sum_i \lfloor (1+i)\alpha \rfloor)
$$

(c) lemma with bound $|\alpha - k| < 1/4$; (d) rational-lemma;
(e) proof step "$\alpha$ must be integer"; (f) intermediate identity
with reformulation in the condition; (g) ceiling reformulation;
(h) negated sums; (i) analogous subset statement; (j) parameter
perturbation +0.6.

## 3. Programmatic component (counts and guarantees)

The LLM co-generator is *prompt-stochastic*; the programmatic
component ensures the coverage floor:

$$
V_T^{\mathrm{prog}} = \bigcup_{\sigma \in \Sigma^{\mathrm{prog}}}
\sigma^{(r)}\big(T\big) \quad \text{(systematic variants, local)}
$$

then each candidate must *parse* in Lean and be *syntactically
valid* (open statement typechecks): denote
$\mathrm{Valid}(\cdot)$. Deduplication by canonicalization of the
Lean term (S: the pretty-printer's normalized string:
meta-level `hash` of the elaborated declaration, ignoring name
aliasing).

## 4. Evolutionary expansion (N_evo rounds)

With $\mathrm{sim}(\cdot,\cdot)$ = token-level Jaccard or
Levenshtein on the pretty-printed statement; $k$-Nearest expansion:

$$
V_{i+1} = V_i \cup \mathrm{Valid}\big( \mathcal{G}( \mathrm{prompt}(T,
\Omega, \Pi_i) ) \big), \qquad
\Pi_i = \mathrm{top}_k \big[ V_i \text{ by } \mathrm{sim}(T,\cdot) \big]
$$

$\Omega = 791$ curated (problem, variant) pairs, $\Pi_i$ = sampled
prompting strategy, $k \approx 5$-10 (S), $N_{\mathrm{evo}} = 15$
(paper). Final size ~ hundreds of thousands 10^5 — 10^6.

## 5. Similarity = curriculum slope (difficulty estimation)

Hypothesis (supported by Extended Data Fig. 3): *hard-curriculum
variants teach better per unit of gradient*. Difficulty proxy of a
variant $v$:

$$
\mathrm{diff}(v) = 1 - \mathrm{sim}(T, v)
$$

or better: use the *fraction of variants eventually proved in a
short run* as realizable difficulty proxy (since the same prover
measures it):

$$
\mathrm{diff}_{\mathrm{emp}}(v) = \frac{1}{1 + \mathrm{solveRate}_{500}(v)}
$$

**Curriculum scheduling choice (S):** order the TTRL pass two times:
ascending $\mathrm{diff}$ (easy-first), descending (hard-first).
The 2024 ablation suggests the optimum sits at the hard end:
hard-first (or the paper's natural-selection homogeneous mix which
lets the solver order itself via value-guided simulation, equivalent
to our recommendation). Schedule in E11 and adopt best.

## 6. What to train on (exact same machinery)

Identical losses to main RL (06 §4):

$$
\mathcal{L}_{\mathrm{pol}} = -\log \pi_\theta(a \mid s) \quad
\text{(state-act on successful path)}, \qquad
\mathcal{L}_{\mathrm{val}} = -\log p_v(\mathrm{bin}(G(s)) \mid s)
$$

with a *replay window* per target (S): buffer of successful states
of the TTRL run (not cross-target spillover unless shared
matchmaker). The 10% SFT mixing should be *re-sampled* from Mathlib
during TTRL (else the policy drifts to the variant domain; it
finished with the solver at IMO 2024 within 2-3 days so drift is
mild).

## 7. Stopping rules and budget allocation (multi-target)

Single target: stop when $\mathrm{solved}(T)$ (paper) — the
theos-tum model. Multi-target: shared matchmaker with global budget
$B_{\mathrm{tot}}$; allocation between targets
$\beta_j = \mathrm{TPU-days}_j$; (S) use a targeted bandit over
targets with feedback = marginal success rate over recent TTRL
days:

$$
\rho_j(t) = \frac{\partial \Pr[\mathrm{solve}(T_j)]}{\partial \beta_j}
$$

allocate $\Delta \beta_j \propto \rho_j$ with a minimum floor (this
is a standard "marginal gains" assignment); stop-when-solved per T.

## 8. Data validity & avoidance of catastrophic forgetting

Forgetting during TTRL: the specialist is initialized from the
generalist; if TTRL is too long the model forgets general skills
(scoring on sibling held-out targets degrades). Guard metric
(monitor): sibling-problem solve rate before/during TTRL.
Shelf-life rule: stop when
$\Delta(\text{heldout}) < 0$ for 2 consecutive evals. Also:
(e) recompute the SFT mix so the general-class probing stays.

## 9. Compute budget of TTRL at full scale

- variant generation: 100-500 GPU-hours per target (prompts ~ 1e5
  calls + validation);
- focused RL: 50-500 TPU-days per target (paper; 2-3 days per IMO
  2024 problem at their scale);
- expected values: +15pp on formal-imo & PutnamBench-test at 500
  TPU-days (paper Table 1).
