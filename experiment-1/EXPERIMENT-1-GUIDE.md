# Experiment 1 — Learn evaluation with the Eval Lab

A hands-on guide for a beginner. You build one small dataset, measure a retrieval system with it, run one honest experiment, and learn the statistics behind every number.

- **Dataset you build:** `Kestrel HR Policies (en)` — 16 HR policy documents of a made-up company, 40 questions.
- **Tabs you use:** Golden Sets (all six steps, including **Gate → Power**), Retrieval, Experiments, Stats Lab, Learning Path.
- **Time:** about 3–4 hours. You can stop after any step; everything is saved.
- **Language:** English only for now.

> **How to use this guide.** Do the steps in order. Each step has four parts: **Do** (what to click and type), **Fields** (what each field means and the value to use), **Expect** (what you should see), and **Learn** (the idea behind it). Your numbers can differ a little from the numbers here. That is normal — the retriever and your own judgements add small changes.

---

## Contents

0. [Before you start](#0-before-you-start)
1. [The data in this folder](#1-the-data-in-this-folder)
2. [Words you need](#2-words-you-need)
3. **Part A — Golden Sets:** [Create](#step-1--create-the-dataset) · [Collect](#step-2--collect-upload-the-documents) · [Ask](#step-3--ask-write-two-questions-by-hand) · [Import](#step-4--ask-import-the-other-38-questions) · [Candidates](#step-5--ask-generate-and-review-synthetic-questions) · [Chunking](#step-6--index-look-at-the-chunks) · [Index](#step-7--index-index-the-corpus) · [Pool](#step-8--judge-pool-and-judge) · [Audit](#step-9--judge-audit-your-labels-blind) · [Checks](#step-10--gate-checks) · [Split](#step-11--gate-split) · [**Power**](#step-12--gate-power) · [Coverage](#step-13--gate-coverage) · [Freeze](#step-14--freeze-lock-version-1)
4. **Part B — Retrieval:** [Probe](#step-15--retrieval-probe) · [Score](#step-16--retrieval-score-a-split) · [Summary / Sweeps / Failures](#step-17--read-a-run-summary-sweeps-failures) · [Compare](#step-18--retrieval-compare-two-runs)
5. **Part C — Experiments:** [Design](#step-19--experiments-design-and-register) · [Dev](#step-20--experiments-run-on-dev) · [Holdout](#step-21--experiments-run-on-the-holdout-once) · [Conclude](#step-22--experiments-conclude)
6. **Part D — Stats Lab:** [Bootstrap](#step-23--stats-lab-bootstrap) · [CI coverage](#step-24--stats-lab-ci-coverage) · [CLT](#step-25--stats-lab-clt) · [Power](#step-26--stats-lab-power) · [P-hacking](#step-27--stats-lab-p-hacking)
7. **Part E — Versions:** [Fork and diff](#step-28--freeze-fork-change-diff)
8. **Part F — Learning Path:** [Check your missions](#step-29--learning-path)
9. [Self-test](#self-test) · [Troubleshooting](#troubleshooting) · [Start again](#start-again)

---

## 0. Before you start

1. Start the eval stack. From the repo root:

   ```bash
   cd infra && make evals-lab
   ```

   Wait for `eval-api ready`. The first run downloads about 5 GB of models.

2. Start the UI in a second terminal:

   ```bash
   cd ui && npm run dev
   ```

3. Open `http://localhost:5173/evaluation`. You do not need to log in on this stack.

4. Keep this folder open in your file explorer. You upload files from `data/`.

> **Why a lean stack?** The eval stack runs only what measurement needs: Postgres, MongoDB, MinIO, Kafka, Ollama, know-hub and the eval-api. It has no Kong and no Keycloak. This keeps runs fast and repeatable.

---

## 1. The data in this folder

```
experiment-1/
├── EXPERIMENT-1-GUIDE.md          ← this guide
└── data/
    ├── corpus/                    ← 16 documents to upload (.md)
    ├── golden-items-import.jsonl  ← 38 questions to import (q-003 … q-040)
    └── golden-items-all.jsonl     ← all 40 questions (only if you skip Step 3)
```

### 1.1 The story

**Kestrel Components Ltd.** is a made-up electronics company. Its People, Finance, IT and Legal teams wrote 12 policies. An employee asks an HR assistant questions like *"Can I work from a beach in Spain for a month?"*. The assistant must first **retrieve** the right policy. You measure how well it does that.

### 1.2 The 16 documents

| doc_id | Role | Why it is in the corpus |
|---|---|---|
| `annual-leave-policy` | answer | Long (2 chunks). Holiday days, carry-over, sickness on holiday. |
| `sick-leave-policy` | answer | Reporting times, fit notes, sick pay. |
| `parental-leave-policy` | answer | Birth parent, partner leave, keeping-in-touch days. |
| `remote-work-policy` | answer | Long. Anchor days, working abroad, home office costs. |
| `expense-reimbursement-policy` | answer | **Current** version 4: meals 60 €, mileage 0.30 €/km, form FIN-17. |
| `business-travel-policy` | answer | Long. Class of travel, hotels, per diem 45 €. |
| `it-security-policy` | answer | Passwords, lost devices, phishing. |
| `gifts-and-hospitality-policy` | answer | Gift limits, vouchers, conflicts of interest. |
| `performance-review-process` | answer | Ratings, pay rises, appeals. |
| `onboarding-guide` | answer | First day, mandatory training, probation. |
| `benefits-overview` | answer | Long. Pension match, health cover, learning budget. |
| `whistleblowing-procedure` | answer | Anonymous reports, response times. |
| `expense-policy-2021-archived` | **distractor** | Old version 3. Different numbers (meals 45 €, mileage 0.25 €). A retriever that picks it gives a wrong answer. |
| `contractor-handbook` | **distractor** | Same topics (leave, expenses) but for contractors, not employees. |
| `office-facilities-guide` | **distractor** | Mentions "anchor days" but does not say which days. |
| `company-newsletter-2026-q2` | **distractor** | Mentions benefits and policy updates, but gives no rules. |

A **distractor** is a document that is close to the topic but answers no question. Without distractors, ranking is too easy and every retriever looks good.

### 1.3 The 40 questions

- 38 **answerable** questions and 2 **unanswerable** ones (`q-039`, `q-040`).
- **Difficulty** (answerable): 11 easy · 14 medium · 13 hard.
- **Trap** (answerable): 13 none · 11 synonym · 8 exact-term · 6 multi-doc.
- Every answerable question has at least one **evidence span**.
- Some questions mark a document as **judged not relevant** (grade 0) — for example the archived expense policy.

Every bucket has 5 or more questions. The lab warns you when a bucket has fewer than 5, because a mean over 3 questions moves by 33 points when one question changes.

### 1.4 The traps — what each one tests

| Trap | Meaning | Example | What it tests |
|---|---|---|---|
| `none` | Control. The question uses words from the document. | *"How many weeks at full salary does the birth parent receive?"* | The baseline. If these fail, something basic is broken. |
| `synonym` | The user uses different words from the document. | *"Can I work from a beach in Spain for a month?"* — the document says *"outside your country of employment … 20 working days"*. | Meaning-based (vector) search. Word matching fails here. |
| `exact-term` | The answer depends on an exact code or number. | *"Which form do I fill in after losing a receipt?"* → `FIN-17`. | Word (full-text) search. Vectors often blur codes and numbers. |
| `multi-doc` | The full answer needs two documents. | *"What's the daily allowance on a business trip and does it include food?"* → travel policy **and** expense policy. | Recall of **all** relevant documents, not only the first one. |
| `unanswerable` | No document answers it. | *"What is the company policy on bringing dogs to the office?"* | Abstention in generation. Retrieval metrics skip these items. |

### 1.5 The grading rubric (use it every time)

| Grade | Meaning | Example for *"What is the mileage rate?"* |
|---|---|---|
| **3** | Fully answers the question. | `expense-reimbursement-policy` (0.30 €/km). |
| **2** | Substantially helps; part of the answer. | — |
| **1** | Related, but not enough to answer. | `business-travel-policy` (travel, but no mileage rate). |
| **0** | Not relevant — or wrong/out of date. | `expense-policy-2021-archived` (old rate 0.25 €). |

> **Rule for distractors:** always grade the four distractors **0**. The archived policy looks relevant but gives the wrong answer. A document that gives a wrong answer is not relevant.

---

## 2. Words you need

| Word | Plain meaning |
|---|---|
| **Corpus** | All the documents the system can search. |
| **Golden item** | One test question plus the correct documents for it. "Golden" means you trust it as the correct answer. |
| **Golden set / dataset** | The corpus plus all golden items, with a version number. |
| **Relevance grade** | A number 0–3 that says how well a document answers a question. |
| **Evidence span** | A sentence copied **word for word** from a document that answers the question. |
| **Judged not relevant** | You looked at a document and decided it does not answer the question (grade 0). A document you never looked at is **unjudged**. |
| **Index** | The searchable copy of the corpus inside know-hub (text is split into chunks and turned into vectors). |
| **Chunk** | A piece of a document (here: up to 512 tokens, about 384 words). |
| **Retriever / variant** | One exact search setup: level, hybrid on/off, thresholds and so on. |
| **Run** | One variant scored over many questions. |
| **k** | How many top results you look at (here usually 5). |
| **Dev split** | Questions you may look at as often as you like while you improve the system. |
| **Holdout split** | Questions you keep aside. You look at them once, at the end, to get an honest number. |
| **Freeze** | Lock a dataset version. Nothing can change it. It gets a content **hash** (a fingerprint). |
| **Confidence interval (CI)** | A range that probably contains the true value. |
| **Statistical power** | The chance that a test detects a real improvement. |

---

# Part A — Golden Sets: build the dataset

Sidebar → **Evaluation → Golden Sets**. The page shows a corridor with six steps: **Collect → Ask → Index → Judge → Gate → Freeze**. You follow it left to right.

## Step 1 — Create the dataset

**Do**
1. Click **New dataset**.
2. Fill in the dialog and click **Create**.

**Fields**

| Field | Value | Why |
|---|---|---|
| Name | `Kestrel HR Policies (en)` | A human name. The lab turns it into the id `kestrel-hr-policies-en`. |
| Language | `English` | Sets the default `lang` of documents and questions. |

**Expect** — The dataset opens at **v1 · draft**. The corridor shows step 1, Collect.

**Learn** — A dataset has **versions**. v1 is a *draft*: you can change it. Later you *freeze* it. You never edit a frozen version; you *fork* a new one.

---

## Step 2 — Collect: upload the documents

**Do**
1. Open **Collect**.
2. Drag all 16 files from `data/corpus/` onto the drop zone (or click **Choose files**).
3. Click one row, for example `expense-policy-2021-archived`, to open it. Read it. Click **List** to go back.

**Fields** (when you open a document)

| Field | Value | Meaning |
|---|---|---|
| doc_id | taken from the file name, e.g. `annual-leave-policy` | The stable id. Questions point to documents by this id. Letters, digits, `.`, `_`, `-` only. |
| lang | `en` | Language of the text. |
| Source (provenance) | the file name | Where the text came from. Good practice for audits. |
| Body | the document text | The text that gets indexed. |

**Expect** — `16 documents`. Every row shows `distractor` for now, because no questions exist yet. After Step 4 the 12 answer documents show `answers N`.

**Learn** — **Corpus vs dataset.** The index depends only on the corpus. Change a question → no reindex. Change a document → the index becomes *stale* and must be rebuilt.

---

## Step 3 — Ask: write two questions by hand

You type two questions yourself so that you learn every field. The other 38 are imported in Step 4.

**Do**
1. Open **Ask → Questions**. Click **+ Item**. The lab proposes the id `q-001`.
2. Fill in the fields from the table below. Click **Save**.
3. Click **List**, then **+ Item** again for `q-002`.

### q-001 — a simple question

| Field | Value | What it means |
|---|---|---|
| id | `q-001` | Unique id of the question. |
| lang | `en` | Language. |
| Query | `How many days of paid holiday do full-time staff get each year?` | The question, written the way a real employee asks it. Do **not** copy a sentence from the document — that tests string matching, not understanding. |
| Relevant documents (graded) | choose `annual-leave-policy` → **Add** → grade `3 · fully answers` | The correct document and how well it answers. |
| Judged NOT relevant (grade 0) | choose `contractor-handbook` → **Mark 0** | You checked it: it talks about leave but not for employees. Now it is a *judged* 0, not an *assumed* 0. |
| Evidence spans (verbatim) | **+ Span** → doc `annual-leave-policy` → text `Full-time employees receive 25 days of paid annual leave per calendar year` | The exact sentence that answers. A green **found** badge must appear. |
| Difficulty | `easy` | How hard the question is for a retriever. |
| Trap | `none` | The kind of difficulty. `none` = control question. |
| Tags | `leave` | Free labels to filter and group results later. |
| Ground truth answer | `25 days per calendar year plus public holidays; 28 days after five years of service.` | The correct answer in words. Retrieval metrics ignore it; generation evaluation (RAGAS) uses it. |
| Notes | `Manual practice item 1.` | Anything useful for other people. |

### q-002 — two documents, two grades, and a trap

| Field | Value |
|---|---|
| id | `q-002` |
| Query | `What is the daily meal limit when I'm travelling for work?` |
| Relevant documents | `expense-reimbursement-policy` → grade **3**; `business-travel-policy` → grade **1** (it talks about travel and sends you to the expense policy, but does not give the number) |
| Judged NOT relevant | `expense-policy-2021-archived` (it says 45 € — the old, wrong limit) |
| Evidence span | doc `expense-reimbursement-policy`, text `meals are reimbursed up to a daily limit of 60 euros` |
| Difficulty | `medium` |
| Trap | `exact-term` |
| Tags | `expenses, travel` |
| Ground truth | `60 euros per day (current policy, version 4). The archived 2021 policy said 45 euros; it no longer applies.` |
| Notes | `Manual practice item 2. The archived policy is a trap: it quotes an old limit.` |

**Expect** — The list shows 2 items. In the `judgements` column, q-002 shows `expense-reimbursement-policy·3`, `business-travel-policy·1`, `✕ 1` and a green `1 ev`.

**Try it (1 minute)** — Open q-002. Change one word in the evidence text, for example `60` → `sixty`. The badge turns red: **not in doc**. Change it back. An evidence span must be copied, never paraphrased.

**Learn**
- **Graded relevance** lets graded NDCG see the difference between "answers it" (3) and "mentions it" (1).
- **Evidence spans** keep chunk-level scoring correct even if you change the chunk size later: a chunk counts as relevant if it contains the span.
- **Unjudged is not irrelevant.** Any document you never graded is only *assumed* to be 0.

---

## Step 4 — Ask: import the other 38 questions

**Do**
1. In **Ask → Questions**, click **JSONL**.
2. Open `data/golden-items-import.jsonl` in a text editor. Copy **all** lines. Paste them into the box.
3. Choose **append**. Click **Import**.

> If you skipped Step 3, paste `golden-items-all.jsonl` instead and choose **replace all**. Do not append `-all` on top of your manual items — duplicate ids are refused.

**Fields**

| Control | Value | Meaning |
|---|---|---|
| append / replace all | `append` | Append keeps q-001 and q-002. Replace all deletes every existing item first. |

**Expect** — `40 items · 0 dev · 0 holdout · 40 unassigned`. Go back to **Collect**: the 12 answer documents show `answers N`; the 4 distractors still show `distractor`.

**Look at these rows** — they show each trap in practice:
- `q-012` *Can I work from a beach in Spain for a month?* — `synonym`, hard.
- `q-015` *Which form do I fill in after losing a receipt?* — `exact-term`, easy.
- `q-038` *What's the daily allowance on a business trip and does it include food?* — `multi-doc`, hard, grades 3 and 2, three evidence spans.
- `q-039` — `unanswerable` (no relevant document). It shows a yellow `unanswerable` badge.

**Learn** — One JSON object per line is the standard way to move eval data. The lab also accepts the older harness format (`relevant_doc_ids`, `holdout`).

---

## Step 5 — Ask: generate and review synthetic questions

A local LLM (llama3.1 8B in Ollama) writes candidate questions. You decide which ones to keep. No document text leaves your machine.

**Do**
1. Open **Ask → Candidates**.
2. Fill in the panel **Generate candidate questions** and click **Generate**.
3. Wait for the job to finish (1–3 minutes on CPU).
4. Read the **Review queue** numbers: *Pending*, *Accepted*, *Evidence found*, *Mean overlap*, and the overlap histogram.
5. Below the queue, each candidate has a card. Read its badges: `overlap N%`, `copied verbatim`, `evidence found` / `evidence NOT in doc`.
6. **Accept one** good candidate. **Reject one** bad candidate.

**Fields**

| Field | Value | Meaning |
|---|---|---|
| Style | `user-like` | How the LLM phrases questions. `keywords` = search-box style. `hard paraphrase` = few shared words (harder). |
| Per document | `2` | Candidates per document. |
| Temperature | `0.7` | Randomness. Higher = more varied, less reliable. |
| Seed | `42` | Same seed + same settings → same candidates. |
| Documents | `remote-work-policy`, `business-travel-policy` | Generate only from these two. (They land on the dev side of the split in Step 11, so the synthetic item cannot leak into the holdout.) |

**Card controls**

| Control | Value | Meaning |
|---|---|---|
| Question text box | edit if needed | You may fix the wording before you accept. The lab records `edited=true` in the item notes. |
| Split | `dev` | Where the accepted item goes. The lab **pins** it there, so a later split cannot move it. Never choose `holdout ⚠`. |
| **Accept** | click on the good candidate | Creates item `syn-001` (grade 3 on the source document, evidence if found). |
| Reject reason | choose one, e.g. `leaky: copies the document wording` | The reasons become a record of how your generator fails. |
| **Reject** | click on the bad candidate | Removes it from the queue. |

**How to choose**

| Accept a candidate when … | Reject a candidate when … (reason to pick) |
|---|---|
| a real employee could ask it | it copies the document (overlap badge red, or `copied verbatim`) → `leaky: copies the document wording` |
| its evidence badge is **evidence found** | the badge says `evidence NOT in doc` → `evidence not in document` |
| the answer is in one clear place | it is vague → `ambiguous question`; or it reads oddly → `unnatural phrasing` |

**Expect** — A new item `syn-001` with the tag `synthetic` and a yellow `synthetic` badge in the Questions list.

**Learn** — Generated questions echo their source text. They are easy for word search and never multi-hop. Use them to **grow dev**. Keep the **holdout human-written**, so the number you quote reflects real users.

---

## Step 6 — Index: look at the chunks

**Do**
1. Open **Index → Chunking**.
2. Choose document `annual-leave-policy`. Keep **Max tokens 512**, **Overlap 64**.
3. Read the two boxes: **Chunks** `2` and **Evidence spans** `… · 0 split · 0 missing`. The span of `q-004` (*can reclassify those days as sick leave*) sits in the second chunk; the spans of `q-001` and `q-003` sit in the first.
4. Now choose document `sick-leave-policy`. Set **Max tokens** `128` and **Overlap** `0`.
5. **Evidence spans** now shows `1 split`: the fit-note sentence of `q-006` is cut across two chunks. No single chunk contains it.
6. Move **Overlap** up to `64`. The split span is whole again — overlap exists for exactly this reason.
7. Click **Reset 512/64**.

**Fields**

| Field | Value | Meaning |
|---|---|---|
| Document | `annual-leave-policy`, then `sick-leave-policy` | The document to preview. |
| Max tokens | `512` (then `128`) | Largest chunk size. 512 is what know-hub really uses (about 384 words). |
| Overlap | `64` (then `0`) | Tokens repeated between neighbour chunks, so a sentence at a border is less likely to be cut. |

**Expect** — 2 chunks and 0 split spans at 512/64. At 128/0 on the sick leave policy, 1 split span (q-006).

**Learn** — If an evidence span is cut across two chunks, no single chunk contains it. Chunk-level recall for that question can never reach 1.0. That is a chunking problem, not a retrieval problem.

---

## Step 7 — Index: index the corpus

**Do**
1. Open **Index → Index**.
2. In the card **Index · off**, click **Index corpus**.
3. Wait until the card shows **current** and `INDEXED 16`.

**Fields**

| Card | Use it? | Meaning |
|---|---|---|
| Index · off | **Yes** | The corpus as written. |
| Index · conservative | No | Arabic spelling normalisation at ingestion. Not needed for English. |
| Index · aggressive | No | Stronger Arabic normalisation. Not needed for English. |

**Expect** — `current`, `INDEXED 16`. If some documents stay `PENDING`, click **Resume** after a minute (Ollama may still be loading).

**Learn** — Embedding runs in the background. If you score before every document is `INDEXED`, you get zeros that look like retrieval failures but are only timing problems. Each corpus hash gets its own isolated index, so old and new versions never mix.

---

## Step 8 — Judge: pool and judge

Your labels cover only the documents *you* thought of. **Pooling** asks three different retrievers for their top results and puts the union in front of you to judge.

**Do**
1. Open **Judge → Pool**.
2. Set **Depth** to `10`. Click **Build pool**. Wait for the job.
3. Read the four boxes: **Judged**, **Items fully judged**, **Relevant in pool**, **Relevant never pooled**.
4. Judge these **5 items by hand**. Click each one in the queue and give a grade to every *unjudged* (yellow) document:

   | Item | Grade these documents | Reason |
   |---|---|---|
   | `q-011` Which weekdays … office? | `office-facilities-guide` **0**, `company-newsletter-2026-q2` **0**, all others **0** | Only the remote-work policy names the days. |
   | `q-016` rate per km … own car? | `expense-policy-2021-archived` **0**, `business-travel-policy` **0**, others **0** | The archived rate (0.25 €) is out of date. |
   | `q-019` hotel budget … Zurich? | `contractor-handbook` **0**, `expense-reimbursement-policy` **0**, others **0** | Only the travel policy gives hotel limits. |
   | `q-036` change pension … buy extra holiday? | `company-newsletter-2026-q2` **0**, others **0** | The newsletter mentions the benefits window but gives no rule. |
   | `q-038` daily allowance … food? | `expense-policy-2021-archived` **0**, others **0** | Trap: the old meal limit is also 45 €. |

   If a document you did not expect really answers the question, grade it honestly (1–3). That is the point of pooling.
5. Click **Simulate Labeling** once. It marks the remaining unjudged pooled documents as **0**, so you do not need to click 400 pairs. Your gold grades stay.

**Fields**

| Field | Value | Meaning |
|---|---|---|
| Depth (top-k per retriever) | `10` | How many results each retriever adds to the pool. Deeper = more complete, more work. |

The three retrievers are: hybrid (vector + full-text), vector-only, and chunk-level vector. Their differences find documents the others miss.

**Expect** — **Judged** near 100 % after Simulate Labeling. **Relevant never pooled** should be 0 or very small.

**Learn** — A retriever that finds a relevant document you never labelled is *punished for being right*. Pooling (the method TREC uses) reduces this bias. Runs later report **judged@k** — the share of top results that rest on real judgements.

---

## Step 9 — Judge: audit your labels blind

Two careful people often disagree on a grade. If your labels are noisy, a small metric difference between two systems means nothing.

**Do**
1. Open **Judge → Audit**. Click **New audit batch**.
2. Fill in the dialog and click **Start batch**.
3. For each item you see the question and shuffled documents, **without** the existing grades. Grade every document with the rubric in §1.5. Click **Submit & next**.
4. After 10 items, read the agreement results.
5. In **Disagreements**, look at the biggest ones. If gold is wrong, click **adopt annotator-2's grade**.

**Fields**

| Field | Value | Meaning |
|---|---|---|
| Annotator name | `annotator-2` | The name of the second rater (you). Gold is rater 1. |
| Batch size | `10` | Items in this session. The mission needs ≥ 10 real items. |

> Do **not** use **Simulate Auditing** for the mission. It fakes a second rater from gold. Its numbers only show the simulation's noise.

**Read these numbers**

| Number | Meaning | Good value |
|---|---|---|
| % agreement | Share of (question, document) pairs where both raters gave the same grade. | High, but see κ. |
| Cohen's κ (binary) | Agreement on relevant / not relevant, **minus the agreement you get by chance**. | ≥ 0.6 |
| Weighted κ (0–3) | Like κ but a 3-vs-2 disagreement costs less than a 3-vs-0. | ≥ 0.6 |
| Confusion grid | Rows = gold grade, columns = your grade. Off-diagonal cells are disagreements. | Most counts on the diagonal. |

**Learn** — Most documents are *not* relevant, so two raters who both say "0" agree a lot by luck. κ removes that luck. Below about 0.6, tighten your rubric before you trust close comparisons.

---

## Step 10 — Gate: checks

The Gate asks one question: *can this dataset support a claim?*

**Do** — Open **Gate → Checks**. Click any check to open it. Read its detail, its affected items, and the note **why it matters**.

**Severities**

| Severity | Meaning |
|---|---|
| **blocker** | The number would be *wrong*. You must fix it before you register an experiment. |
| **warning** | The number is valid but has a caveat. |
| **info** | Something to know. |
| **pass** | The check is fine. |

**Expect (before the split)**
- **blocker:** *Split not assigned* — you fix this in Step 11.
- **warning:** *Dataset is a mutable draft* — you fix this in Step 14.
- **info:** *Unanswerable questions* (2), *Precision@5 is capped* (most questions have 1–2 relevant documents), *1 synthetic question*, and *Label agreement with 'annotator-2'* with your weighted κ (it turns into a warning if κ is below 0.6).
- **pass:** *Every label resolves*, *Distractors present* (4 of 16), *Index (off) current*, *Pool … judged*.

**Try it (optional, 2 minutes)** — Break something on purpose, then fix it:
1. Open `q-003` in Ask → Questions. Change its evidence to `Leave carried over must be used by March` (a paraphrase). Save.
2. Back in Gate → Checks: a **blocker** *Evidence spans not found in their documents* appears.
3. Put back the original text `Carried-over days must be used by 31 March`. Save. The blocker disappears.

**Learn** — The Gate turns "data quality" into a checklist. Each check names a way that a number can lie: leakage, missing labels, thin buckets, stale index.

---

## Step 11 — Gate: split

**Do**
1. Open **Gate → Split**.
2. Set the fields below. Click **Assign split**.

**Fields**

| Field | Value | Meaning |
|---|---|---|
| Holdout fraction | `30%` | Share of questions kept aside for the final test. |
| Seed | `20260915` | Makes the split repeatable: same seed + same items → same split on any machine. |
| Stratify by | `difficulty` | Split easy, medium and hard separately, so both sides get the same mix. |
| Group by | `primary document (no leakage)` | Keep **all questions about one document on the same side**. |

**Expect** — about **26 dev · 14 holdout**, with **13 answerable** holdout questions and **0 straddling** documents. (The `primary_doc` of a question is its best-graded relevant document.) Go back to **Checks**: the blocker is gone. The verdict becomes *ready with caveats*.

**Learn — why group by document?** Suppose q-005 (dev) and q-006 (holdout) both ask about the sick leave policy. You tune the system until q-005 works. That also helps q-006. Your holdout score is now too good. This is **leakage**. Grouping by document prevents it.

---

## Step 12 — Gate: Power

**Question this tab answers:** *With the questions I have, how big must an improvement be before I can see it?*

**Do** — Open **Gate → Power**. Change the sliders as in the exercises below and write down the three results.

### The fields

| Field | Default | Meaning |
|---|---|---|
| **Per-query score sd** | `0.35` | How much the score changes from one question to the next (standard deviation). 0.35 is a placeholder until you have a real run. More spread → harder to see a change. |
| **Target effect** | `0.05` | The smallest improvement worth acting on (here: +0.05 recall). You choose it **before** you see results. |
| **Confidence** | `95%` | How often the method's interval contains the truth in the long run. Not a probability about one result. |
| **Power** | `80%` | The chance you detect the target effect **if it is real**. 80 % means you miss a real effect 1 time in 5. |

### The three results

| Result | Meaning |
|---|---|
| **Questions needed** | How many questions you need to detect the target effect. |
| **Dev detects ±…** | The smallest effect your dev split (n ≈ 25) can detect. |
| **Holdout detects ±…** | The smallest effect your holdout split (n ≈ 13) can detect. This decides your experiment. |

The formula behind all three:

```
questions needed          = ((z_confidence + z_power) × sd ÷ target effect)²
smallest detectable effect = (z_confidence + z_power) × sd ÷ √n
```

With 95 % confidence and 80 % power, `z_confidence + z_power ≈ 1.96 + 0.84 = 2.80`.

### Exercises — fill in the last column yourself

| # | Set | Expected result | What it shows |
|---|---|---|---|
| 1 | Defaults (sd 0.35, effect 0.05, 95 %, 80 %) | Questions needed **385** · Dev **±0.20** · Holdout **±0.27** | Your 13-question holdout can only see very large changes. |
| 2 | Target effect **0.10** | Questions needed **97** | Double the effect → need about ¼ of the questions. |
| 3 | Target effect **0.27** | Questions needed **14** | This is about your holdout size — it matches "Holdout detects ±0.27". |
| 4 | Back to 0.05. sd **0.25** | Holdout **±0.19** | Less noisy scores → you see smaller effects. |
| 5 | sd **0.45** | Holdout **±0.35** | More noisy scores → you see only big effects. |
| 6 | sd 0.35. Power **90%** | Holdout **±0.32** | Being more sure to catch an effect costs precision. |
| 7 | Power 80 %. Confidence **99%** | Holdout **±0.33** | Fewer false alarms also costs precision. |
| 8 | After Step 16: click **use recall@k sd … from run** | Your real sd replaces 0.35 | Always prefer the spread from real data. |

The **power curve** chart shows the smallest detectable effect for every n. It falls with **√n**: 4 × the questions → only ½ the detectable effect.

**Learn — the most important sentence in this guide:** if your holdout can only detect ±0.27 and the true improvement is +0.05, the experiment will usually say "not significant". That means **"too small to see"**, not **"no effect"**. Decide this before you run anything.

---

## Step 13 — Gate: coverage

**Do** — Open **Gate → Coverage**.

**Expect** — Four bar groups: `difficulty`, `trap`, `lang`, `|relevant|` (how many relevant documents each question has). Every difficulty and trap bucket has 5 or more questions, so no bar is amber. `|relevant|` shows mostly 1, some 2.

**Learn** — **Stratified reporting** (results per bucket) is how you find *where* a system fails, for example "synonym questions fail". A bucket with fewer than 5 questions is an anecdote, not a measurement.

---

## Step 14 — Freeze: lock version 1

**Do**
1. Open **Freeze → Lifecycle**.
2. Read the four boxes: **Items** `41` (`1 synthetic`), **Documents** `16` (with the count of explicit 0-judgements), **Content hash**, **Created**.
3. Click **Freeze v1**.
4. (Optional) Click **Export golden JSONL** to save a copy of the frozen questions.

**Expect** — The status pill shows **frozen**; a **frozen hash** line appears. Gate → Checks now shows *Frozen content intact* (pass) and **zero blockers**. The Split tab now says *Frozen — the split is part of the content hash*.

**Learn** — A result is a pair: *(system, dataset hash)*. If the dataset can still change, a number has no stable meaning. After freezing, "v1" always means the same 41 questions and 16 documents.

---

# Part B — Retrieval: measure one system

Sidebar → **Evaluation → Retrieval**. In the dataset picker at the top, choose `kestrel-hr-policies-en` **v1**. The page has three tabs: **Probe**, **Score**, **Compare**.

## Step 15 — Retrieval: Probe

**Probe** means: run one query and look closely at how every metric is computed.

**Do**
1. Open the setup fold (the variant editor) and check the values below.
2. In **Golden item**, choose `q-001`. The query fills in and runs.
3. Read **Results** (the ranked documents with grade chips) and **How the numbers were computed**.
4. Repeat with `q-012` (synonym, hard) and `q-038` (multi-doc, hard).
5. Choose **free query — judge results yourself**. Type `Can I claim a parking fine on expenses?`. Click **Run**. Click the grade chips on the results to judge them (click again to change the grade). Watch the metrics change.

**Fields — the setup (variant)**

| Field | Value now | Meaning |
|---|---|---|
| Name | `probe` | A label for this setup. It also becomes the name of runs you start from here. |
| Level | `document` | `document`: one result per document. `chunk`: one result per chunk, scored with evidence spans. |
| Ranking · Hybrid (RRF) | **on** | Mix vector search (meaning) with full-text search (words) using Reciprocal Rank Fusion. Off = vector only. Only works at document level. |
| Corpus index | `off` | Which index copy to search. Only `off` is built. |
| Query normalisation | `off` | Arabic spelling clean-up of the query. Not needed for English. |
| Min similarity | `0` | Drop results below this score. 0 = keep all. |
| Fetch depth | `50` | Results to request. Rank metrics read up to max(k, 10). |
| k | `5` | Top results that count for recall@k and precision@k. |

**Read the metrics**

| Metric | Question it answers | Watch out |
|---|---|---|
| **Hit rate@k** | Is at least one relevant document in the top k? | Stops moving once one hit is found. |
| **Recall@k** | What share of *all* relevant documents is in the top k? | For multi-doc questions it shows missing documents. |
| **Precision@k** | What share of the top k is relevant? | Capped: with 1 relevant doc, the best possible P@5 is 0.20. |
| **MRR** | 1 ÷ rank of the first relevant result. | Ignores everything after the first hit. |
| **NDCG** | How close is the ranking to the perfect order? | Treats all relevant documents as equal. |
| **NDCG graded** | Same, but a grade-3 hit counts 7 × a grade-1 hit (gain = 2^grade − 1). | Only useful if your grades are consistent. |
| **AP / MAP** | Precision averaged at every relevant hit. | Rewards finding **all** relevant documents. |

**Expect**
- `q-001`: the leave policy at rank 1 → most metrics = 1.0, precision@5 = 0.20.
- `q-038`: two relevant documents with grades 3 and 2. If the expense policy (grade 2) comes before the travel policy (grade 3), **NDCG** and **NDCG graded** differ.
- Free query: a bar says *This query has ad-hoc judgements*. The **Add as golden item** button is disabled (tooltip: *Frozen dataset — fork to add judgements*). You add it properly in Step 28. **Reset** clears your ad-hoc grades.

**Learn** — The table under *How the numbers were computed* shows the math row by row. Rank 1 is worth 1.000, rank 10 only 0.289. Position matters a lot.

---

## Step 16 — Retrieval: score a split

**Do** — Score the whole dev split with two setups. They differ in **one** field: hybrid.

1. **Run A.** On the **Probe** tab set **Name** `vector-only` and switch **Hybrid (RRF)** **off**. Open the **Score** tab. Split: `dev (… answerable)`. Click **Run vector-only**. Wait for the job.
2. **Run B.** Back on **Probe**, set **Name** `hybrid` and switch **Hybrid** **on**. On **Score**, click **Run hybrid**.
3. (Optional) **Run C.** Name `chunk`, Level `chunk`. Run it too. Chunk level is vector-only by design.

**Expect** — Each run shows a result panel. Holdout questions are **not** in these runs: ad-hoc runs cannot see the holdout. Only a registered experiment can, and every look is logged.

**Learn** — A run is *exploratory*. It guides your next change. It does not prove anything yet.

---

## Step 17 — Read a run: Summary, Sweeps, Failures

Every run result has three tabs.

### Summary

| Item | Meaning |
|---|---|
| Metric table | The mean of each metric with a 95 % bootstrap interval. |
| `judged@5` | Share of top-5 results that carry a real judgement. After pooling, it should be high (green ≥ 80 %). |
| `unanswerable skipped` | `q-039`/`q-040` if they are on dev — not scored, not counted as 0. |
| Per-query distribution | Histogram of one metric over all questions. Scores pile up at 0 and 1 — this *bimodal* shape makes a plain mean and a t-interval misleading. |
| Per-query table | Every question and its score, worst first. Red = 0. |

### Sweeps

| Chart | What it shows | Mission |
|---|---|---|
| **Every metric at k = 1…20** (k-sweep) | Each metric recomputed at every k. Recall rises with k, precision falls, hit rate flattens early. | `rm-ksweep`: find the k where hit rate stops moving; explain which k you would pick. |
| **Dropping results below a similarity floor** (threshold sweep) | What happens when you add a minimum similarity. A low floor keeps noise; a high floor returns empty results (dashed line). | `rm-threshold`: find where empty results start; explain the two failure modes. |

### Failures

**Do**
1. Set **Failure = metric below** `Recall@k` and **Threshold** `1` (every question that missed at least one relevant document).
2. Read **Suggested causes (auto)**.
3. Open 5 failure cards. For each one, confirm or correct the cause tag. Use the table below.

| Cause tag | Meaning | Likely on which trap |
|---|---|---|
| Not retrieved | The relevant document is not in the list at all. | synonym |
| Ranked too low | Retrieved, but below k. | any |
| Distractor outranked | A document judged 0 won. | exact-term (archived policy) |
| Possible label gap | The top result was never judged. Judge it before blaming the retriever. | — |
| Vocabulary mismatch | The query shares few words with the answer document. | synonym |
| Partial multi-document | Only some of the needed documents were found. | multi-doc |

**Mission** `rm-failures`: tag ≥ 5 failed queries.

**Learn** — A mean says *that* a system fails. Only individual failures say *why* — and "why" is the only thing you can fix.

---

## Step 18 — Retrieval: compare two runs

**Do**
1. Open the **Compare** tab.
2. In **Run history**, choose **A** = the `vector-only` run and **B** = the `hybrid` run.
3. Click **Compare A → B**.

**Read the table**

| Column | Meaning |
|---|---|
| A, B | Mean score of each run. |
| Δ (B−A) | Paired difference: B minus A, computed per question, then averaged. |
| paired CI | The interval of that difference. If it crosses 0, the difference could be noise. |
| d_z | Effect size: mean Δ ÷ spread of Δ. |
| p (shuffle) | Chance of a difference this large if B were really the same as A. |
| W/L/T | Questions where B wins / loses / ties. Many ties are normal in retrieval. |

The **forest plot** draws each metric's Δ and interval next to a line at 0.

**Expect** — Hybrid often wins on `exact-term` questions (codes like FIN-17, numbers). Vector search often does as well on `synonym` questions. The overall Δ may be small, and its interval may cross 0.

**Learn** — This is still **exploratory**. You looked at dev many times and chose what to compare after seeing results. It is a lead, not a claim. The claim comes from Part C.

---

# Part C — Experiments: one honest test

Sidebar → **Evaluation → Experiments**. An experiment compares a **baseline** and a **treatment** that differ in **one** thing, on a **frozen** dataset, against success criteria you write **before** you see a result.

## Step 19 — Experiments: design and register

**Do**
1. Click **New**. The **Design** tab opens.
2. Fill in every field below.
3. Click **Save draft**. Read the **Pre-registration checklist**: every row must be green.
4. Click **Save & register**. The design is now locked.

### Hypothesis & success criteria

| Field | Value | Why |
|---|---|---|
| Name | `Hybrid vs vector-only on Kestrel HR` | Short, says what changes. |
| Dataset version | `kestrel-hr-policies-en · v1 · frozen` | Only frozen versions can be registered. |
| Hypothesis | `If we add full-text search to vector search (hybrid RRF), recall@5 on the Kestrel HR holdout will increase by at least 0.05, because exact-term questions (form codes, amounts, rates) are matched by words that embeddings blur.` | Pattern: *If we <change>, then <metric> will <direction> by at least <min effect>, because <reason>.* |
| Rationale / prior evidence | `Exploratory dev runs (Step 18) showed hybrid winning on exact-term items such as q-015 (FIN-17) and q-016 (0.30 per km).` | What you saw before. It explains the hypothesis. |
| Primary metric | `Recall@k` | **One** metric decides. Everything else is secondary. |
| k | `5` | The assistant reads the top 5 results. |
| Minimum effect | `0.05` | The smallest gain worth shipping. |
| Confidence level | `95%` | Standard choice. |
| Direction | `increase` | You expect recall to go up. (`non-inferior` would test "not worse by more than the margin".) |
| Secondary metrics (exploratory) | `MRR`, `NDCG` | Reported, never decisive. Each extra metric adds a chance of a false "significant". |
| Guardrails | **+ Guardrail** → `Precision@k`, max drop `0.05` | Recall must not win by flooding the top 5 with noise. |

### Arms

| Field | Baseline | Treatment |
|---|---|---|
| Name | `vector-only` | `hybrid` |
| Level | `document` | `document` |
| Hybrid (RRF) | **off** | **on** ← amber ring = the one change |
| Corpus index | `off` | `off` |
| Query normalisation | `off` | `off` |
| Min similarity | `0` | `0` |
| Fetch depth | `50` | `50` |

The **Arms** panel must show `varies: hybrid`. If it shows two fields, the design is *confounded*: you could not tell which change caused the result.

**If registration fails**, the message lists the problems:

| Message | Fix |
|---|---|
| The dataset version must be frozen | Freeze v1 (Step 14). |
| Write the hypothesis down | Fill in Hypothesis. |
| Baseline and treatment are identical | Change exactly one field. |
| Gate blocker: … | Fix it in Gate → Checks (on a new version if needed). |
| The holdout split has no answerable items | Assign the split (Step 11) before freezing. |

**Learn** — **Pre-registration** stops you from moving the goal after you see the data. Name, arms and criteria are fixed with a timestamp.

---

## Step 20 — Experiments: run on dev

**Do**
1. Open the **Report** tab → chapter **Score**.
2. Under **Dev split**, click **Run both arms on dev**. Wait.
3. Keep the split switch on **dev**. Open the chapters:
   - **Verdict** — the decision *if* this were the final data. It is labelled exploratory.
   - **Queries** — the per-question differences. This is where the mean comes from.
   - **Record** — power, guardrails, integrity (hash, holdout looks, all runs).

**Learn** — Dev results help you check the pipeline works. They do not decide. You may run dev again as often as you want.

---

## Step 21 — Experiments: run on the holdout, once

**Do**
1. In chapter **Score**, under **Holdout split**, switch on **I will not change the system in response to this result before concluding.**
2. Click **Evaluate on holdout**. The panel now shows `accessed 1×`.
3. Switch the report to **holdout**. Open **Verdict**.
4. Change the interval method between **bootstrap**, **t-dist** and **cluster**. Note the three intervals of the primary metric.

**Read the verdict**

| Verdict | Meaning |
|---|---|
| **Success** | The whole interval is above your minimum effect. |
| **Real but below threshold** | Clearly above 0, but not clearly above your minimum effect. |
| **No meaningful effect** | The interval sits between −min effect and +min effect. |
| **Inconclusive** | The interval crosses both 0 and the minimum effect. The data cannot decide. |
| **Regression** | Clearly worse. |
| **Guardrail breach** | The primary improved, but precision dropped by more than 0.05. |

**Expect** — Most likely **Inconclusive**. Your holdout has about 13 questions and can only detect about ±0.27 (Step 12). A true gain of +0.05 is too small to see with this set. This is the correct, honest result — and exactly what Power predicted.

**The three intervals**
- **bootstrap** — resamples questions.
- **t-dist** — assumes the differences are normal. Often too narrow on 0/1 scores.
- **cluster** — resamples *documents*. Questions about the same document fail together, so they are not independent. This interval is usually wider. It is the honest one to quote when questions share documents.

**If you must look again** (for example the first run failed), the lab asks *Why look again?* and keeps your reason in the record.

---

## Step 22 — Experiments: conclude

**Do**
1. With the report on **holdout**, find the panel **Conclude**.
2. Write a **Conclusion note**, for example: `Inconclusive at n=13 (detects ±0.27 only). Keep hybrid (no guardrail breach, no harm). Grow the holdout to ~100 questions before testing gains of 0.10.`
3. Click **Conclude experiment**.

**Expect** — Status **concluded**. The decision is frozen in the record. No more runs are allowed.

**Learn** — A decision is honest only when it comes from data you did not tune on, and when you write down what you will do *before* you move on.

---

# Part D — Stats Lab: see the statistics work

Sidebar → **Evaluation → Stats Lab**. Every simulation draws from a population whose **true mean is known**. That is the only setting where "did the interval contain the truth?" has an answer.

## Step 23 — Stats Lab: Bootstrap

**Do** — Tab **Bootstrap**. Run twice:

| Run | Population | Other fields |
|---|---|---|
| 1 | `real per-query scores from a run` → Run: your `hybrid` run · Metric: `Recall@k` | Resamples `2000` · Confidence `95%` |
| 2 | `Bernoulli (binary hit / miss)` · Success rate p `0.70` | Sample size n `20` · Resamples `2000` · Confidence `95%` |

**Look at** — the histogram of bootstrap means (lumpy, with gaps), and the three intervals in **Three intervals, same data**.

**Learn** — The bootstrap pretends your sample *is* the population, resamples it thousands of times, and records each mean. With 0/1 scores and small n, only a few different means are possible — so the histogram has gaps. **Mission** `st-bootstrap`: explain why.

## Step 24 — Stats Lab: CI coverage

**Do** — Tab **CI coverage**.

| Field | Value |
|---|---|
| Population | `bimodal (works / fails / in between)` · Share at 1.0 `0.70` · Share at 0.0 `0.20` |
| Sample size n | `10` (then run again with `100`) |
| Simulated samples | `300` |
| Bootstrap resamples | `500` |
| Confidence | `95%` |

**Look at** — **Actual coverage at n = 10** for each method, and **The first 60 intervals** (red ones miss the true mean).

**Learn** — "95 %" is a property of the *procedure*, and it can fail. At n = 10 one method covers far less than 95 %. At n = 100 all methods come close. **Mission** `st-coverage`.

## Step 25 — Stats Lab: CLT

**Do** — Tab **CLT**. Population `right-skewed (most near 0)`. **Sample sizes** `2, 5, 30, 100`. **Samples per n** `2000`. Run.

**Look at** — The skew of the sample means at each n, and `sd of means` vs `σ/√n`.

**Learn** — The **Central Limit Theorem** says averages become bell-shaped as n grows, even from a skewed population. Normal-theory intervals (t, z) are only as good as the CLT at your n. **Mission** `st-clt`.

## Step 26 — Stats Lab: Power

**Do** — Tab **Power**. Run three times:

| Field | Run 1 | Run 2 | Run 3 |
|---|---|---|---|
| Questions n | `13` (your holdout) | `50` | `230` |
| True effect (Δ recall) | `0.10` | `0.10` | `0.10` |
| Discordance (share of questions that change) | `0.30` | `0.30` | `0.30` |
| Simulated experiments | `300` | `300` | `300` |
| Bootstrap resamples | `500` | `500` | `500` |
| Confidence | `95%` | `95%` | `95%` |

**Expect**

| n | Empirical power | Analytic power | n for 80 % |
|---|---|---|---|
| 13 | ≈ 0.12 | ≈ 0.10 | ≈ 228 |
| 50 | ≈ 0.23 | ≈ 0.26 | ≈ 228 |
| 230 | ≈ 0.79 | ≈ 0.80 | ≈ 228 |

**Learn** — *Discordance* is the share of questions where the two systems give a different answer. Most questions do not change, so the real signal comes from a few. With your 13-question holdout, a real +0.10 improvement is detected only about 1 time in 10. **Mission** `st-power`: compare empirical and analytic power and explain any gap.

## Step 27 — Stats Lab: P-hacking

**Do** — Tab **P-hacking**. Every simulated experiment has a true effect of **exactly 0**.

| Field | Value |
|---|---|
| Questions n | `40` |
| Metrics tested | `7` |
| Correlation between metrics | `0.5` |
| Discordance | `0.3` |
| Peek every … questions | `5` |
| Simulated experiments | `500` |

**Look at**
- **Test 7 metrics, report any win** — the raw false-win rate is far above 5 %. **Holm** correction brings it back near 5 %.
- **Peek every 5 questions, stop when p < α** — stopping at the first "significant" look also inflates false wins.

**Learn** — This is why Part C used **one** primary metric, a **fixed** minimum effect, and **one** holdout look. These rules are structural defences, not good intentions. **Mission** `st-phacking`.

---

# Part E — Versions

## Step 28 — Freeze: fork, change, diff

**Do**
1. Golden Sets → your dataset → **Freeze → Lifecycle** (v1 selected). In **Changelog** write `v2: add a question about parking fines.` Click **Fork to v2**.
2. v2 opens as a draft. In **Ask → Questions**, click **+ Item** and add:

   | Field | Value |
   |---|---|
   | id | `q-041` |
   | Query | `Can I claim a parking fine on expenses?` |
   | Relevant | `expense-reimbursement-policy` → 3 |
   | Evidence | `fines and penalties, including parking fines` |
   | Difficulty / Trap | `easy` / `none` |
   | Tags | `expenses` |
   | Ground truth | `No. Fines and penalties, including parking fines, are never reimbursed.` |

3. Open **Freeze → Diff**. From `v1`, To `v2`. Read the result.

**Expect** — 1 item added, no documents changed, a **comparability verdict**. The new item has no split yet, so v2 needs a split before it can be frozen and registered.

**Learn** — Every change to the data changes what a number means. Results from v1 and v2 are comparable only as far as the diff allows. Your concluded experiment stays tied to v1's hash. **Mission** `dc-versions`.

---

# Part F — Learning Path

## Step 29 — Learning Path

**Do** — Sidebar → **Evaluation → Learning Path**. Missions marked *auto* are checked from the lab's data. Missions marked *reflection* need your own written explanation (at least 80 characters).

| Mission | Done in step | Type |
|---|---|---|
| dc-scale — dataset with real distractors | 2, 4 | auto |
| dc-graded — grades and verbatim evidence | 3, 4 | auto |
| dc-pool — pool three retrievers and judge | 8 | auto |
| dc-kappa — audit your labels blind | 9 | auto |
| dc-synthetic — generate and reject leaky questions | 5 | auto |
| dc-versions — freeze, fork, diff | 14, 28 | auto |
| ed-split — leakage-free split | 11 | auto |
| ed-ready — zero blockers on a frozen version | 10, 14 | auto |
| ed-register — one-variable experiment | 19 | auto |
| ed-conclude — holdout once, conclude | 21, 22 | auto |
| rm-graded — graded vs binary NDCG disagree | 15, 16 | auto |
| rm-ksweep — read a k-sweep | 17 | reflection |
| rm-threshold — the similarity-floor trade-off | 17 | reflection |
| rm-failures — tag five failures | 17 | auto |
| st-bootstrap · st-coverage · st-clt · st-power · st-phacking | 23–27 | reflection |
| st-cluster — explain a cluster interval | 21 | reflection |

**Reflection example** (st-cluster): *"The cluster CI was wider than the bootstrap CI because several holdout questions share the same answer document, so they fail together. I would quote the cluster interval because it does not pretend those questions are independent."*

---

## Self-test

Answer without looking. The answers follow.

1. Why does the corpus need distractors?
2. A retriever returns a relevant document that you never graded. How do the metrics score it, and what fixes this?
3. Why group the split by `primary_doc`?
4. Your holdout detects ±0.27. The experiment says *Inconclusive*. What does that mean?
5. Why only one primary metric?
6. Why is the cluster interval wider than the normal bootstrap interval?
7. Precision@5 is 0.20 for a perfect answer on `q-001`. Is the retriever bad?
8. Why must evidence spans be copied word for word?

<details>
<summary>Answers</summary>

1. Without them, a random ranking already does well and you cannot tell a good retriever from a lucky one.
2. As not relevant (0), by assumption. Pooling and judging the pool fixes it; judged@k measures how much it matters.
3. So questions about one document cannot sit on both sides. Otherwise tuning on dev also improves holdout, and the holdout score is too optimistic (leakage).
4. The interval crosses both 0 and the minimum effect. The test was too small to decide — not "there is no effect".
5. With many metrics, one crosses "significant" by chance (Step 27). One pre-registered primary metric keeps the false-win rate near 5 %.
6. Questions that share a document fail or succeed together. The cluster bootstrap resamples documents, so it counts fewer independent units and gives a wider, more honest range.
7. No. The question has one relevant document, so the best possible precision@5 is 1/5 = 0.20. Read precision next to recall.
8. A chunk counts as relevant only if it contains the span. A paraphrase is never found, so chunk recall would be 0 by construction.

</details>

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| Evaluation pages show an API error | eval-api is not running | `cd infra && make evals-lab` |
| *Index the corpus … before scoring* | Index `off` not current | Golden Sets → Index → **Index corpus** or **Resume** |
| Documents stay `PENDING` | Ollama still loading the embedding model | Wait 1–2 minutes, click **Resume** |
| Import says `duplicate item ids` | You appended a file that contains q-001/q-002 | Use `golden-items-import.jsonl` with **append**, or `-all` with **replace all** |
| Import says `invalid_jsonl` | A line was cut when copying | Copy the whole file again |
| Candidate generation fails | The LLM model is missing | Check `make ps`; `ollama-init` pulls the model on first start |
| *Ad-hoc runs cannot evaluate holdout items* | You tried to score the holdout from Retrieval | Use a registered experiment (Part C) |
| Cannot judge, audit or edit | The version is frozen | Fork a new version (Step 28) |
| Registration refused | See the table in Step 19 | Fix each listed problem |

## Start again

- **Soft reset:** create a second dataset with a new name, for example `Kestrel HR Policies (en) try 2`.
- **Full reset:** stop the lab (`cd infra && make evals-down`) and delete the lab store folder shown under `lab_dir` at `http://localhost:3005/api/eval/status`. This deletes **all** eval datasets, runs and experiments.

---

*Data: fictional company, written for this guide. Checked with the lab's own split and Gate code: after Step 11 the Gate shows zero blockers, 13 answerable holdout questions and no document on both sides of the split.*
