# Experiment 3 protocol — fill and freeze before measured runs

Status: **template; not yet preregistered or executed**. Record a dated commit or immutable copy before inspecting MIRACL development results. Any later change belongs in the deviations table, with its reason and timing.

## Question and hypotheses

Question: Under a fixed retrieval pipeline, how does Arabic orthographic normalization affect retrieval quality, and which query/document characteristics explain gains and losses?

Primary hypothesis: applying the conservative preset to both queries and passages improves mean BM25 nDCG@10 compared with raw text on the declared evaluation population.

Secondary hypotheses: the dense retriever changes less than BM25; aggressive normalization causes more query-level regressions than conservative normalization; named-entity queries are vulnerable to character conflation. These are hypotheses, not promised findings. A zero or negative primary difference is a valid outcome.

## Freeze this decision record


| Decision                      | Initial specification / fill before running                                                                                        |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Author, date, protocol commit | Ahmad, Oct 11th, V1                                                                                                                |
| Dataset scope                 | `full-corpus-all-dev-queries` for the main claim; `diagnostic` and `miracl-pilot` with `split: train` are development exercises    |
| Evaluation query split        | MIRACL Arabic dev; use train only for code debugging / any hyperparameter tuning                                                   |
| Query population              | All dev queries, or a declared deterministic sample with saved IDs; do not select by outcomes                                      |
| Corpus                        | Complete official Arabic passage collection for benchmark claims; preserve passage IDs                                             |
| Topics/qrels revision         | `5be20db9509754dadad47689368639fcec739c00`                                                                                         |
| Corpus revision               | `d921ec7e349ce0d28daf30b2da9da5ee698bef0d`                                                                                         |
| Retrieval unit                | Original passage; no new chunking, deduplication, or relabeling                                                                    |
| Input text                    | Title + newline + passage text; same rule in every arm                                                                             |
| Primary comparison            | BM25 conservative minus BM25 raw, both-side normalization                                                                          |
| Primary metric                | Per-query nDCG@10, averaged equally across queries                                                                                 |
| Secondary metrics             | Recall@5, Recall@10, MRR@10, Recall@100                                                                                            |
| Retrieval depth               | 100                                                                                                                                |
| BM25                          | Record implementation, tokenizer, k1, b, and tie-breaking from exported run                                                        |
| Dense model                   | `facebook/mcontriever-msmarco`, revision `9ff6abed2c2fdf32bbbd8b4e98fb10160e317375`                                                |
| Dense pipeline                | SentenceTransformers mean pooling, L2-normalized vectors / cosine; no text prefixes; length 256; batch 32; record effective config |
| Dense interpretation          | A fixed Brainwave pipeline; not a reproduction of the original dot-product baseline                                                |
| Normalization side            | Both query and document; query-only is a separately labeled secondary study                                                        |
| Bootstrap                     | Paired query resampling with replacement, 10,000 samples, seed 42, percentile 95% interval                                         |
| Primary effect scale          | Absolute mean metric difference; also report relative change only if baseline > 0                                                  |
| Practical effect threshold    | TODO: choose before seeing results, or state explicitly that no practical threshold is set                                         |
| Other comparisons             | Exploratory; no family-wide significance claim from unadjusted individual intervals                                                |
| Manual review                 | Five largest nDCG wins, five largest losses, and ten random ties per chosen comparison; report fewer when unavailable              |
| Named-entity labels           | Label a fixed random query sample before viewing rankings; save annotator, rubric, disagreements and counts                        |
| Hardware/software             | CPU/GPU/RAM, OS, Python, package lock, model/cache revision, git revisions                                                         |


The public dev set is already visible; this is preregistration of your analysis, not a claim of a private blind test. If you inspect dev outcomes to change transforms, report that phase as exploratory. Do not silently promote a tuned dev result into a held-out claim.

## Exact primary transformations

These are the dedicated Arabic Norm experiment presets. Their names do **not** mean the same thing as the older, broader general Eval Lab presets.


| Preset       | Transformations, in order                             |
| ------------ | ----------------------------------------------------- |
| raw          | No external normalization.                            |
| diacritics   | Remove U+064B–U+0652 only.                            |
| tatweel      | Remove U+0640 only.                                   |
| alef         | U+0623 أ, U+0625 إ, U+0622 آ → U+0627 ا.              |
| conservative | Diacritics + tatweel + Alef above.                    |
| aggressive   | Conservative + U+0649 ى→U+064A ي + U+0629 ة→U+0647 ه. |


No preset adds stemming, stop words, digit conversion, Unicode NFC/NFKC, wasla ٱ replacement, standalone hamza deletion, ؤ/ئ replacement, or punctuation conversion. Those can be follow-up studies after this one. If the model tokenizer transforms text internally, keep that tokenizer fixed and disclose that raw means **no external Arabic normalization**, not absence of internal model preprocessing.

## Statistical and interpretation rules

For each query, compute `d_i = metric(candidate_i) - metric(raw_i)`. Resample the same query indices on both sides and calculate the mean difference for every bootstrap sample. Report the observed delta and the 2.5th/97.5th percentiles. Do not subtract independently bootstrapped marginal intervals. If the interval includes zero, say the data do not distinguish the sign at that uncertainty level; do not claim equivalence. Very few queries yield fragile intervals. If related queries create dependence, discuss it and consider a separately documented cluster bootstrap.

The different retrievers must use identical corpus IDs, query IDs, qrels, and tie policy. Retain relevant passages missing from an index in the recall denominator; missing positives are a data-integrity failure for this experiment, not grounds to make recall easier. Unjudged MIRACL passages receive zero credit in standard qrels-based scoring; this does not prove that humans judged them irrelevant.

Measure activation on both queries and documents. An unchanged query can change rank because document text, lexical document frequency, or embeddings changed. A query-changed subset is a useful description, not a complete accounting of the treatment's effect.

Show all declared variants, even losing ones. Preserve original Arabic, normalized text, passage IDs, ranks, and metric deltas for case studies. Report collision examples as mechanisms; the tiny authored fixture is not evidence of their real-world frequency.

## Deviations log


| Date | Change | Reason | Were evaluation results already viewed? | Analysis affected |
| ---- | ------ | ------ | --------------------------------------- | ----------------- |
| TODO |        |        |                                         |                   |




## Completion criteria

- Data provenance, complete configs and rankings are saved.
- The raw BM25 run reproduces when rerun on the same data.
- The normalizer/tokenizer checks and metric tests pass.
- All declared arms either have results or a documented failure; missing dense runs are not represented as completed.
- The main result includes a paired interval and win/loss/tie counts.
- Case studies include both gains and regressions where they exist.
- Claims match the actual corpus/query scope, model, and preprocessing.
- Every `TODO` in the article is resolved or the affected claim removed before publication.

