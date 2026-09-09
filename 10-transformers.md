# 10 Transformers: encoder, decoder, attention (full math)

## 1. Input representation

Tactic state string $x = \mathrm{tok}(\mathrm{pp}(s))$ with
$|x| = L \le L_{\max}$ (S: 4096). Embedding:

$$
h^{(0)} = \mathrm{Embed}(x) + \mathrm{RoPE}^{(\mathrm{abs})}(x), \quad
\mathrm{Embed} \in \mathbb{R}^{|V| \times d}
$$

Position information is injected via rotary encoding (§6); no learned
positional matrix needed.

## 2. Self-attention

For attention head $l$ with projections
$W^{Q}_l, W^{K}_l, W^{V}_l \in \mathbb{R}^{d \times d_k}$:

$$
A_l = \mathrm{softmax}\Big( \frac{Q_l K_l^{\top}}{\sqrt{d_k}} + M \Big),
\qquad
\mathrm{Att}_l(h) = A_l V_l
$$

- $Q_l = h W^{Q}_l$, $K_l = h W^{K}_l$, $V_l = h W^{V}_l$;
- $M$: additive mask, $-\infty$ in the masked positions. Encoder:
  $M = 0$ everywhere. Decoder: causal $M_{ij} = -\infty$ for $j > i$.
  Cross-attention: mask by the state (tokens of goal/hypotheses all
  visible, but decoder sees its own past causally).
- Multi-head concatenation:

$$
\mathrm{MHA}(h) = \mathrm{Concat}_l \big(\mathrm{Att}_l(h)\big) W^{O}, \quad
W^{O} \in \mathbb{R}^{(H d_k) \times d}
$$

$H = d / d_k$ heads (S: $H = 32$, $d_k = 64$).

Why softmax: it makes self-attention a normalized convex combination
(non-negative weights sum to 1), giving a well-defined learned
"similarity routing" and gradient flow through all pairs. Numerical
form: subtract per-row max before exponentiating; online split:
with $m = \max_j z_j$, $l = \sum_j e^{z_j - m}$, standard stable logit
scaling is the LSE trick.

## 3. Blocks and norms

Pre-norm transformer block (encoder), identical for decoder with
causal + cross attention:

$$
\tilde{x}^{(l)} = x^{(l)} + \mathrm{MHA}\big(\mathrm{RMSNorm}(x^{(l)})\big), \qquad
x^{(l+1)} = \tilde{x}^{(l)} + \mathrm{FFN}\big(\mathrm{RMSNorm}(\tilde{x}^{(l)})\big)
$$

RMSNorm (chosen over LayerNorm in modern LMs):

$$
\mathrm{RMSNorm}(x) = \gamma \odot \frac{x}{\sqrt{\frac{1}{d} \sum_i x_i^2 + \varepsilon}}
$$

$\gamma \in \mathbb{R}^d$: learned scale; no mean-subtraction needed —
rescaling the whole vector is what matters for stability of the
softmax/QK products. (S: $\varepsilon = 10^{-6}$.)

FFN (SwiGLU, S): with $W_1, W_2 \in \mathbb{R}^{d \times 4d}$,
$W_g \in \mathbb{R}^{d \times 4d}$, $b$'s biases, $\sigma$ = sigmoid:

$$
\mathrm{FFN}(x) = \big( x W_1 \odot (x W_g)\,\sigma(x W_g) \big) W_2
$$

## 4. Encoder / decoder asymmetry (why 12T vs 3T tokens)

Encoder reads long context (the full tactic state, hypotheses +
goals) — expensive at $O(L^2\,d)$ per layer. Decoder emits short
strings (tactics, ~10–100 tokens). In pretraining the encoder and
decoder are trained on corrupted/reconstructed segments respectively:
encoder forward pass mostly processes *context*, decoder mostly
*target* — the 4:1 token throughput ratio of the paper.

## 5. Cross-attention (decoder ↔ encoder)

Decoder layer with latent $z$ (decoder hidden sequence) and encoder
states $h^{\mathrm{enc}} = (h_1, \ldots, h_L)$:

$$
Q = z\, W^{Q}_{l}, \qquad K = h^{\mathrm{enc}}\, W^{K}_{l}, \qquad
V = h^{\mathrm{enc}}\, W^{V}_{l}
$$

