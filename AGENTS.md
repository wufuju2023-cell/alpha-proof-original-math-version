# AGENTS.md — AlphaProof reconstruction dossier

This repository is a **mathematical reconstruction dossier** of
AlphaProof (Hubert et al., Nature 2025), for humans and agents who
want to:

1. **Learn** AlphaProof — how the pieces work, at formula depth;
2. **Rebuild** AlphaProof (open source stack) — procedural recipe;
3. **Compare** the rebuilt system against the original — measure gaps
   and advances.

## 0. What is in this repo

- `refs-alpha-proof-paper-nature-2025.md` — the original Nature
  article (open access, CC BY 4.0; do not remove the copyright notice
  or the citation below). The embedded `images/...` figure files of
  the source archive are NOT included.
- `NN-*.md` (00–20) — the reconstruction dossier:
  - `00-overview.md` — notation, module map, global constants;
  - `01..08` — core pipeline (Lean env / proof network / search /
    pretrain-SFT / auto-formalization / main RL / TTRL / benchmarks);
  - `09` — open-source reconstruction plan;
  - `10..16` — deep-dive math (transformers, tokenizer, RL theory,
    MCTS theory, PUCT+temperatures, search alternatives, LLM
    post-training choices);
  - `17` — design-choice ledger (every knob + why + experiment id);
  - `18` — TTRL advanced math;
  - `19` — concrete open-source training recipe;
  - `20` — experiment suite E1–E15 (protocols + decision rules).

Citation used throughout:
T. Hubert et al., "Olympiad-level formal mathematical reasoning with
reinforcement learning", Nature (2025),
doi:10.1038/s41586-025-09833-y.

## 1. Reading order

- **Learn (paper author intent):** 00 → 01 → 03 → 02 → 04 → 06 →
  05 → 07 → 08; then choose: transformers `10`, MCTS `13`,
  PUCT `14`; compare with alternatives `15`, `16`.
- **Rebuild:** 09 first (stack), then 19 (recipe), then 20 (how to
  pick each knob). Backfill from 01–08 when implementing a specific
  module.
- **Compare (rebuild vs original):** anchors are 08 (numbers from the
  paper) and 20 (measurement protocol). Follow the protocol in
  `## 4. Comparison workflow` below.
- **If you are an agent short on context:** read `00-overview.md`
  fully first (it is the shortest module); then only the module your
  task touches; NEVER assume constants from `00` or any `(S)` value
  are facts from the paper.

## 2. Hard rules for editing this repo

1. **Paper facts vs suggestions.** Anything marked **(S)** is a
   suggestion, not the paper. Do not silently drop or promote the
   marker. When you add a constant, mark it exactly like:
   `$c_{\mathrm{base}} = 19652$ (S)`. When you report a paper number,
   cite the section/table (e.g. "Table 1; Fig. 4a").
2. **No tables.** This repo targets phone-size reading. Tables are
   forbidden; replace with bullet lists or prose lines.
3. **Math syntax.** Follow the md-latex-rule skill (installed at
   `~/.config/opencode/skills/md-latex-rule/`):
   - block formulas `$$...$$` on their own lines, zero indent, blank
     line before AND after;
   - inline `$...$` never spans lines, always separated from ASCII
     letters/digits by a space;
   - no Unicode math symbols inside formulas (use `\le \in \times \to
     \neg \wedge` etc.);
   - before finishing any edit, run on the modules (NOT on
     `refs-*.md`, which is a verbatim source copy and intentionally
     not lint-clean):

     ```
     node ~/.config/opencode/skills/md-latex-rule/scripts/check_md_math.mjs [0-9][0-9]-*.md
     ```

     and require `OK` (or `REAL REMAINING ISSUES: 0`).
4. **Module numbering.** New modules continue from `21`. New
   experiments continue from E16 (in `20-experiment-suite.md`). One
   module per file; keep files under ~200 lines.
5. **Division of claims.** Math deductions belong in modules;
   *measured* results belong in `08` or in `reports/NNN-*.md` (never
   in modules); reproduction notes belong in `artifacts/` (gitignored).
6. **Language.** Math-first, minimal prose. Chinese/English mixed is
   acceptable but formulas are universal.

## 3. Data hygiene

- Never introduce evaluation-benchmark problems (miniF2F, formal-imo,
  PutnamBench-test, IMO-2024 live) into training corpora, and never
  keep auto-formalized statements similar to them in the curriculum
  (redaction rules in `05` §4 and `08` §1).
- Keep hyperparameter metadata in the module where the constant is
  defined; do not duplicate definitions without pointing to the
  source.
- The paper itself was kept as reference; if a figure's text is
  needed, re-read it from the markdown in `refs/`.

## 4. Comparison workflow (rebuild vs original)

Use it when the "compare" task runs:

1. Fix computational definitions: solve rate formula
   (`08` §3), compute budget per problem (TPU-h per problem),
   benchmark splits (`08` §1). Never mix budgets.
2. Benchmarks: same splits for both sides: miniF2F-valid/test
   (corrected version), formal-imo 258 problems, PutnamBench-test 189
   (even years ≥1990). Exclude geometry (Mathlib gap) like original.
3. Run the equal-compute protocol: same sim budget per problem,
   measured in *total compute* (GPU-days), not wall clock; seeds ≥ 3;
   report mean ± 95% bootstrap CI over per-problem results
   (method in `20` §0).
4. Compare against the original numbers stored in `08`:
   - 2 TPU-min: m2f-test 96.3 / formal-imo 33.2 / putnam-test 27.9;
   - 12 TPU-h: 97.7 / 43.7 / 39.4;
   - TTRL 500 TPU-d: 99.6 / 58.3 / 56.1.
   Use the equivalent-compute conversion
   (approx 1 TPU-v6e-day ≈ 0.6–0.8 H100-day) from `09` §4 but always
   state the conversion in the report.
5. Record in `reports/NNN-topic.md`: setup (config, hardware,
   budget), raw metric table as bullet lines, gap analysis
   (which component explains most of the gap: search? value? auto-
   formalization? curriculum?), and a "advances" section where the
   rebuild beats the original (e.g. better efficiency at low budget,
   better subject coverage, cleaner engineering).

## 5. Workflow with agents

- If a task touches math rendering, load skill `md-latex-rule`.
- If a task is a long build/install/run of the Lean toolchain or
  training, use the global rules of the user config
  (4-minute command cap, checkpoint-driven batching, idempotent
  commands). See `~/.config/opencode/AGENTS.md` (global rules).
- Prefer writing experiments as code stubs in `artifacts/` with
  conf-JSON, one directory per experiment (see `20` §0 for naming);
  never run experiments inside this doc repo except for
  reproduction.

## 6. Known gaps / open items

- All (S) constants are unvalidated: first priority experiments are
  E1 (γ), E4 (K, τ), E6 (λ) — see `20`.
- Supplementary tables of the paper (hyperparameters) are not in the
  public article text; `(S)` stands in, and the sidebar should be
  updated as soon as verified.
- Geometry problems were not handled by AlphaProof itself; the
  AlphaGeometry 2 interface is out of scope here.
- The auto-formalization chain-of-thought data and the variant
  generator exemplar set (791 pairs) are not public; open substitutes
  must be built (LLM few-shot) and their quality measured (E11
  protocol).
