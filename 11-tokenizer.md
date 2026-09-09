# 11 Tokenization (byte-pair encoding for states and tactics)

## 1. Problem

The whole pipeline depends on a tokenizer that compresses both Lean
tactic states (context) and math/code text (pretraining). Cost model:
attention runs $O(L^2)$ so tokenizer quality directly scales search
throughput. Goal: minimize average sequence length on the *state
distribution* (not the tactic distribution — tactics are short).

## 2. BPE algorithm

Corpus $\mathcal{C}$ = sequence of characters (or UTF-8 bytes,
byte-level BPE, S: byte-level with 256 seed bytes + reserved
Lean-specific symbols). Vocab $V_0$ = byte alphabet. Iteration:

1. count adjacent pairs in the current tokenized corpus:
   $f(a,b) = \#\{(x_i, x_{i+1}) = (a,b)\}$;
2. merge the best pair $(a^{\ast}, b^{\ast})$ subject to score;
3. replace every occurrence $a^{\ast} b^{\ast}$ by a new token;
4. repeat until $|V| = N_{\mathrm{vocab}}$ (S: $128\mathrm{K}$).

Greedy frequency rule chooses $\operatorname*{argmax}_{(a,b)} f(a,b)$.
MinBPE criterion (S, used here) maximizes corpus likelihood under a
unigram model $p(x_i) = f_i / T$, $T$ = total tokens: merging $(a,b)$
into $ab$ changes the objective by

$$
\Delta \mathcal{L}(a,b) = f(a,b)\, \log \frac{f(a,b)\, T}{f(a)\, f(b)}
$$

justification: pre-merge the token sequence contains $a$ and $b$ with
counts $f_a, f_b$; post-merge their mass rewrites to $f_{ab}$; the
log-likelihood ratio is exactly the term above, which is symmetric,
non-negative for co-occurrence-above-chance pairs, and prefers
overlap between frequent primitives — the pairwise MI-like criterion.

## 3. Special tokens and Lean-aware reservation

- `<|startofstate|>`, `<|endofstate|>`, `<|true|>` (goal closed),
  `<|pad|>`;
- S: reserve a range for the most frequent Mathlib identifiers and
  tactics (`linarith`, `rw`, `simp`, ...), or better: start merges
  with seeded units = Mathlib identifier atoms, so that tactic names
  stay single tokens;
- S: no normalization should destroy Lean syntax (no Unicode NFKC
  merging of operators like $\times$ used by Lean pretty-printer).

## 4. Suggested training recipe (S)

- Corpus mix: 1 : 1 : 1 code (Python/C-like), math in LaTeX
  (arXiv/training dumps), converted Mathlib `.lean` sources,
  stripped of proofs if the auto-formalizer targets olympiad
  statements.
- Train amount: 10GB–100GB; T = 2–20GB considered standard.
- Deduplicate; exclude evaluation problems per data-separation rule.
- Evaluate on a held-out set of tactic-state pretty-prints with the
  compression ratio

$$
\mathrm{CR} = \frac{\sum_x |\mathrm{tok}(x)|}{\sum_x |\mathrm{bytes}(x)|}
$$

  and on the *validity-preserving* metric: the first
  $L_{\max} = 4096$ tokens must retain all Mathlib-aware content
  (target: 99% of states fully fit without truncation at CR to be
  measured).

## 5. Ablation to run (see 20, E12)

Vocab sizes $\{32\mathrm{K}, 64\mathrm{K}, 128\mathrm{K}, 256\mathrm{K}\}$:
greedy BPE vs minBPE vs unigram
(SentencePiece, for comparison), quality metrics CR and solve-rate
with fixed model size. Expected: $64\mathrm{K}$–$128\mathrm{K}$ gives
the sweet spot: too small vocab → long sequences (attention cost);
too big → rare tokens never seen in math/Lean stats ($\approx$ data
efficiency loss).

## 6. Practical notes

- Encoder–decoder share the same vocabulary (alpha-proof-style);
  this is what lets the decoder *tactics* reuse the same symbol space.
- Hyphens: `-` inside math integers is a separate concern: the env
  normalizes tokens but the tokenizer should avoid splitting negative
  numeric literals (`-42` one token; Lean prints them with parens).
- Quantization effects: measure token-coverage of hypothesis
  identifiers; those that don't fit hurt the value head precision
  (it only sees the first 4096 tokens).
