# Does Arabic Text Normalization Actually Improve Search?

*A controlled experiment with BM25, a multilingual dense retriever, and Arabic relevance judgments.*

> **UNPUBLISHED DRAFT — RESULTS NOT YET RUN.** This is an article prepared for completion after the experiment. Every `TODO` marks evidence or an editorial choice that must be supplied. No sentence below should be read as an observed improvement. The included diagnostic data are fictional engineering examples, not MIRACL results.

An Arabic search query can miss a useful passage without a single change in its intended meaning. A person may type `اجراءات السفر` while a document uses `إجراءات السفر`. A decorative stretch may produce `إجــراءات`. Some documents contain vowel marks; most everyday queries do not.

It is tempting to solve this by normalizing everything. But transformations that make strings match also remove distinctions. `عِلْم` refers to knowledge or science; `عَلَم` can refer to a flag. Removing the vowels gives both the same spelling. Mapping `ى` to `ي` also collapses `على`, a preposition, into `علي`, a person's name.

I wanted to measure this tradeoff rather than assume that cleaner-looking text must produce better search.

## The question

The experiment asks: **How do specific Arabic normalization rules change retrieval quality when everything else in the retrieval system stays fixed?**

Before the measured run, I recorded this hypothesis:

> Applying a conservative set of Arabic normalization rules to queries and documents will improve BM25 nDCG@10 compared with raw text. The dense retriever will change less. More aggressive letter conflation will create additional query-level regressions.

“Conservative” is only a name for a smaller set of transformations. It does not mean that those transformations preserve every linguistic distinction.

Protocol date and version: **TODO**. Any changes made after viewing evaluation results: **TODO: link the deviations log, or state none**.

## Data and scope