$$ \mathrm{CrossAtt}(z, h^{\mathrm{enc}})_i =
\sum_j \frac{\exp\big(\frac{q_i \cdot k_j}{\sqrt{d_k}}\big)}{\sum_{j'} \exp\big(\frac{q_i \cdot k_{j'}}{\sqrt{d_k}}\big)}\, v_j $$

All encoder positions are visible: the decoder can condition on any
hypothesis or goal, which is why *whole-state context* tokens are
worth keeping for the tactic head.

## 6. RoPE (rotary positional embeddings) — S

For channel pairs $(x_{2i}, x_{2i+1})$ and
$\theta_i = 10000^{-2i/d}$: position $m$ rotates the pair:

$$
(x_{2i}, x_{2i+1}) \mapsto
\big(x_{2i} \cos m\theta_i - x_{2i+1}\sin m\theta_i,\;
x_{2i} \sin m\theta_i + x_{2i+1}\cos m\theta_i\big)
$$

Key property (relative encoding falls out of the dot product):
$R_m^{\top} R_n = R_{n-m}$, so

$$
\langle R_m q, R_n k \rangle = \langle q, k \rangle \cos\big((m-n)\theta_i\big)
+ \langle q_{\perp}, k_{\perp} \rangle \sin\big((m-n)\theta_i\big)
$$

i.e. attention between tokens $m, n$ depends only on $n - m$: the
network can learn shift-invariant tactics idioms (e.g., rewriting
patterns, arithmetic facts).

## 7. Value head (exact)

Summary vector $\bar{h} = h^{\mathrm{enc}}[0]$ (or RMS-pooled mean of
the encoded state, S):

$$
\mathrm{MLP}(\bar{h}) = W_2\,\mathrm{SiLU}\big(W_1\,\mathrm{RMSNorm}(\bar{h})\big), \quad
p_v = \mathrm{softmax}\big(\mathrm{MLP}(\bar{h})\big) \in \mathbb{R}^B
$$

with $B$ buckets over remaining steps $d \in [0, D_{\max}]$
(S: $B = 512$, $D_{\max} = 512$);
$\widehat{V}(s) = -\sum_b m_b\, p_v[b]$, $m_b$ = buckets midpoints.

## 8. Computation cost

Per forward-pass FLOPs (dense, no Flash optimizations):

$$
\mathrm{FLOPs} \approx 6\, N_\theta \quad \text{per token} \quad
\Rightarrow \mathrm{FLOPs}(x) = 6\, N_\theta\, L
$$

Attention is $O(L^2 d)$ per layer; for $L = 4096$ this dominates: the
reason the state is truncated and the context-length tradeoff be
tuned: raising $L_{\max}$ costs quadratically in search throughput.
Flash attention (S: use it) keeps memory cost rather than compute:
the online-softmax rescaling rule:

$$
m_{\mathrm{new}} = \max(m, m_2), \qquad
l_{\mathrm{new}} = l\, e^{m - m_{\mathrm{new}}} + l_2 e^{m_2 - m_{\mathrm{new}}}
$$

$$ o_{\mathrm{new}} = \frac{o\, e^{m - m_{\mathrm{new}}} + o_2\, e^{m_2 - m_{\mathrm{new}}}}{l_{\mathrm{new}}} $$

(sketch of one block of the tiling algorithm).

## 9. Inference-time temperature and sampling

Logits $z \in \mathbb{R}^{|V|}$ (last decoder hidden × output
embedding):

$$
p_\tau(v) = \frac{\exp(z_v /\tau)}{\sum_{v'} \exp(z_{v'} /\tau)}
$$

- $\tau \to 0$: distribution collapses to $\mathrm{argmax}$ (greedy);
- $\tau \to \infty$: uniform over vocabulary (entropy
  $\to \log |V|$);
- entropy $H(p_\tau)$ is monotone non-decreasing in $\tau$
  (the family is log-convex, so $p_\tau$ majorizes $p_{\tau'}$ for
  $\tau \ge \tau'$: larger $\tau$ = more exploration).

Top-p (nucleus) and top-k are hard truncations used to prevent the
long tail: support = smallest prefix with cumulative mass
$\ge \hat{p}$. (S for tactic generation: $\tau_{\mathrm{gen}} = 0.8$,
top-p $= 0.95$, length cap 128.)

## 10. KV-cache decode cost

At decode step $t$ with cached keys/values: memory $O(t \cdot d)$ per
layer, compute $O(t \cdot d)$ per token instead of $O(L^2 d)$. In
AlphaProof the decoder generates only short tactics, so batching the
$K$ candidates of one state in a single forward pass is efficient.
