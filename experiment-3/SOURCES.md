# Sources, data access and alternatives

These are primary sources consulted on **2026-10-06**. Dataset and model revisions below were resolved from the official Hugging Face metadata API. Preserve the revisions in your experiment even when `main` later changes. The guide's commands download data when you run them; the full corpus is not bundled here.

## MIRACL Arabic

- [Official MIRACL repository](https://github.com/project-miracl/miracl): collection description, language counts, query/qrels formats, baselines and citation.
- [Topics and relevance judgments](https://huggingface.co/datasets/miracl/miracl): native-speaker queries and positive/negative annotations; the corpus is a separate dataset.
- [Passage collection](https://huggingface.co/datasets/miracl/miracl-corpus): original `docid`, `title` and `text`; Arabic has 2,061,414 passages. Preserve the `article#passage` identifier as a string.
- [MIRACL paper](https://aclanthology.org/2023.tacl-1.63/): benchmark design, judgments, evaluation and limitations.

| Source | Pinned revision | Relevant files |
|---|---|---|
| `miracl/miracl` | `5be20db9509754dadad47689368639fcec739c00` | `miracl-v1.0-ar/topics/topics.miracl-v1.0-ar-{train,dev}.tsv`; `miracl-v1.0-ar/qrels/qrels.miracl-v1.0-ar-{train,dev}.tsv` |
| `miracl/miracl-corpus` | `d921ec7e349ce0d28daf30b2da9da5ee698bef0d` | `miracl-corpus-v1.0-ar/docs-0.jsonl.gz` through `docs-4.jsonl.gz` |

The primary data schemas are `qid<TAB>query` and four-column TREC `qid Q0 docid relevance`. The preparation command converts them to the local files below without changing identifiers or judgments. It stores selected query IDs and SHA-256 file hashes in `provenance.json`.

The dataset cards identify Apache-2.0. Retain their license notices and the MIRACL citation; the collection is derived from Wikipedia, so preserve its upstream source and attribution information when redistributing excerpts. The original diagnostic fixture has separate provenance and the repository's MIT license. Do not relabel third-party downloaded passages as original Brainwave data.

Recommended citation:

```bibtex
@article{zhang2023miracl,
  title = {MIRACL: A Multilingual Retrieval Dataset Covering 18 Diverse Languages},
  author = {Zhang, Xinyu and Thakur, Nandan and Ogundepo, Odunayo and Kamalloo, Ehsan and Alfonso-Hermelo, David and Li, Xiaoguang and Liu, Qun and Rezagholizadeh, Mehdi and Lin, Jimmy},
  journal = {Transactions of the Association for Computational Linguistics},
  volume = {11},
  pages = {1114--1131},
  year = {2023},
  doi = {10.1162/tacl_a_00595}
}
```

## Dense model choice

The guide chooses [`facebook/mcontriever-msmarco`](https://huggingface.co/facebook/mcontriever-msmarco), revision `9ff6abed2c2fdf32bbbd8b4e98fb10160e317375`. The [official MIRACL checkpoint list](https://github.com/project-miracl/miracl#checkpoints-for-dense-models) identifies this checkpoint as not fine-tuned on MIRACL. The [Contriever repository](https://github.com/facebookresearch/contriever) documents its multilingual training and retrieval setup.

Brainwave's adapter uses local SentenceTransformers encoding with normalized vectors and cosine scores. This checkpoint has no SentenceTransformers module configuration, so the adapter's mean-pooling fallback is recorded as part of this experiment. Do not claim exact reproduction of an official mContriever baseline. This is a controlled comparison of external normalizers inside one fixed dense pipeline.

[`intfloat/multilingual-e5-small`](https://huggingface.co/intfloat/multilingual-e5-small) is a practical alternative if you explicitly choose it before the study. Its model card includes MIRACL train among its fine-tuning data; it is not a zero-shot MIRACL model. The card specifies `query: ` and `passage: ` prefixes and a 512-token limit. Changing to E5 requires recording its exact revision and model-specific settings consistently across every arm, and disclosing training exposure. A trained-on-train model evaluated on dev is not automatically dev-label leakage, but it answers a different generalization question.

## Analyzer and metric references

- [Lucene ArabicAnalyzer](https://lucene.apache.org/core/9_12_3/analysis/common/org/apache/lucene/analysis/ar/ArabicAnalyzer.html) documents normalization, light stemming and stop-word filtering. This is a realistic production reference, but it is not the raw controlled tokenizer in this experiment.
- [Lucene ArabicNormalizer](https://lucene.apache.org/core/9_12_3/analysis/common/org/apache/lucene/analysis/ar/ArabicNormalizer.html) documents Alef, Taa Marbuta, Alef Maqsura, harakat and tatweel transformations. Similar labels across libraries do not guarantee identical code-point coverage.
- [NIST trec_eval](https://github.com/usnistgov/trec_eval) is an independent reference evaluator for TREC qrels and ranking files. Match cutoffs, zero-result queries and relevance conventions before comparing numbers.

## If you choose another dataset

[Mr. TyDi](https://huggingface.co/datasets/castorini/mr-tydi) is an alternative multilingual retrieval collection with an Arabic subset. Read its dataset card and use its matching corpus and judgments. Do not combine one collection's qrels with another's passage IDs, even when both use Wikipedia. Use a new dataset ID and protocol version. This guide does not ship an automatic Mr. TyDi downloader.

You can also use a real Arabic search corpus that you are permitted to evaluate, paired with independently written queries and human passage relevance labels. A useful local study needs difficult irrelevant passages as well as positives. Record how queries were sampled, who judged them, and what scope the evidence supports. Keep domain data separate from the authored diagnostics and from MIRACL statistics.

To adapt another source, create UTF-8 files in this exact local format:

```jsonl
{"passage_id":"source-document#0","title":"عنوان اختياري","text":"نص المقطع الأصلي"}
```

```jsonl
{"qid":"q-001","query":"نص سؤال البحث","tags":["optional_predefined_tag"]}
```

```text
q-001 0 source-document#0 1
q-001 0 source-document#1 0
```

The qrels example assumes both passage IDs exist in the corpus. Every evaluated query needs at least one positive judgment. Do not invent negatives for every unjudged passage: preserve the difference between known negative and unjudged. Save a `provenance.json` containing at least source, license, split, scope, source revision, corpus selection and query selection. The same `import --corpus ... --queries ... --qrels ... --provenance ...` command then works.

Existing questions and relevance annotations are the reason to use a benchmark. You do not need to generate a large synthetic dataset to conduct this experiment.
