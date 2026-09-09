# 05 Auto-formalization (statement curriculum factory)

Maps natural-language (LaTeX) problem statements to Lean statements:
the bridge that makes $8\times10^{7}$ RL start positions possible.

## 1. Model and training data

Specialist: Gemini 1.5 Pro, fine-tuned on triplets
$(P_{\mathrm{nl}},\, \mathrm{CoT},\, F_{\mathrm{lean}})$:

$$
\mathcal{L}_{\mathrm{af}} =
-\log p_\theta\big( F_{\mathrm{lean}},\, \mathrm{CoT} \mid P_{\mathrm{nl}} \big)
$$

Data seed: $2{,}500$ expert-formalized problems (pre-algebra to IMO
difficulty); $\sim 50$ with expert-written detailed CoT. Prompting
public Gemini with the expert examples generates CoTs for the rest →
$\approx 7{,}000$ triplets; low-quality CoTs removed by formatting
checks (ad-hoc regex/structural rules).

## 2. Iterative refinement (STaR, paper ref. 60)

Given golden formalizations $F^{\ast}$ for a subset:

```
repeat:
    (CoT, F) ~ M_θ(· | P)          # sample candidate
    if equiv_check(F, F*):          # §3
        add (P, CoT, F) to data
    re-fine-tune θ
```

until $\approx 7\times10^{4}$ high-quality triplets.

## 3. Equivalence check by low-compute prover

$$
\mathcal{E}(F, F^{\ast}) :\Longleftrightarrow
\mathrm{Prover}_{\mathrm{low}}\ \text{derives}\
\vdash \mathrm{type\_of\%\ @}F = \mathrm{type\_of\%\ @}F^{\ast}
$$

i.e. the prover is tasked with certifying that the **type** of the
generated declaration equals the type of the golden declaration
(definitional equality up to proof search). Accepted traces only.

## 4. Dataset generation (80M)

- $\approx 10^{6}$ informal statements as sources (competition
  problems, textbook problems, variations).
- Multiple translations per source (sampling temperature; diversity
  goal): expected multiplicity $m \approx 80$, giving

$$
|\mathcal{D}| \approx 10^{6} \times 80 = 8\times10^{7}
$$

- Any correct candidate out of the $m$ suffices; incorrect ones are
  still **valid** Lean statements → they become (often quickly
  refutable) RL instances: good for disproof learning.
- Answer-shaped problems ("Find all…"): plausible answers collected
  alongside the statement (see 08 §IMO protocol for the k = 500
  answer generation) and injected into the statement before
  formalization; the placeholder is formalized with `sorry` and left
  for the prover to refute/confirm.
- Deduplication + redaction: any auto-formalized document similar to
  evaluation-benchmark problems is removed (data separation, 08 §2).
- Continuous augmentation: re-apply improved pipeline to the source
  set after every model upgrade.

## 5. Metrics and measured quality

Internal A/B benchmark: 50 IMO + 50 Putnam statements, single
candidate per problem, human panel ($\approx 3$ Lean experts).

$$
\mathrm{pass@1} = \frac{\#\{\text{problems with correct formalization}\}}{50}
$$

Results (Extended Data Table 1 of paper):

- IMO: 60.0% overall; algebra 81.3%, number theory 76.9%,
  combinatorics 33.3%; pass@k $= 96\%$ at $k = 29$.
- Putnam: 64.0% overall; number theory 72.7%, algebra 61.9%,
  combinatorics 60.0%, geometry 54.6%; pass@k $= 98\%$ at $k = 16$.

Pass@k calculation (single-pool sampling):

$$
\mathrm{pass@k} = 1 -
\frac{\binom{n-c}{k}}{\binom{n}{k}}
$$

with a small correction for many-trial sampling; empirically: pooling
$k$ samples per problem and asking if $\ge 1$ is correct.

Significance: even at pass@1 = 60%, multi-sample (pass@k ≈ 96–98%)
plus the disproof machinery of main RL makes the 80M curriculum
usable: the ~2% of wrong-but-rifle statements mostly yield disproof
training signal rather than noise.

## 6. Notes

- Only statements (not proofs) are auto-formalized; proofs come from
  the agent's own RL.
- The auto-formalizer itself is RL-free: supervised CoT fine-tuning +
  STaR. Its "reward" = the equivalence check (prover-verified).
- Compute: $\approx 10^{5}$ TPU-days total for the project (paper),
  dominated by prompt+fine-tune iterations, not single-shot inference.
