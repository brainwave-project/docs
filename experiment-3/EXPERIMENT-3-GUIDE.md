# Experiment 3 — Does Arabic normalization improve retrieval?

This guide takes you from a small Arabic diagnostic exercise to a reproducible MIRACL experiment, then to a Medium article supported by saved evidence. You compare controlled BM25 with one frozen multilingual dense model, using six explicit normalization conditions.

**Start here:** complete Steps 1–5 with the included fixture. They need no Docker, login, Ollama, know-hub, or model download. Use the MIRACL pilot to learn the workflow, then decide whether you can afford the full corpus before making a benchmark claim.

The implementation and authored fixture are ready to run. **No MIRACL result or dense-model result is supplied with this guide.** The article is an unpublished draft with results placeholders.

## What is in this folder

| File | Use |
|---|---|
| `EXPERIMENT-3-GUIDE.md` | Follow these steps in order. |
| `PROTOCOL.md` | Record the hypothesis and analysis decisions before measured runs. |
| `ARTICLE-DRAFT.md` | Fill with your measured results, case studies and limitations before publishing. |
| `SOURCES.md` | Official dataset/model links, pinned revisions, alternatives and attribution notes. |
| `data/diagnostic-bundle.json` | Import directly into the new Arabic Norm page. |
| `data/diagnostic/` | The same original data as corpus/queries JSONL, TREC qrels and provenance. |
| `templates/run-register.tsv` | Record the identity and conditions of each run. |
| `templates/failure-analysis.tsv` | Record the evidence behind your case studies. |

New downloads and intermediate work belong in `work/`, which is ignored by Git. Do not commit the complete MIRACL corpus or embedding matrices. Save selected small publication artifacts separately after reviewing them.

## Step 1 — Write down the question

**Purpose:** Decide exactly what you will test before viewing measured results.

**Do:** open `PROTOCOL.md`. Fill in your name, date, intended evaluation scope, hardware, and any practical effect threshold. Read the exact transformations table before proceeding.

The main question is whether **conservative normalization of both queries and passages improves BM25 nDCG@10 versus raw text**. Dense retrieval and the other transformations are additional comparisons.

Do not choose a preferred transform after seeing dev scores and then describe that choice as a prediction. Keep a dated protocol copy and log changes. Use the fixture and the **train-split pilot** to debug. The final measured study uses **dev**, after the protocol is frozen. The downloader supports both splits; always pass `--split` explicitly in your record.

**Expect:** a falsifiable hypothesis, one primary comparison, fixed normalization rules and a clear statement of whether the final claim targets the full corpus or a pilot.

## Step 2 — Start the local Arabic Norm workspace

**Purpose:** Start the API and UI used to import datasets, run comparisons and inspect results.

Prerequisites: Docker, Make, and the existing UI Node/npm dependencies. Run the commands from the `infra` directory. Keep both terminals running while using the UI.

In **Terminal A**:

```text
make evals-lab
```

In **Terminal B**:

```text
make evals-ui
```

