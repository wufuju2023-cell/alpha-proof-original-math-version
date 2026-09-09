# 02 Proof network (architecture)

3B-parameter encoder–decoder transformer (Vaswani et al. 2017;
AlphaCode-style), three outputs: latent state, policy, value.

## 1. Encoder

Input: $\mathrm{pp}(s)$, tokenized with a shared code/math vocabulary
(S: 128K tokens, Lean + Mathlib tokenizer):

$$
x = \mathrm{tok}(\mathrm{pp}(s)) \in V^{L}, \quad L \le L_{\max}
$$

Encoder output (latent of current proof state):

$$
h = \mathrm{Enc}(x) \in \mathbb{R}^{L \times d}, \qquad
\bar{h} = \mathrm{Enc}(x)[0] \quad (\text{summary vector})
$$

State normalization done at tokenization: deterministic pretty-print,
canonical hypothesis ordering, $\alpha$-renaming, duplicate merge.

## 2. Policy head (decoder)

Autoregressive over tactic tokens, cross-attending to $h$:

$$
\pi_\theta(a \mid s) = \prod_{j} p_\theta\big(a_j \mid a_{<j}, h\big)
$$

Sampling: $K$ tactic strings drawn at inference (S: $K = 64$,
top-k/temperature $\tau_{\mathrm{sample}} = 0.8$, length cap 128
tokens). Distinct generation happens through sampling; candidates are
deduplicated and validated by the environment afterwards.

## 3. Value head (categorical, MuZero-style)

Head atop the encoder summary:

$$
p_v(\cdot \mid \bar{h}) = \mathrm{softmax}(\mathrm{MLP}(\bar{h})) \in \Delta^{B-1}
$$

S: $B$ buckets covering remaining steps $d \in \{0, \ldots, D_{\max}\}$
with $D_{\max} = 512$ (log-spaced; the paper just says categorical,
ref. 72 pattern):

$$
\widehat{V}(s) = -\sum_b m_b\, p_v(b \mid s), \qquad m_b = \text{bucket midpoints}
$$

The value target is the observed return $G$ (binned), i.e. the value
head regression target for a state is $G(s) = -d(s)$.

## 4. Suggested dimensions (total $\approx 3\,\mathrm{B}$)

- $d_{\mathrm{model}} = 2048$, heads $= 32$, FFN $= 8192$.
- Encoder: 32 layers $\approx 12 \cdot d^2 \cdot 32 \approx 1.6\,\mathrm{B}$ params.
- Decoder: 20 layers $\approx 1.0\,\mathrm{B}$ params.
- Embeddings (tied input/output): $128\mathrm{K} \cdot 2048 \approx 0.26\,\mathrm{B}$.
- Value MLP (2048 to 2048 to 512) $\approx 0.01\,\mathrm{B}$.
- Total $\approx 2.9\,\mathrm{B}$, reported as "3B".

Parameter count model: per layer $|W| \approx 12 d^2$ (attention
$4d^2$ $+$ FFN $8d^2$ $(2\times d \times 4d)$), ignoring biases.

## 5. Training signals summary

1. Pretraining (04): next-token prediction + masked span
   reconstruction, 300B tokens, $\approx 50$ epochs.
2. SFT (04): cross-entropy on 300K (state, tactic) pairs; value head
   initialized by regressing known remaining-step counts.
3. RL (06): policy CE on self-generated successful proof/disproof
   states; value CE on achieved returns. 10% SFT replay mix.

## 6. Notes on architecture choices

- Encoder processes the full (potentially long) state; decoder output
  is short (tactics) — asymmetry explains 12T encoder-tokens vs 3T
  decoder-tokens in pretraining.
- Sample-based decoding (ref. 18, Sampled MuZero) matches tree search
  expansion: $K$ i.i.d. draws rather than a top-k or beam.
- The value head predicts *remaining effort*, not win probability;
  this is the meaning of "value function estimating the expected
  return" with return $= -\text{steps}$.