The intended benchmark is MIRACL Arabic, which pairs Wikipedia passages with Arabic queries and human relevance judgments. The official Arabic corpus contains 2,061,414 passages. Its public development split has 2,896 queries. The retrieval unit is a passage, so I preserved the original passage identifiers and boundaries. [MIRACL repository](https://github.com/project-miracl/miracl)

**TODO — replace this paragraph with the exact experiment actually completed:** I evaluated `[N queries]` from `[split / fixed sample rule]` against `[N passages]`. The saved dataset revisions are `[revisions]`, and the query IDs and preparation manifest are available at `[repository link]`.

If only the reduced pilot was run, replace the preceding benchmark description with: “This is a MIRACL-derived pilot on a reduced candidate corpus. Its passages were selected partly through relevance judgments, so its scores are not directly comparable with full MIRACL results.” Do not call the pilot a full benchmark evaluation.

A separate, small authored Arabic dataset checks that the normalizer and retrieval code behave as intended. Its deliberately chosen examples illustrate mechanisms. I exclude it from the benchmark averages because a designed stress test cannot tell us how common those mechanisms are.

## What changed between runs

| Condition | External Arabic transformations |
|---|---|
| Raw | None |
| Diacritics | Remove U+064B–U+0652 |
| Tatweel | Remove U+0640 |
| Alef | أ, إ, آ → ا |
| Conservative | The previous three transformations combined |
| Aggressive | Conservative, plus ى→ي and ة→ه |

The main comparison applies the same transformation to the query and the document. Each condition rebuilds its lexical representation or recomputes its document embeddings. Titles, document boundaries, model weights, retrieval depth, relevance judgments, and query selection stay fixed. No condition adds stemming or changes the stop-word list.

That last detail matters. A normal Arabic analyzer may already normalize text. For example, Lucene's Arabic analysis includes normalization, stop-word removal, and light stemming. Calling that pipeline “raw BM25” would make the experiment hard to interpret. My controlled BM25 baseline uses the explicitly recorded tokenizer and no Arabic normalization outside the assigned condition. [Lucene ArabicAnalyzer](https://lucene.apache.org/core/9_12_3/analysis/common/org/apache/lucene/analysis/ar/ArabicAnalyzer.html)

The dense model is **TODO: confirm the executed model and revision**. The planned model is `facebook/mcontriever-msmarco`, which MIRACL's authors identify as a checkpoint without MIRACL fine-tuning. Brainwave uses a fixed mean-pooled, cosine-similarity pipeline for this study; this is not a claim to reproduce the original model's reported benchmark score. [MIRACL checkpoint list](https://github.com/project-miracl/miracl#checkpoints-for-dense-models)

The complete run config records the model revision, tokenizer, pooling, input length, prefixes, batch size, hardware, and software versions. “Raw” for the dense model means no external Arabic transformation; the model still has its own fixed tokenizer.

## How I measured the effect

The primary metric is nDCG@10. It rewards placing relevant passages near the top of the result list. Recall@5, Recall@10, MRR@10, and Recall@100 provide additional views of whether relevant passages are found and how early the first one appears.

For every query, I subtract its raw score from its normalized score. I then calculate a paired bootstrap interval by resampling query identities with replacement. This preserves the pairing between conditions and gives uncertainty around the mean difference. The primary comparison is conservative BM25 versus raw BM25; the remaining comparisons are exploratory.

The judgments are incomplete: a passage without a relevance label receives no credit in these metrics, but it has not necessarily been judged irrelevant by a human. This limits what a small measured difference can tell us.

## Results

**TODO: populate exclusively from saved run artifacts. Do not publish the empty table.**

| Retriever | Condition | nDCG@10 | Δ versus raw | Paired 95% CI for Δ | R@5 | R@10 | MRR@10 | R@100 |
|---|---|---:|---:|---|---:|---:|---:|---:|
| BM25 | Raw | TODO | — | — | TODO | TODO | TODO | TODO |
| BM25 | Diacritics | TODO | TODO | TODO | TODO | TODO | TODO | TODO |
| BM25 | Tatweel | TODO | TODO | TODO | TODO | TODO | TODO | TODO |
| BM25 | Alef | TODO | TODO | TODO | TODO | TODO | TODO | TODO |
| BM25 | Conservative | TODO | TODO | TODO | TODO | TODO | TODO | TODO |
| BM25 | Aggressive | TODO | TODO | TODO | TODO | TODO | TODO | TODO |
| Dense | Raw | TODO | — | — | TODO | TODO | TODO | TODO |
| Dense | Diacritics | TODO | TODO | TODO | TODO | TODO | TODO | TODO |
| Dense | Tatweel | TODO | TODO | TODO | TODO | TODO | TODO | TODO |
| Dense | Alef | TODO | TODO | TODO | TODO | TODO | TODO | TODO |
| Dense | Conservative | TODO | TODO | TODO | TODO | TODO | TODO | TODO |
| Dense | Aggressive | TODO | TODO | TODO | TODO | TODO | TODO | TODO |

**TODO: one factual paragraph stating the primary effect, interval, sample size, and practical importance.** If the interval crosses zero, say the result is uncertain; avoid claiming that normalization has no effect. Report absolute metric points separately from relative percentages.

**TODO: add one figure of paired nDCG@10 deltas with intervals and one win/loss/tie chart.** Label the corpus and query scope on each figure.

## Where the averages hide something useful

First, I measured how often each transformation actually changes text. If a rule barely activates in this collection, a small aggregate effect is unsurprising. I measured document changes as well as query changes: an untouched query can still retrieve a different ranking after the documents are normalized.

**TODO: report the changed-query and changed-passage counts for every rule.**

Second, I inspected the largest gains, largest losses, and a fixed random sample of ties. A useful case study needs the original query, original passage, normalized strings, before/after ranks, and relevance labels. An attractive linguistic story is not enough to establish why a ranking changed.

**Case 1 — a gain:** TODO: real query ID, Arabic text, English explanation, relevant passage ID, ranks, metric difference, mechanism supported by the trace.

**Case 2 — a regression:** TODO: same evidence, including the competing passage and whether the loss follows a collision, altered term statistics, truncation, or another cause.

**Case 3 — a tie or negligible change:** TODO: explain whether the transformation did not activate, the model already treated the inputs similarly, or the changed ranking left the metric unchanged. Do not infer internal model behavior without evidence.

For named entities, I used **TODO: annotation method, sample size, and label timing**. This subgroup analysis is exploratory unless it was specified and labeled before inspecting the outcomes. If the data do not support a named-entity conclusion, remove that claim.

## What this experiment can and cannot support

These results apply to `[actual corpus]`, `[query population]`, and `[the two fixed retrieval pipelines]`. They do not establish a universal best Arabic normalizer. Wikipedia passages differ from dialectal messages, OCR text, legal archives, and fully vocalized educational material. One dense model also cannot represent all Arabic-capable embedding models.

The experiment holds the model fixed, so it measures inference-time normalization. It does not test whether training an embedding model on normalized Arabic would lead to a different result.

**TODO: disclose resource constraints, reduced corpus selection if used, tokenizer preprocessing, truncation, missing runs, and any protocol deviations.**

## What I would use in practice

**TODO — write this only after results exist.** State which exact transformations the evidence supports for which retriever, the observed cost, and where uncertainty remains. If the result is mixed or nearly zero, make that the finding.

The reusable part of this work is the experiment: preserved original text, explicit transformation rules, fixed retrieval systems, paired comparisons, and examples of both gains and losses. The code, data preparation instructions, protocol, and saved run artifacts are available at **TODO: public repository URL and release/commit**.

---

## Author's publication checklist — remove from published article

- Every number comes from a saved run, with dataset/model revisions.
- The title and opening accurately reflect whether a pilot or full corpus was tested.
- All twelve declared arms are shown or their omissions explained.
- The primary comparison has a paired interval and a clearly stated denominator.
- Original diagnostic examples are never presented as benchmark findings.
- The interpretation acknowledges uncertainty and adverse cases.
- Dataset/model/code references and upstream attribution are retained.
- No `TODO`, editorial instruction, or hypothetical result remains.
- A clean reproduction produces matching BM25 scores; dense tolerance and hardware are documented.
- Publish manually after final review; this file has not been submitted to Medium.