Open [Arabic Norm](http://localhost:5173/evaluation/normalization), under the Evaluation navigation. Development mode requires no session.

**Expect:** a page with three panels: **Choose the evidence**, **Run the controlled comparison**, and **Read the evidence**. Arabic Norm uses its own passage dataset store; it does not require a Golden Sets dataset, an indexing job in know-hub, or a generation judge.

If an existing eval-api already occupies port 3005, use that API if it includes these changes. Otherwise stop that specific instance before running `make evals-lab`. Do not run two servers on the same port. All Make shortcuts below execute inside this Docker eval-api and therefore share its mounted store.

## Step 3 — Import the original diagnostic data

**Purpose:** Load the small authored dataset used to verify that the experiment works correctly.

**Do:** click **Import JSON** in the first panel and select:

```text
C:\Users\ALIEN WARE\repo\brainwave\docs\experiment-3\data\diagnostic-bundle.json
```

**Expect:** 16 passages, 12 queries, and 192 judgments. Scope should say `diagnostic`. Expand **Provenance and dataset hashes** and confirm the source is the original Brainwave fixture.

Read `data/diagnostic/README.md` for the intended meaning of each query. The examples deliberately include collisions, including `عِلْم` / `عَلَم`, and plain-text controls. Every query-passage pair is labeled. These authored labels are for checking the experiment; they are not an independent estimate of real search quality.

## Step 4 — Run the six BM25 conditions

**Purpose:** Compare raw text with the five declared normalization treatments while keeping the retrieval setup fixed.

Set these fields in **Run the controlled comparison**:

| Field | Value |
|---|---|
| Retrieval systems | Controlled BM25 |
| Apply normalization to | Queries + passages (primary) |
| Bootstrap resamples | 1000 for this diagnostic exercise |
| Random seed | 42 |
| Conditions | raw, diacritics, tatweel, alef, conservative, aggressive |

Click **Start comparison** and wait for `done`.

**Expect:** six rows in the score table. The same corpus, queries, qrels and retrieval depth are used for every row. BM25 rebuilds a representation for each document-side transformation. Raw is always present as the reference.

This is a controlled BM25 implementation, with its tokenizer stated on the page. It does not use the platform's English PostgreSQL text-search index and is not an exact reproduction of MIRACL's official Lucene analyzer. Lucene's Arabic analyzer already performs normalization and stemming; it would not be an unnormalized baseline. [Lucene analyzer documentation](https://lucene.apache.org/core/9_12_3/analysis/common/org/apache/lucene/analysis/ar/ArabicAnalyzer.html)

**Learn:** scores do not have to improve. This fixture intentionally overrepresents spelling mismatches and collisions. Its intervals quantify variation among twelve authored queries, not uncertainty about Arabic search in general.

## Step 5 — Inspect what changed

**Purpose:** Examine individual queries and rankings to understand gains, losses, ties and character collisions.

1. Click **bm25 · conservative** in the score table or use **Inspect condition**.
2. Keep the metric at `ndcg@10` in the paired-comparison table. Read the delta, interval, and win/loss/tie count together.
3. Step through query IDs with the arrows. Compare the original query, normalized query and two ranked lists.
4. Inspect `diag-q04`: the query itself has no vowel marks, but the relevant passage does. A document change can matter when the query is unchanged.
5. Inspect `diag-q09` under diacritics or conservative: `عِلْم` and `عَلَم` collapse. A collision may create a tie without causing a rank loss under this corpus's tie order.
6. Inspect `diag-q07` and `diag-q11` under aggressive: look for the competing passage after `ى→ي` or `ة→ه`.
7. Expand **Query slices**. The tags describe authored test cases; they are not new human labels for MIRACL.
8. Click **Full report JSON** and keep the export with the run ID.

The page shows the top ten passage IDs per query. Click a passage ID in either ranking to compare its original text with the document normalization used by the selected condition. This view precedes model prefixes, tokenization and truncation. The full report retains depth-100 rankings and per-query scores; do not infer a cause from rank changes alone.

**Checkpoint:** you can explain one match gained, one distinction collapsed, and why an unchanged query can still be affected. Do not expect a gain for every preset. Digits and standalone hamza are deliberately outside these six presets.

## Step 6 — Reproduce the exercise from the CLI

**Purpose:** Confirm that the command-line workflow reproduces the diagnostic results shown in the UI.

Run from the `infra` directory:

```text
make evals-arabic-diagnostic
```

The shortcut prints the exact normalization options, imports the diagnostic files, runs all six BM25 conditions, and saves the generated dataset ID and run ID for later Make commands.

Click **Refresh** on the UI's dataset panel to see the CLI import, then select its saved comparison. Rerun the same BM25 command and confirm aggregate scores and rankings match. Timestamps and run IDs will differ.

To list imported datasets and completed runs later, use:

```text
make evals-arabic-list
```

## Step 7 — Prepare a bounded MIRACL pilot

**Purpose:** Download official MIRACL data and build a smaller train-split pilot for testing the workflow at manageable cost.

The official source has three components: the **complete passage corpus**, **query topics**, and **relevance judgments**. Do not build the main evaluation corpus using only the passages listed under a query's positive examples; that makes retrieval artificially easy.

The preparation command has two underlying modes (`pilot` and `full`), used in these three common ways:

| Mode | What it includes | What you may claim |
|---|---|---|
| `pilot` | A seeded query sample, all judged passages for those queries, and seeded background passages | A MIRACL-derived reduced-corpus pilot; scores are not comparable with full MIRACL. |
| `full --split dev` | Complete Arabic corpus and all dev queries | Full-corpus evaluation; verify counts and preserve the preparation manifest. |
| `full --split dev` with `--query-limit` | Complete Arabic corpus, seeded subset of dev queries | Full-corpus result on the declared query sample, not the entire dev set. |

**Download cost:** the pilot retains a small corpus but still streams **all five Arabic corpus shards** to choose its background passages without taking only the first articles. It saves indexing and embedding work, not the initial network scan. No Hugging Face dataset script or remote Python code is executed.

Run from the `infra` directory after `make evals-lab`. The output directory must not already exist:

```text
make evals-arabic-pilot-prepare
```

This shortcut selects 100 train queries with seed 42, adds 1,000 seeded background passages, pins both official source revisions, and writes the prepared files under `docs/experiment-3/work/`.

**Expect:** `corpus.jsonl`, `queries.jsonl`, `qrels.tsv`, and `provenance.json`. The final manifest states `miracl-pilot`, `split: train`, query selection, retained IDs, immutable source revisions and file hashes. Its corpus selection is explicitly `label_informed: true`. The total passage count is the union of judged passages plus the requested background sample, so it is not simply 1,000.

If the transfer fails, an `INCOMPLETE` marker remains and the CLI refuses to import that directory. Choose a new output directory for a retry; there is no resumable-download promise. Check network/proxy access and free disk space first.

## Step 8 — Run and inspect the MIRACL pilot

**Purpose:** Import the prepared pilot and run all six controlled BM25 normalization conditions.

```text
make evals-arabic-pilot-run
```

This shortcut imports the four files from Step 7, captures the generated dataset ID, runs BM25 with all six presets on queries and passages, and prints the saved report path.

Refresh the dataset list on Arabic Norm and repeat Step 5. Record the run ID in the run register. Inspect activation counts before interpreting a near-zero effect: a rule can only alter external text where its target occurs, but changed document frequencies can also affect an unchanged query.

The prepared query tags identify literal character patterns, such as `contains_alef_variants`. They do **not** identify named entities. For a named-entity subgroup claim, label a predefined random query sample and save the labeling rubric and timing.

## Step 9 — Add one frozen dense model

**Purpose:** Compare BM25 with one fixed multilingual dense retriever without changing its model configuration between conditions.

The planned dense model is `facebook/mcontriever-msmarco`, at commit `9ff6abed2c2fdf32bbbd8b4e98fb10160e317375`. MIRACL's official repository lists it as not fine-tuned on MIRACL. This avoids describing a MIRACL-trained checkpoint as zero-shot. [Official checkpoint list](https://github.com/project-miracl/miracl#checkpoints-for-dense-models)

Brainwave uses SentenceTransformers' mean-pooling fallback for this checkpoint, L2-normalized vectors and cosine similarity. It records the model configuration and revision. Treat this as a fixed internal dense baseline; the original mContriever evaluation uses a different retrieval setup. Do not compare the absolute score as if you reproduced its official baseline.

Install the optional dependency and run the tiny diagnostic dataset:

```text
make evals-arabic-dense-diagnostic
```

The first dense run downloads model weights if needed. Encoding happens locally. An error loading the model is an error, not a fallback to BM25. Empty query/passage prefixes are the defaults and appropriate for this selected model. The maximum length includes title and passage tokens; normalization may alter token count and truncation, which is part of the effect you need to discuss.

Once this succeeds, run all six dense pilot conditions:

```text
make evals-arabic-dense-pilot
```

The diagnostic shortcut automatically imports the tiny fixture if Step 6 has not already saved its dataset ID. Both dense shortcuts install the optional dense dependency in the eval-api environment, use the exact model revision, length 256, batch 32 and empty prefixes, and save their generated IDs. A GPU can reduce time, but no fixed runtime is promised. The browser API uses the same environment, so no separate restart command is needed.

## Step 10 — Budget the full-corpus experiment

**Purpose:** Check storage, memory and runtime requirements before committing to the complete MIRACL evaluation.

The full corpus contains 2,061,414 passages, with 2,896 dev queries according to the official MIRACL release. Preserve passage IDs and validate your downloaded counts against its pinned manifest. [MIRACL data details](https://github.com/project-miracl/miracl)

The CLI stores BM25 postings in SQLite and dense embeddings in disk-backed arrays. That bounds memory use, but a full six-condition dense run is still substantial. For 768-dimensional float32 embeddings, six corpus matrices alone require approximately **38 GB decimal**, before raw files, database copies, lexical indexes, query data or reports. Budget additional disk and measure your pilot encoding throughput before starting. Full MIRACL runtime/performance has not been validated in the delivered change.

A valid intermediate option is the **complete corpus with a fixed query sample**: it reduces query scoring work while preserving the retrieval problem. It does not reduce the cost of embedding every passage. Report the query sample explicitly.

When ready, prepare and import the full corpus:

```text
make evals-arabic-full-prepare
```

This downloads the complete Arabic corpus and all dev queries, imports them, and saves the generated dataset ID for the following commands.

The command above selects all available dev queries, so the scope should be `full-corpus-all-dev-queries`. To use a fixed sample of 100 dev queries instead, run `make evals-arabic-full-prepare query_limit=100`. That produces `full-corpus-query-subset`; change the article claims accordingly.

Freeze the completed protocol before inspecting the main results. Run BM25:

```text
make evals-arabic-full-bm25
```

Run dense separately so each expensive stage produces its own complete report:

```text
make evals-arabic-full-dense
```

Do not compare conditions from different dataset imports unless corpus, query and qrels hashes agree. If a full run exceeds your resources, publish a clearly titled pilot or reduce the question before collecting a replacement evaluation. Do not fill the article's dense rows using lexical scores.

## Step 11 — Analyze paired effects and failure cases

**Purpose:** Measure query-level differences and inspect concrete examples behind improvements and regressions.

Use `PROTOCOL.md` as the analysis contract. The report includes means, query-level scores, activation counts, descriptive slices, and paired 95% bootstrap intervals for every candidate versus its own retriever's raw condition.

- Lead with the primary absolute delta and its interval. Distinguish `+0.02 nDCG` from `+2% relative`.
- Report all declared conditions, including negative and near-zero results.
- Read win/loss/tie counts: the same average can come from broad small gains or a few large ones.
- Treat intervals for secondary comparisons as exploratory. Individual 95% intervals are not a multiple-comparison correction.
- Inspect five largest wins, five largest losses and ten randomly chosen ties where available. Use the failure-analysis template and save passage IDs, original text and before/after ranks.
- Do not call unjudged passages human-confirmed negatives. Standard benchmark scoring gives them zero credit; the label coverage is a limitation.

For a direct summary of the main BM25 paired comparison and its five largest wins and losses:

```text
make evals-arabic-main-analyze
```

Use the UI for pilot inspection and for selecting random ties with the preregistered seed; the Make summary intentionally shows only the primary comparison and largest deltas.

## Step 12 — Export and independently check rankings

**Purpose:** Save standard TREC run files and verify the main metrics with an independent evaluator.

The UI's **Full report JSON** retains the full analysis and original retrieval scores. The CLI additionally writes standard six-column TREC runs. Their score column uses strictly decreasing rank-based integers, so an independent evaluator preserves the measured ranking even when original retrieval scores tie. [trec_eval reads scores and ignores the rank column](https://github.com/usnistgov/trec_eval/blob/main/get_trec_results.c).

```text
make evals-arabic-main-export
```

Use these files with a standard evaluator, such as [NIST trec_eval](https://github.com/usnistgov/trec_eval), against the exact saved qrels. On a system where `trec_eval` is installed, a typical independent check is:

```text
trec_eval -c -m ndcg_cut.10 -m recall.5,10,100 -m recip_rank qrels.tsv bm25-raw.trec
```

`recip_rank` on a depth-100 file is **not** MRR@10: truncate to ranks ≤10 in a separate copy first when checking that metric. The `-c` option includes all queries in the supplied qrels, assigning zero to queries absent from the run. Supply exactly the imported query population's qrels: a query with no retrieved hits still contributes zero in Brainwave. The primary nDCG and recall checks should agree to rounding when the same queries and binary judgments are used.

Keep your protocol, git revisions, data manifests, package lock, run JSONs, TREC outputs and hardware notes together. Repeat the raw run in a clean environment before publication. A clean reconstruction should recover the same data hashes and deterministic BM25 ranking; document numerical tolerances for dense retrieval.

## Step 13 — Finish and publish the article

**Purpose:** Replace draft placeholders with measured evidence and limit every claim to the experiment actually completed.

Open `ARTICLE-DRAFT.md`. Replace every `TODO` with measured evidence or remove the associated claim. Add the aggregate table, an interval plot, a win/loss/tie figure and three real query case studies. Clearly label every figure's corpus, query count and retriever.

If only the pilot was completed, say so in the title, methods and conclusion. A credible pilot with narrow claims is publishable; calling it full MIRACL would not be supported. The included diagnostic examples may illustrate transformations if you identify them as authored examples.

Use the draft's publication checklist, then publish the final article manually to Medium. Nothing in this folder has been published or submitted. Your public repository should contain reproducible code, manifests, commands and small result artifacts; link to the official data rather than copying the full corpus into Git.

## Suggested month

| Week | Deliverable |
|---|---|
| 1 | Complete the diagnostic exercise, read the source cards, freeze the preliminary protocol, prepare the pilot. |
| 2 | Validate metrics and raw repeatability; verify dense configuration and resource budget. |
| 3 | Execute the declared main study; inspect paired deltas, activation and failure cases. |
| 4 | Reproduce from a clean environment, fill the article with evidence, review and publish. |

## Troubleshooting

| Symptom | Check |
|---|---|
| Arabic Norm does not load | Both UI and the updated eval-api must run; confirm port 3005 and the Vite proxy. |
| CLI import absent in UI | Ensure API `EVAL_LAB_DIR` equals the parent of CLI `--root`; refresh the page; host and Docker filesystems differ. |
| Browser import rejected | Choose the JSON bundle, not a JSONL/qrels file; use CLI for large datasets. |
| Corpus IDs or qrels rejected | Preserve string IDs including `#`; keep every judged passage in the declared corpus; inspect the validation message. |
| All variants look identical | Inspect activation counts, exact text, tokenizer behavior and ranking ties before concluding no effect. |
| Zero raw lexical hits | This can be a legitimate mismatch; inspect text before assuming a metric bug. |
| Dense package missing | Rerun the relevant dense Make command; it installs the `rerank` dependency in eval-api before starting. |
| Dense model fails | Confirm exact model revision, local/network access, free disk and available memory; reduce batch size consistently if needed. |
| MIRACL transfer interrupted | Keep the incomplete folder for diagnosis; retry into a new directory. Never remove the marker to force import. |
| Huge run takes too long | Measure a pilot first; retain full corpus with fewer predefined queries if query scoring dominates; disclose any new scope. |
