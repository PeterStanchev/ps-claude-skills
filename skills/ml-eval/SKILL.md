---
name: ml-eval
description: Choose and compute the right ML/LLM/RAG evaluation metrics, and validate automatic scores against human judgment. Use when evaluating model output quality, picking a metric for regression/classification/text-generation/ranking, scoring generated-vs-reference text, building an eval harness, or checking whether an automatic judge agrees with humans.
---

# ML evaluation: pick the metric, compute it, validate it

## 1. Pick by task
- **Regression** → R², MSE/RMSE, MAE, MAPE. If errors are heteroscedastic or you need uncertainty, predict a distribution (**Gaussian NLL**) and report **prediction corridors** (μ ± nσ); check calibration, not just point error.
- **Classification** → precision/recall/F1; macro- or weighted-F1 under imbalance; **set-based / hierarchical F1** when labels form a tree; a confusion matrix to see *where* it fails. If two classes share a genuinely ambiguous boundary, **merge them or confidence-gate** rather than training harder.
- **Text generation / fuzzy string** → edit-distance family (§2). Token-F1/BLEU are weak on short strings and on reordering.
- **Ranking / retrieval** → rank-correlation (§3) plus nDCG / MRR.

## 2. Text-generation scorers
- **EED (Extended Edit Distance)** — Levenshtein extended with a *jump* operation + a per-character *visit penalty*. Training-free → language-independent; more reorder-tolerant than plain edit distance and more robust than BLEU on short strings. Interface: `EED(hyp, ref)` over character lists — a solid drop-in "generated vs gold" scorer.
- **CharacTER** — character-level TER **normalized by hypothesis length** (not reference — this counters TER's short-output bias), with a word-level shift allowed when word edit-distance is below a threshold.

## 3. LLM-judge / agreement with humans
- **Binary "harsh" accuracy** — correct/incorrect per field, no partial credit; clean signal for thresholds and downstream ML.
- **Agreement-with-human** — before trusting any automatic judge at scale, correlate its scores against human ratings with **Pearson** (linear), **Spearman** (monotonic), and **Kendall's τ** (pairwise), and **handle ties explicitly**. Low agreement = the metric is lying to you.

## 4. Label quality
- Manually-verified **ground truth** is the canonical baseline. Confident-learning tools (e.g. CleanLab) surface likely-mislabeled examples — use them as a **filter/auditor**, never as the final accuracy number.

## Practice
Match the metric to the decision being made; report uncertainty, not just point estimates; and validate any automatic metric against human judgment before relying on it.
