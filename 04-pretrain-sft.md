# 04 Pretraining and supervised fine-tuning

## 1. Pretraining

Data: $\approx 3\times10^{11}$ tokens of public code + mathematical
text. Distributional disclaimers: all Lean code is excluded; all
formal proofs of evaluation benchmarks excluded (prevents leak).

Objective: next-token prediction with regularization:

$$
\mathcal{L}_{\mathrm{pt}} =
-\mathbb{E}_{x}\Big[\sum_{t \in \mathcal{M}} \log p_\theta\big(
x_t \mid x_{<t},\, x_{\setminus \mathcal{M}} \big)\Big]
+ \lambda_{\mathrm{drop}} \cdot \mathrm{dropout}
$$

- $\mathcal{M}$: masked span positions; span reconstruction loss
  applied only on (a superset of) corrupted tokens (T5-style masking,
  S: span probability $0.15$, span length $\sim$ Geometric(3), limited
  to 3 spans per sample? — in practice: masked span reconstruction as
  an auxiliary regularizer).
- Dropout kept on during pretraining as regularization (paper: "with
  dropout and masked span reconstruction for regularization").

Token totals (paper): encoder saw $\approx 12\,\mathrm{T}$ tokens,
decoder $\approx 3\,\mathrm{T}$ tokens, over $\approx 50$ epochs of
the 300B corpus:

$$
E_{\mathrm{enc}} \approx \frac{12\mathrm{T}}{300\mathrm{B}} = 40,
\qquad
E_{\mathrm{dec}} \approx \frac{3\mathrm{T}}{300\mathrm{B}} = 10,
\qquad E_{\mathrm{enc}} + E_{\mathrm{dec}} = 50
$$

Compute (approx FLOPs, $\mathrm{FLOPs} \approx 6 N_p D$):

$$
6 \cdot 3\times10^{9} \cdot 15\times10^{12}
\approx 2.7\times10^{23} \ \mathrm{FLOPs}
$$

Purpose: encoder learns to read code-like/math text; decoder learns to
emit structured, syntax-correct strings, plus general math vocabulary.

## 2. Supervised fine-tuning (SFT)

Data: $\approx 3\times10^{5}$ (state, tactic) pairs extracted from
human-authored Mathlib proofs, $\approx 5\times10^{6}$ tactic tokens.

Policy loss (chain-rule over tactic tokens, only tactic positions):

$$
\mathcal{L}_{\mathrm{sft}} =
-\frac{1}{|a|}\sum_{j=1}^{|a|}
\log \pi_\theta\big(a_j \mid s, a_{<j}\big)
$$

Value head initialization: from the same Mathlib proofs, for each
state $s_t$ of a human proof we know the exact remaining cost:

$$
d^{\mathrm{gt}}(s_t) = \ell - t, \quad \ell = \text{total tactics in the proof}
$$

binned into the categorical targets:

$$
\mathcal{L}_{\mathrm{v}} =
-\log p_v\big(\mathrm{bin}(d^{\mathrm{gt}}(s_t)) \mid s_t\big)
$$

This injects "proof hard/soft" knowledge into the value head before
RL begins.

Extraction procedure (needed for reproduction):

1. parse each Mathlib theorem into tactic sequence (tactic = one
   action; ignore comments/example blocks);
2. at each position: record pretty-printed state + next tactic;
3. filter: no `sorry`, state length $\le L_{\max}$, tactic length cap.

Fidelity detail: skip `:=`-term proofs; keep tactic-mode proofs
splitting at each `;`-separated tactic.

## 3. Cost

SFT: $\approx 10$ TPU-days (paper). Sanity:
$3\times10^{5}$ pairs $\times$ short sequences $\approx 10^{8}$ tokens
$\Rightarrow$ trivial versus pretraining.

## 4. Joint statement of inductive bias

Pretraining → syntax & math knowledge; SFT → Lean-grammar, tactic
idioms, initial proof-length prior; RL (06) → actual proving strategy
beyond imitation.
