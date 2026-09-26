# Experiment 2 — Evaluate the answers, the judge, and the release

A hands-on guide that continues Experiment 1. There you measured **retrieval**: did the right document come back? Here you measure **generation**: is the answer right, and is it supported? Then you turn the tools on the measurer itself (the **LLM judge**), write a **release gate** that decides what may ship, and learn to find **which stage** of the pipeline caused a failure.

- **Dataset:** `kestrel-hr-policies-en` **v1 (frozen)** from Experiment 1 — 16 documents, 27 dev questions.
- **New tabs:** Evaluation → **Generation**, **Judge Lab**, **Release Gate**.
- **Time:** about 4–5 hours. You can stop after any step; everything is saved.
- **Hardware:** a GPU helps a lot. Every generation metric is several calls to a local 8B model (llama3.1 in Ollama). No text leaves your machine.

> **How to use this guide.** Same format as Experiment 1. Each step has **Do**, **Fields**, **Expect** and **Learn**. The numbers in *Expect* come from a real run of this lab on the Kestrel set with `llama3.1:8b-instruct-q4_K_M`. Your numbers will differ a little — the model is the same, but your own labels and small version differences move them. When they differ a lot, the guide tells you what that means.

---

## Contents

0. [Before you start](#0-before-you-start)
1. [The data in this folder](#1-the-data-in-this-folder)
2. [Words you need](#2-words-you-need)
3. **Part A — Generation metrics:** [The four texts](#step-1--the-four-texts) · [Control](#step-2--run-the-control) · [Stale context](#step-3--faithful-but-wrong) · [Hallucination](#step-4--one-invented-detail) · [Relevant but wrong](#step-5--relevant-but-wrong) · [Vacuous](#step-6--the-vacuous-10) · [Rank](#step-7--context-precision-rank-matters) · [Recall](#step-8--context-recall-and-the-reference) · [Judge errors](#step-9--when-the-metric-itself-is-wrong)
4. **Part B — Component-level evaluation:** [Baseline run](#step-10--run-the-pipeline) · [Read a trace](#step-11--read-one-trace) · [Components](#step-12--components-funnel-and-blame) · [Fault injection](#step-13--test-the-attribution-with-fault-injection) · [Fix and move the bottleneck](#step-14--fix-the-bottleneck-watch-it-move) · [Compare](#step-15--compare-two-runs)
5. **Part C — LLM-as-judge design:** [Import samples](#step-16--import-the-calibration-set) · [Label blind](#step-17--label-blind) · [Vague judge](#step-18--a-vague-judge) · [Validate](#step-19--validate-it-against-your-labels) · [Anchored judge](#step-20--an-anchored-reference-guided-judge) · [Position bias](#step-21--position-bias) · [Verbosity bias](#step-22--verbosity-bias) · [Self-consistency](#step-23--self-consistency) · [Drift](#step-24--judge-drift) · [Real answers](#step-25--validate-on-real-pipeline-answers)
6. **Part D — Regression gates:** [Read the policy](#step-26--read-the-policy) · [Baseline](#step-27--promote-a-baseline) · [Evaluate an improvement](#step-28--evaluate-a-real-improvement) · [Block a regression](#step-29--block-a-regression) · [Preconditions](#step-30--a-comparison-the-gate-refuses) · [Noise](#step-31--a-rule-that-cries-wolf) · [Override](#step-32--record-and-override) · [CI](#step-33--wire-it-into-ci) · [Your policy](#step-34--write-your-own-policy)
7. [Self-test](#self-test) · [Troubleshooting](#troubleshooting) · [Start again](#start-again)

---

## 0. Before you start

1. **Finish Experiment 1 first.** You need `kestrel-hr-policies-en` **v1, frozen, split, and indexed** (Golden Sets → Index shows `off · current · INDEXED 16`).

   > **Fast path** if you skipped it: Golden Sets → New dataset `Kestrel HR Policies (en)` → Collect: upload `../experiment-1/data/corpus/*.md` → Ask → JSONL: paste `../experiment-1/data/golden-items-all.jsonl`, *replace all* → Index corpus → Gate → Split (30 %, seed `20260915`, stratify `difficulty`, group by `primary document`) → Freeze v1. Your numbers will then match this guide less closely (no synthetic item, no pooled 0-judgements), but every step works.

2. Start the eval stack and the UI, as in Experiment 1:

   ```bash
   cd infra && make evals-lab        # wait for "eval-api ready"
   cd ui && npm run dev              # second terminal
   ```

3. Open `http://localhost:5173/evaluation/generation`. In the dataset picker (top right) choose **kestrel-hr-policies-en · v1**.

4. Keep this folder open. You paste files from `data/`.

> **Why so many LLM calls?** A generation metric is not one question to the model. Faithfulness splits the answer into claims and checks **each claim separately**. Context precision checks **each chunk separately**. That is slower, and it is on purpose: one call per decision means no decision can see (and be influenced by) another one.

---

## 1. The data in this folder

```
experiment-2/
├── EXPERIMENT-2-GUIDE.md           ← this guide
└── data/
    ├── judge-calibration.jsonl     ← 31 answers to judge, with the guide author's labels (rater "guide")
    └── rubrics/
        ├── rubric-v1-vague.txt     ← a bad rubric, on purpose
        └── rubric-v2-anchored.txt  ← an anchored, reference-guided rubric
```

### 1.1 The calibration set

31 answers to 11 Kestrel questions. Every question has 2–3 answers of different **kinds**. The kind is a design tag; the lab **hides it while you label**.

| Kind | Count | Example | Should pass? |
|---|---|---|---|
| `correct` | 10 | *"220 euros per night — Zurich counts as a high-cost city."* | yes |
| `verbose-correct` | 2 | a correct answer inside a long paragraph | yes |
| `correct-abstention` | 1 | *"None of the policies say anything about dogs…"* (the question is not covered) | yes |
| `wrong-number` | 3 | *"Use form FIN-12."* (it is FIN-17) | no |
| `stale` | 1 | *"The daily meal limit is 45 euros."* (the archived 2021 policy) | no |
| `contradiction` | 4 | *"An email to the shared HR inbox before 10:00 is enough."* | no |
| `partial` | 3 | *"45 euros per day."* (the question also asked about food) | no |
| `evasive` / `verbose-evasive` | 3 | *"Please contact the Finance team."* | no |
| `hallucination` / `fabrication` | 2 | *"…and an extra day off on your birthday."* | no |
| `plausible-wrong` | 1 | *"Gifts ≤ 50 € are allowed, so the voucher is fine."* (vouchers are never allowed) | no |
| `verbose-wrong` | 1 | a fluent, friendly paragraph with an invented hotel limit | no |

13 pass, 18 fail. The failing answers are built to look good in different ways: fluent, confident, long, or *almost* right. Those are the answers a weak judge lets through.

### 1.2 The two rubrics

| File | What it is |
|---|---|
| `rubric-v1-vague.txt` | *"Rate the quality of the answer from 1 to 5…"* — one line, no anchors, no reference. The rubric most people write first. |
| `rubric-v2-anchored.txt` | One criterion (correct and usable against the reference), an **anchor** for every score level, and explicit rules for length, tone, abstentions and outdated facts. |

---

## 2. Words you need

| Word | Plain meaning |
|---|---|
| **Question / context / answer / reference** | The four texts every generation metric reads: what was asked, what retrieval gave the model, what the model said, and the correct answer you wrote (the *ground truth*). |
| **Claim** | One fact in an answer that can be true or false on its own. *"25 days plus public holidays"* is two claims. |
| **Faithfulness** | Share of the answer's claims that the **context** supports. |
| **Answer relevance** | Does the answer address the **question**? (Not: is it true.) |
| **Context precision** | Are the useful chunks ranked **first**? Rank-weighted. |
| **Context recall** | Does the context contain everything the **reference** says? |
| **Answer correctness** | Does the answer agree with the **reference**? Here: F1 over claims. |
| **Answer verdict** | A binary judgement: same core answer as the reference, yes or no. The pipeline's `answer_ok`. |
| **Abstention** | The system says it cannot answer. Correct on an unanswerable question; a **false abstention** on an answerable one. |
| **Judge** | An LLM used as a measuring instrument: a rubric + a model + decoding settings + an output format. |
| **Rubric** | The judge's instructions. **Anchored** = every score level has a concrete description. |
| **Reference-guided** | The judge sees the reference answer, so it compares instead of guessing. |
| **Fingerprint** | A hash of everything that defines a judge. Same fingerprint = same instrument. |
| **Cohen's κ (kappa)** | Agreement beyond chance (Experiment 1, Step 9). 1 = perfect, 0 = chance. |
| **Fail recall** | Of the answers a human failed, the share the judge also failed. |
| **Position bias** | In an A-vs-B comparison, preferring whichever answer is shown first (or second). |
| **Verbosity bias** | Scoring a longer answer higher when the extra words add nothing. |
| **Drift** | The judge's scores change because the judge changed (model, prompt, settings), not because the answers changed. |
| **Stage** | One step of the pipeline: ingestion, retrieval, rerank, grade, generation. |
| **Blame** | The first stage that failed for a question. |
| **Fault injection** | Breaking one stage on purpose to check that your diagnostics find it. |
| **Gate / policy** | A written rule set that decides whether a candidate may ship. |
| **Baseline / candidate** | The version in production / the version you want to ship. |
| **Precondition** | Something that must be true before two runs can be compared at all. |
| **Non-inferiority** | "Not worse than the baseline by more than a margin", proven by the lower end of the interval. |

---

# Part A — Generation metrics

Sidebar → **Evaluation → Generation** → tab **Metrics playground**. The left panel picks a **scenario**; the main panel shows the four texts, which you can edit. **Score** runs the metrics and shows every intermediate step.

## Step 1 — The four texts

**Do** — Choose scenario **1 · Control: a good answer**. Do not run it yet. Read the four fields.

**Fields**

| Field | Meaning |
|---|---|
| Question | What the employee asked. |
| Reference answer | The correct answer, written by you when you built the dataset (`ground_truth`). |
| Answer | What the system said. This is what you evaluate. |
| Retrieved context | The chunks retrieval gave the generator, in rank order. The arrows reorder them, the bin removes one, **+ chunk** adds one. |
| Metrics | Which metrics to compute. Keep all five on. |
| + embedding relevance | Also compute answer relevance the way RAGAS does (see Step 5). |
| Judge model | The model that computes every metric. Keep the default. |

**Learn — the one table to remember.** Each metric reads exactly two of the four texts. It is blind to the other two, and that blindness is where it lies.

| Metric | Reads | Blind to | So it cannot see… |
|---|---|---|---|
| Faithfulness | answer ↔ context | question, reference | a wrong context |
| Answer relevance | answer ↔ question | context, reference | a wrong answer |
| Context precision | context ↔ question + reference | answer | whether the answer used the chunk |
| Context recall | context ↔ reference | answer | noise in the context |
| Answer correctness | answer ↔ reference | context | *why* the answer is wrong |

---

## Step 2 — Run the control

**Do** — Click **Score**. Wait 10–30 seconds.

**Expect** — Faithfulness **1.0**, context precision **1.0**, context recall **1.0**, answer correctness **1.0**, verdict **correct**. Answer relevance (rubric) may be only **0.5**: read its reason — the judge gave this short, semicolon-style answer 3/5. The rubric scale has only five steps, so relevance moves in jumps of 0.25. The badge row shows the judge model, `T=0`, a **fingerprint** and the number of LLM calls (about 16).

**Look at the steps under each card:**
- **Faithfulness** lists the claims it extracted, a ✓ or ✗ for each, and the judge's one-sentence reason.
- **Context precision** lists every chunk with *useful ✓/✗* and precision@k.
- **Answer correctness** lists each answer claim as **TP** (the reference agrees) or **FP**, and each reference statement as **covered** or **FN** (missing from the answer). Score = TP / (TP + ½(FP + FN)).

**Learn** — The number is the last line of an argument. Always open the argument.

---

## Step 3 — Faithful but wrong

**Do** — Scenario **3 · Faithful but wrong: the archived policy**. Score.

**Expect**

| Metric | Value | Why |
|---|---|---|
| Faithfulness | **1.0** | The answer says 45 €; the context (the 2021 archived policy) says 45 €. Faithful. |
| Answer relevance | **1.0** | It answers the question asked. |
| Context precision | **0** | The only chunk is not useful for the reference (60 €). |
| Context recall | **0** | Nothing in the context supports "60 euros". |
| Answer correctness | **0–0.5** | Low — but maybe not 0. See below. |
| Verdict | **incorrect** | |

**Learn**
1. **Faithfulness is not correctness.** It measures "did the model stick to what it was given". If retrieval gave it the wrong document, a perfectly faithful answer is wrong. This is exactly the Kestrel distractor from Experiment 1, now seen at the answer level.
2. Look at the correctness steps. The answer splits into two claims, *"meals are reimbursed"* (TP) and *"up to 45 € per day"* (FP). One of two claims is right, so claim-F1 gives **partial credit to a wrong answer**. The binary verdict does not. When the question is "is the core fact right?", a binary verdict is the more honest measure.

---

## Step 4 — One invented detail

**Do** — Scenario **4 · Hallucination**. Score.

**Expect** — Faithfulness about **0.67**: 3 claims, the birthday claim ✗. Answer relevance stays high.

**Learn** — Per-claim scoring tells you **which** sentence is invented. A single "is this answer faithful? yes/no" call would say *no* and stop there. Also notice: an answer with 10 true claims and 1 invented one scores 0.91 — faithfulness is a **rate**, and a high rate can still hide a dangerous claim. That is why Part D puts a **floor**, not an average target, on it.

---

## Step 5 — Relevant but wrong

**Do** — Scenario **5 · Relevant but wrong**. Score.

**Expect** — Answer relevance (rubric) **1.0**, embedding relevance about **0.9**. Faithfulness **0**, correctness **0**.

**Learn** — Answer relevance is blind to truth *by definition*: "a wrong answer to the right question is still relevant". Look at the embedding version: RAGAS asks the LLM to write the questions this answer would reply to, embeds them, and averages the cosine to your question. A confidently wrong answer about Zurich hotels produces questions about Zurich hotels — high cosine. Also notice the scale: cosines from `nomic-embed-text` rarely go below ~0.4 even for unrelated text, so 0.75 is *not* "75 % relevant".

---

## Step 6 — The vacuous 1.0

**Do** — Scenario **6 · Vacuous faithfulness**. Score.

**Expect** — Faithfulness **1.0** with the flags `abstention` and `vacuous`. Relevance low. Correctness **0**. Context recall **1.0** — the answer *was* in the context.

**Learn** — "I don't know" has no claims, so no claim is unsupported: 1.0 by convention (RAGAS does the same). A system that always abstains is 100 % faithful. Two rules follow:
- Never report faithfulness without answer relevance or correctness next to it.
- Report the **abstention rate** next to every average. In Part B you will see faithfulness averaged over **4** answered questions out of 26 — a perfect-looking number about almost nothing.

---

## Step 7 — Context precision: rank matters

**Do**
1. Scenario **7 · the useful chunk ranked last**. Score. Note context precision.
2. Use the arrows to move the leave chunk (chunk #4) up to #1. Score again.

**Expect** — First run: context precision **0.25**, recall **1.0**. After the move: precision **1.0**, recall **1.0**. Same four chunks.

**Learn** — Context precision = Σ (precision@k × useful_k) ÷ (number of useful chunks). One useful chunk at rank 4 → precision@4 = 1/4. It is the retrieval metric **MAP** (Experiment 1) with an LLM deciding "useful". It rewards putting the evidence first — which matters because generators read the top of the context more carefully.

---

## Step 8 — Context recall and the reference

**Do**
1. Scenario **8 · half of a multi-document answer**. Score.
2. Scenario **9 · A lazy reference inflates recall**. Score.

**Expect** — Scenario 8: context recall **0.5** (the reference has two sentences; the meal limit is not in the context). Scenario 9: the same context and answer, but a vague reference → context recall **1.0**.

**Learn**
- Scenario 8 is `q-038` from Experiment 1. No generator can give the full answer from this context. An end-to-end score would call this a bad answer; context recall says **retrieval** lost half of it. Part B makes this systematic.
- Scenario 9: every reference-based metric (context precision, context recall, correctness) is only as good as your `ground_truth`. The lab's golden set stores the reference next to the evidence spans; write it with the same care.

---

## Step 9 — When the metric itself is wrong

These scenarios show the judge making mistakes *inside* a metric.

**Do** — Run **2 · Same facts, different words**, **11 · On-topic distractor**, and **12 · A context that contradicts itself**. For each one, open the steps.

**Expect**

| Scenario | What you should see |
|---|---|
| 2 | Same facts as the control, longer sentences. Answer correctness drops to about **0.5**: the judge marks a true claim as FP because it *"only mentions part of the reference"* — although the prompt says partial claims are fine. The verdict stays **correct**. |
| 11 | Context precision **0** (correct), but context recall may say **1.0**: the step's reason says the context *"implies"* Tuesday and Thursday. It does not. The judge filled a gap with a guess. |
| 12 | Faithfulness of *"45 euros"* is **0 or 1**. Chunk 1 says 45, chunk 2 says 60. Is the claim "supported by the context"? The definition does not say. This judge said *not supported (contradicts [2])*. |

**Learn** — A metric computed by an LLM contains that LLM's errors. Three consequences:
1. **The label-based twins.** In Part B every run also computes context precision and recall **from your evidence spans**, with no LLM. When the LLM version and the label version disagree on a question, one of them is wrong, and checking which is the cheapest judge validation there is.
2. **Validate the judge** before trusting a small difference — Part C.
3. **Pin the judge.** Every score carries the judge's fingerprint. Change the judge and the old numbers are no longer comparable — Part D enforces this.

> **Mission A** — Write one sentence per metric: *"<metric> can be high while the answer is bad when …"*. Use a scenario from this part as the example for each.

---

# Part B — Component-level evaluation

An end-to-end score tells you *that* the system fails. You can only fix a failure when you know *where* it happened. Tab **Pipeline runs**.

The lab runs a pipeline shaped like agent-hub's CRAG:

```
question ─► retrieve (k candidates, gRPC vector search)
         ─► rerank + keep top n        ─► grade (CRAG's relevance grader)
         ─► generate (CRAG's prompt)   ─► [grounding check + one retry]
```

For every question it records what each stage **received** and **produced**, and judges each stage **given its input**, using your evidence spans:

| Stage | OK when… | Fails when… |
|---|---|---|
| ingestion | every evidence span sits whole inside an indexed chunk | a span straddles a chunk boundary, or the document is not indexed |
| retrieval | the candidates contain the evidence | no candidate contains it |
| rerank | the kept context still contains the evidence the candidates had | the top-n cut dropped all of it |
| grade | the grader accepts a context that holds the evidence | it rejects it ("false rejection") |
| generation | given good context, the answer is correct | it is wrong, unsupported, or abstains |

A stage whose input was already broken is **skip**, not *fail*: retrieval is not blamed for missing a chunk that was never indexed. **Blame** is the first failing stage.

> **Differences from the real service**, so you do not over-read the numbers: one retrieval (no query-rewrite loop — each stage is then attributable once), the lab's own index, prompts from `adapters/prompts.py` (not Langfuse).

## Step 10 — Run the pipeline

**Do**
1. Set the fields below. Click **Run on dev**. Wait (about 5–7 minutes for 27 questions on a GPU).

**Fields**

| Field | Value | Meaning |
|---|---|---|
| Name | `baseline` | Label for the run. |
| Candidates (k) | `10` | Chunks fetched from know-hub. |
| Context (top n) | `4` | Chunks kept for the grader and generator. |
| Min similarity | `0` | Similarity floor at retrieval. |
| Rerank | `none` | Keep vector order. |
| Generator prompt | `strict` | CRAG's prompt: answer only from context, abstain otherwise. |
| Generator temperature | `0.0` | |
| CRAG relevance grader | **on** | Grades the whole context before generating. Rejected → abstain. |
| Grounding check | off | |
| Fault injection | `none` | |
| Questions | *(empty = all 27)* | |
| LLM context precision/recall | on | |
| Metrics judge model | default | |

**Expect** — a row of tiles with a 95 % interval under each:

| Metric | Value | Read it as |
|---|---|---|
| Answer OK | **≈ 0.19** (n = 27) | Fewer than 1 in 5 questions answered correctly. |
| False abstention | **≈ 0.85** (n = 26) | The system refused 22 of 26 answerable questions. |
| Faithfulness | ≈ 0.94, **n = 4** | Averaged over the 4 questions it answered. |
| Context recall (labels) | **≈ 0.94** | The evidence *was* in the context. |
| Retrieval recall (labels) | **1.0** | Every evidence unit was among the 10 candidates. |

**Learn** — Read the last three rows together: retrieval found the evidence, the context held it, and still the system refused. The faithfulness tile (n = 4) is Step 6's vacuous trap at scale: a high number about almost nothing.

---

## Step 11 — Read one trace

**Do** — In the **Per question** table click `q-003` (*Until when can I use leave days I carried over…*).

**Expect** — The trace shows:
- the stage chips: ingestion **ok**, retrieval **ok**, rerank **ok**, grade **fail** (*False rejection: the context held the evidence, the grader said not relevant*), generation **skip**;
- the 10 candidates; ✦ marks the chunks that contain the evidence, ▸ the ones kept in the context;
- the grader's reason, e.g. *"The context does not specify when carried-over leave can be used"* — while chunk ✦ says *"Carried-over days must be used by 31 March"*;
- the answer: the abstention message.

**Learn** — This is the `CRAG_MIN_SCORE` finding from `agent-hub/src/agent_hub/eval/README.md`, rediscovered by the lab: CRAG's grader judges the **whole** context blob and is told to be conservative, so one good chunk plus three off-topic ones reads as "not relevant". Nothing is broken in isolation. The failure lives in the interaction — which is exactly what only component-level evaluation can show.

---

## Step 12 — Components: funnel and blame

**Do** — Tab **Components**. Choose your `baseline` run.

**Expect**

| Funnel step | n |
|---|---|
| answerable questions | 26 |
| evidence indexed | 26 |
| evidence in candidates | 26 |
| evidence in context | ≈ 25 |
| context accepted | **≈ 4** |
| answered correctly | ≈ 4 |

Blame: **grade ≈ 21**, none ≈ 5, rerank ≈ 1. The *Blame by trap* table shows the grader failing across every trap, not only the hard ones.

**Learn** — The biggest drop in the funnel is the stage to fix first. Here: not retrieval (which Experiment 1 optimised), not the generator — the **grader**. Without this view, the natural reaction to "19 % correct" is to change the generation prompt or buy a bigger model. Both would be wasted.

---

## Step 13 — Test the attribution with fault injection

How do you know the blame logic is right? Break a stage on purpose and see whether it gets blamed.

**Do**
1. In **Fault injection**, set **Questions per run** to `12`, leave **CRAG grader on** switched **off**. Click **Run fault suite**. It runs 6 pipelines over the same 12 questions: `none`, then one per broken stage (about 10 minutes; LLM context metrics are off for speed).
2. When it finishes, the matrix appears.
3. Run the suite again with **CRAG grader on** switched **on**, and compare the two matrices.

**Fields — the faults**

| Fault | What it breaks |
|---|---|
| `ingestion` | Simulated: the chunks that hold the evidence were never indexed. |
| `retrieval` | The evidence chunks are in the index but removed from the candidates. |
| `rerank` | The reranked list is reversed before the top-n cut. |
| `grade` | The grader rejects every context. |
| `generation` | The generator gets no context and an "always answer" prompt. |

**Expect** — Rows = injected fault, columns = blamed stage. *Failed* = questions not answered correctly; *masked* = questions the fault did not break.

| Injected | Grader **off**: blamed (of failed) | Accuracy | Grader **on**: blamed | Accuracy |
|---|---|---|---|---|
| none | generation 7 (5 correct) | — | grade 10 | — |
| ingestion | ingestion 12 | **100 %** | ingestion 12 | 100 % |
| retrieval | retrieval 12 | **100 %** | retrieval 12 | 100 % |
| rerank | rerank 11, generation 1 | 92 % | rerank 11, grade 1 | 92 % |
| grade | grade 12 | **100 %** | grade 12 | 100 % |
| generation | generation 11 (1 masked) | **100 %** | **grade 11**, generation 1 | **8 %** |

**Learn**
- With the grader off, attribution finds every fault it was given. The one rerank miss is a question with evidence in two chunks: reversing the order kept one of them, so rerank was only a *warn* and the generator got the blame.
- With the grader on, the **generation fault disappears**: the grader already rejects 10 of 12 contexts, so blame stops at *grade* and the broken generator is never reached. Attribution reports the **first** failure. A fault downstream of an existing failure is invisible — so test your diagnostics on a pipeline whose own failure rate is low, and re-test after you fix the first bottleneck.
- *Masked* is a finding too: the generator guessed one answer right from general knowledge, with no context at all. A diagnostic you have not tested on a known fault is a diagnostic you are trusting on faith.

---

## Step 14 — Fix the bottleneck, watch it move

Change **one** field at a time, run on dev, and write down Answer OK, false abstention and blame.

| Run name | Change from baseline | Answer OK | False abst. | Main blame |
|---|---|---|---|---|
| `baseline` | — | ≈ 0.19 | ≈ 0.85 | grade |
| `grader-off` | CRAG relevance grader **off** | **≈ 0.56** | ≈ 0.27 | generation ≈ 11 (none ≈ 15) |
| `top-n-2` | Context (top n) **2** | ≈ 0.19 | ≈ 0.85 | grade ≈ 17, rerank ≈ 5 |
| `floor-0.65` | Min similarity **0.65** | ≈ 0.07 | ≈ 0.92 | **retrieval ≈ 20** |

**Learn**
- **Fixing one stage moves the bottleneck.** With the grader off, Answer OK triples (0.19 → 0.56) and false abstentions fall from 22 to 7. The blame moves to **generation**: now the strict generator sometimes abstains or answers wrongly *from good context*. That is the next stage to work on — and you know it without guessing.
- **Fewer chunks is not cleaner context.** `top-n-2` does not help the grader (still ≈ 17 rejections), and the smaller cut now drops the evidence when it was at rank 3 or 4 (**rerank** blamed ≈ 5). One change, two effects in two stages.
- **A U-shape, not a line.** Too permissive (4 noisy chunks) and the grader rejects good context; too strict (floor 0.65) and retrieval returns nothing — most Kestrel chunks score 0.55–0.75, so the floor removes the evidence (retrieval recall (labels) falls from 1.0 to ≈ 0.19). The harness README chose 0.65 for a different corpus; a threshold does not transfer between corpora. The two ends fail in different stages — only per-stage blame shows that they are different failures.
- These are dev-split explorations. A claim that "grader off is better" needs the same discipline as Experiment 1, Part C — and a gate (Part D).

---

## Step 15 — Compare two runs

**Do** — Tab **Compare runs**. A = `baseline`, B = `grader-off`. **Compare A → B**.

**Expect**
- A forest plot of paired deltas (B − A) per metric. For *false abstention* and *answered unanswerable* the sign is flipped, so right = better for every row.
- **Outcome transitions**, e.g. `false_abstention → correct` ×N, `false_abstention → incorrect` ×M.
- **Blame A vs B**: the grade share disappears; generation appears.

**Learn** — The transitions table is the honest version of a mean: "+0.15 answer OK" might be 6 questions fixed and 2 broken. The broken ones are your next trace to read.

> **Mission B** — Pick one question that went `false_abstention → incorrect`. Open its trace in the B run. Which stage is blamed, and what does the faithfulness card say? Write one sentence on what you would change next.

---

# Part C — LLM-as-judge design

Every generation metric in Parts A and B was computed by an LLM judge. Before a judge decides anything, you measure it like any instrument: against a reference (**your labels**), for known biases, and for stability. Sidebar → **Evaluation → Judge Lab**.

## Step 16 — Import the calibration set

**Do**
1. Tab **Human labels**. In **New sample set**: Name `Kestrel judge calibration`.
2. Open `data/judge-calibration.jsonl`, copy all 31 lines, paste into **JSONL**. Click **Create from JSONL**.

**Expect** — The set shows `31` samples and *Labels so far: guide 31*. The guide author's labels are stored under the rater name **guide**.

---

## Step 17 — Label blind

**Do**
1. Set **You are (rater name)** to `me`.
2. For each sample: read the question, the reference, (optionally) the context, and the **answer**. Click **Pass** or **Fail**, a score **1–5**, and write a short note when you hesitate. **Submit & next**. All 31 take 15–20 minutes.

**Fields**

| Control | Meaning |
|---|---|
| Pass / Fail | Could a careful employee act on this answer safely? Wrong, evasive, unsupported or missing the point → **fail**. |
| Score 1–5 | Use the anchors in `rubric-v2-anchored.txt`: 5 same key facts · 4 minor gap · 3 part missing · 2 vague · 1 wrong. |
| Note | Why. You will read these notes again at the disagreements. |

**Expect** — The progress bar reaches 31/31. You never saw the `kind`, the guide's label or any judge output — that is what *blind* means.

**Learn** — Your labels are the **reference** for every judge you build. Two humans rarely agree 100 %: the `partial` answers (Is *"Tell your manager on the first day"* a fail when the question asked *by what time*?) are where you and the guide will differ. In Step 19 that disagreement becomes a number — the **ceiling** a judge can be validated to.

---

## Step 18 — A vague judge

**Do** — Tab **Rubrics** → Judge **+ new judge**.

**Fields**

| Field | Value | Why |
|---|---|---|
| Name | `Quality (vague)` | |
| Model | default | |
| Mode | score 1…N | |
| Temperature | `0.0` | A judge must give the same answer twice. |
| Scale max / Pass if ≥ | `5` / `4` | |
| Show the reference answer | **off** | The vague rubric does not use it. |
| Show the retrieved context | off | |
| Reasoning before the score | on | |
| Rubric | paste `rubric-v1-vague.txt` | |

Click **Create**. Then in **Try it on one answer** click **Judge it** (the default example is the archived 45 € answer).

**Expect** — The prompt preview shows the exact system and user text the judge receives. Without the reference, the judge cannot know 45 € is outdated; it will probably score it 4 or 5.

---

## Step 19 — Validate it against your labels

**Do**
1. Tab **Validate**. Judge `Quality (vague)`, set `Kestrel judge calibration`. **Run judge on the set** (about 1 minute).
2. Compare with rater **guide**, then again with rater **me**.

**Read these numbers**

| Number | Meaning | Vague judge vs *guide* |
|---|---|---|
| Accuracy | Verdicts that match the human. | **0.77** |
| Cohen's κ [CI] | Agreement beyond chance, with a bootstrap interval over samples. | **0.53 [0.21, 0.80]** |
| Fail recall | Of the answers the human failed, the share the judge failed. | 0.83 |
| Pass rate (judge / human) | The pass rate each would report on a dashboard. | 0.39 / 0.42 |
| Corrected pass rate | The pass rate implied by the judge's measured error rates (Rogan–Gladen). | 0.42 |
| Spearman (scores) | Rank agreement of the 1–5 scores. | **0.40** |

**By kind** (the table the aggregate hides):

| Kind | Human pass | Judge pass |
|---|---|---|
| `verbose-wrong` | 0 % | **100 %** |
| `plausible-wrong` | 0 % | **100 %** |
| `wrong-number` | 0 % | 33 % |
| `correct` | 100 % | **60 %** |

**Learn**
- κ 0.53 is "moderate", and its interval is wide: 31 samples cannot pin a judge down. Experiment 1's statistics apply to judges too.
- The pass rates almost match (0.39 vs 0.42) — **and the judge is still bad**. It fails four short correct answers and passes the fluent wrong ones. The errors cancel in the average. Never validate a judge by comparing averages; compare item by item.
- Open **Disagreements**. The fluent `verbose-wrong` hotel answer (180 € invented) scores 4; the one-line correct answer *"220 euros per night"* scores 3. Without a reference, the judge rewards *sounding* helpful.
- Switch the rater to **me**. If your κ with the judge differs from the guide's, look at the **Human vs human** box: that is how much you and the guide agree. A judge cannot be validated more precisely than that.

---

## Step 20 — An anchored, reference-guided judge

**Do**
1. Rubrics → **+ new judge**: Name `Correctness (anchored)`, **Show the reference answer: on**, rubric = paste `rubric-v2-anchored.txt`, other fields as in Step 18. **Create**.
2. Validate → run it on the set → compare with **guide** and **me**.

**Expect (vs guide)**

| Number | Vague | Anchored |
|---|---|---|
| Accuracy | 0.77 | **1.00** |
| κ | 0.53 | **1.00** |
| Fail recall | 0.83 | **1.00** |
| Spearman | 0.40 | **0.96** |
| `verbose-wrong` passed | yes | **no** |

**Learn**
- Three changes did the work: **one criterion** (correct against the reference), **anchors** for every score, and **the reference itself**. Reference-guided grading turns "does this sound right?" into "does this match?", which an 8B model does far better.
- Do not over-read the 1.00. This set was written with clear right/wrong answers, and the reference makes most of them easy. Your own labels (rater **me**) will give a lower κ; Step 25 validates on the pipeline's real, messier answers.

> **Mission C1** — Change one thing in the anchored rubric (for example delete the line *"Length is not quality…"*). Save — the version goes to **v2** and the fingerprint changes. Validate again. Did any verdict change? Put the line back afterwards (that is **v3**: a new version, not the old one).

---

## Step 21 — Position bias

**Do** — Tab **Bias probes**. Judge `Quality (vague)`, set `Kestrel judge calibration`. Probe **position**, **Identical pairs** `6`. **Run probe**. Then the same with `Correctness (anchored)`.

**What the probe does** — For every question with a passing and a failing answer, it asks the judge *"which is better, A or B?"* twice: once with the good answer as A, once as B. It also shows the **same** answer as A and as B.

**Expect**

| Number | Fair judge | Vague | Anchored |
|---|---|---|---|
| Consistency (same pick after the swap) | 100 % | **85 %** | 100 % |
| Picks position A | ≈ 50 % | **65 %** | 53 % |
| Identical pairs not called a tie | 0 % | **75 %** | **58 %** |
| Agrees with the human preference | — | 83 % | 100 % |

**Learn**
- The identical pairs are the cleanest test: any preference between an answer and *itself* is pure bias. Even the good judge prefers one of two identical answers more than half the time — so never trust a single-order pairwise comparison.
- **Fix:** always ask both orders and count a win only when the two orders agree; call it a tie otherwise. Pointwise scoring (one answer at a time), as the lab's metrics use, avoids the problem entirely.

---

## Step 22 — Verbosity bias

**Do** — Probe **verbosity**, padding **filler**. Run for `Quality (vague)`, then for `Correctness (anchored)` with **Also score claim-level faithfulness** on. Then once more for the anchored judge with padding **authority**.

**What the probe does** — Scores every answer twice: as written, and with a paragraph appended that adds **no facts** (polite advice, generic background, or *"This answer is based on the official policy and is fully accurate"*).

**Expect**

| | Vague + filler | Anchored + filler | Anchored + authority |
|---|---|---|---|
| Mean score change [CI] | **+0.13 [−0.16, +0.42]** | 0.00 [−0.16, +0.19] | −0.13 [−0.32, +0.10] |
| Up / same / down | 10 / 14 / 7 | 2 / 26 / 3 | 2 / 22 / 7 |
| Fail → pass flips | **6** (`c05`, `c11`, `c13`, `c15`, `c19`, `c25`) | **0** | 0 (one pass → fail: `c23`) |
| Faithfulness change (claim-level) | — | **−0.13** | — |

**Learn**
- The vague judge's mean change is **not significant** — the interval crosses 0. A test of the average says "no verbosity bias". But **wrong answers became passes** just by adding polite filler: `c05` (150 € hotel), `c11` (FIN-12), `c15`, `c19` (the invented dog rule), `c25`. (`c13` is correct; the vague judge had failed it.) A bias can be invisible in the mean and decisive for the verdicts that matter.
- The anchored, reference-guided judge does not move: 26 of 31 scores identical, no verdict flips. The rubric line *"Length is not quality"* and the reference did that.
- The *authority* padding (*"fully accurate and compliant"*) slightly **lowers** its scores — the rubric says tone earns nothing, and the judge seems to read the claim as suspicious.
- Claim-level faithfulness goes **down** (−0.13) with the same filler: it adds claims the context does not support. Per-claim scoring cannot be bought with more words; a single holistic score can.

---

## Step 23 — Self-consistency

**Do** — Probe **consistency**, judge `Correctness (anchored)`, **Repeats** `5`, **Temperature** `0.7`. Run. Then again with Repeats `3`, Temperature `0.0`.

**Expect**

| | T = 0.7 × 5 | T = 0.0 × 3 |
|---|---|---|
| Verdict flip rate | **≈ 10 %** (`c07`, `c08`, `c22`) | **0 %** |
| Mean score SD | ≈ 0.26 | 0.00 |

**Learn** — At temperature 0.7 the same judge gives the same answer different verdicts from one call to the next: 3 of 31 answers flip, including `c07` (correct) and `c08` (the birthday hallucination). That noise adds straight into every comparison you make with the judge: a "+0.05" between two systems can be the judge's own randomness. Temperature 0 (and a fixed seed) makes it repeatable — but repeatable is not the same as correct. Consistency is necessary, not sufficient.

---

## Step 24 — Judge drift

Drift = the scores change because the **judge** changed. It happens silently when someone edits a prompt, updates the model tag, or "tidies" the output format.

**Do**
1. Rubrics → `Correctness (anchored)` → switch **Reasoning before the score** **off**. Save. The version goes up and the fingerprint changes.
2. Validate → run the judge on the set again.
3. Tab **Drift**. Run A = the anchored run from Step 20, Run B = the new one. **Compare**.

**Expect**

| Number | Value |
|---|---|
| Changed fields | `reasoning_first`, `version` |
| Verdict agreement / κ between runs | ≈ 0.94 / 0.87 |
| Pass rate A → B | 0.42 → **0.48** |
| Mean score shift [CI] | **+0.26 [+0.06, +0.45]** — the interval excludes 0 |
| Against guide: κ A → B | **1.00 → 0.87**, fail recall 1.00 → 0.89 |
| Flips | `c08` (birthday hallucination) and `c14` (partial) now **pass** |

**Optional — a model change.** Pull a second model: `docker exec brainwave-ollama ollama pull llama3.2:3b` (about 2 GB). Create a copy of the anchored judge with Model `llama3.2:3b`, run it, and compare in **Drift**. A smaller model usually agrees less with you — and in Part D the gate will refuse to compare runs scored by it.

**Learn** — One field that "only changes the format" made the judge measurably more lenient: +0.26 on average, and two failing answers — one of them a hallucination — now pass. Asking for the reasoning first is a cheap chain-of-thought; without it the model commits to a score before it has compared the facts. Put the setting back (another new version) before Part D. Every judged number is a pair *(system, judge)*. If the judge moves, old and new numbers are not comparable — that is why each run stores the judge fingerprint, and why the release gate checks it (Step 30).

---

## Step 25 — Validate on real pipeline answers

The calibration set was built by hand. Your judge will grade the pipeline's real answers.

**Do**
1. Human labels → **…or sample 20 answers from a generation run** → choose your `grader-off` run → **Create from run**. (It has many real answers; the baseline mostly abstained.)
2. Label the 20 answers blind as `me`.
3. Validate the anchored judge on this set with rater `me`.

**Expect** — Lower agreement than on the calibration set, and new kinds of disagreement: partially correct multi-document answers, answers with citations like `[doc:…]`, correct abstentions. The `kind` column here is the pipeline's own **outcome** (`correct`, `incorrect`, `false_abstention`) — so the per-kind table shows how well the pipeline's automatic `answer_ok` verdict agrees with you.

**Learn** — Validate the judge on the distribution it will actually grade. A judge that is perfect on clean, hand-made examples can fail on the messy middle that real systems produce.

> **Mission C2** — From your validation, write the judge's report card in three lines: κ with its interval, fail recall, and the one kind of answer it gets wrong most. That is what anybody quoting its scores needs to know.

---

# Part D — Regression gates

A gate decides whether a change may ship. Most teams start with *"block if the mean drops by more than 0.05"*. This part shows why that is not enough, and builds a gate that (1) refuses comparisons it cannot make, (2) knows when a change is too small for its data, and (3) runs in CI. Sidebar → **Evaluation → Release Gate**.

## Step 26 — Read the policy

**Do** — Tab **Policy**. Read the default policy `kestrel-release`.

**Preconditions** — failing any one is a **BLOCK**, because the gate cannot certify what it cannot compare:

| Precondition | Why |
|---|---|
| Same kind of run, both finished, same split | Obvious — but gates silently compare the wrong files. |
| Same dataset content hash | Different questions or labels = different ruler. |
| Frozen dataset | A draft can change under the number. |
| Same judge fingerprint | Otherwise the delta includes judge drift (Step 24). |
| At least 20 paired items | Below that, no interval means much (Experiment 1, Step 12). |
| Error rate ≤ 5 % | Errored items are excluded, which silently shrinks the set. |

**Rules**

| Rule | Metric | Kind | Threshold | Severity | Why |
|---|---|---|---|---|---|
| faithfulness-floor | faithfulness | floor ≥ | 0.80 | block | Unsupported claims are the failure users cannot detect. |
| answer-ok-noninferior | answer_ok | non-inferior | 0.20 | block | The correct-answer rate must not be **measurably** worse by more than 20 points. Why 20 and not 10: see Step 31. |
| unanswerable-ceiling | answered_unanswerable | ceiling ≤ | 0.00 | block | Answering what no document covers is a fabrication. |
| false-abstention-drop | false_abstention | max drop | 0.05 | warn | More refusals hurt usefulness, not safety. |
| relevance-drop | answer_relevance | max drop | 0.05 | warn | Style — worth a look. |
| recall-noninferior | recall@k | non-inferior | 0.05 | block | (retrieval runs) The generator cannot cite what retrieval never returned. |
| mrr-drop | mrr | max drop | 0.05 | warn | (retrieval runs) |

**The four rule kinds**

| Kind | Passes when | Honest about noise? |
|---|---|---|
| floor | candidate mean ≥ threshold | No — an absolute bar. Good for safety. |
| ceiling | candidate mean ≤ threshold | No — for "bad" rates. |
| max_drop | candidate − baseline ≥ −threshold (point estimates) | **No.** Cheap and common; flaps on chance. |
| non_inferior | the **lower bound** of the paired Δ interval ≥ −threshold | **Yes.** "Can I rule out a loss bigger than the margin?" |

---

## Step 27 — Promote a baseline

**Do** — Tab **Evaluate**. Candidate run = your `baseline` generation run. Click **Promote candidate as the new baseline**.

**Learn** — The baseline is what production does today. It is a *recorded* choice, not "whatever ran last".

---

## Step 28 — Evaluate a real improvement

**Do** — Candidate = `grader-off`, Baseline = *promoted baseline*. **Evaluate**.

**Read the rules table** — For each rule: baseline mean, candidate mean with its interval, the paired Δ with its interval (oriented so + = better), **detects ±** (the smallest change these paired questions can resolve), and the result.

**Expect** — **PASS**, exit 0. All 8 preconditions ✓ (same hash, frozen, same judge fingerprint, 27 paired items, 0 errors).

| Rule | Baseline | Candidate [CI] | Δ [CI] | Detects ± | Result |
|---|---|---|---|---|---|
| faithfulness-floor | 0.94 | 0.90 [0.82, 0.97] | 0.00 (n = 4) | — | pass |
| answer-ok-noninferior | 0.19 | 0.56 [0.37, 0.74] | **+0.37 [+0.19, +0.56]** | 0.27 | pass |
| unanswerable-ceiling | 0.00 | 0.00 | — | — | pass |
| false-abstention-drop | 0.85 | 0.27 | **+0.58** (better) | 0.28 | pass + advisory |
| relevance-drop | 1.00 | 0.78 | 0.00 (n = 4) | — | pass |

**Learn**
- The improvement is **measured**: the lower end of the Δ interval (+0.19) is far above the −0.20 margin — and above 0.
- Look at *n* in the faithfulness and relevance rows: **4**. Both metrics exist only for answered questions, and the baseline answered 4. The *paired* Δ uses only questions both runs answered, so it says 0.00, while the candidate's own mean (0.78 relevance, over 19 answers) looks lower than the baseline's (1.00, over 4). Neither mean is wrong; they average over different questions. This is **survivorship**: a metric defined only on survivors changes when the set of survivors changes. That is why the gate's primary rule is on `answer_ok`, which is defined for every question.
- The advisory on `false-abstention-drop` says 26 questions resolve only about ±0.28. It does not change the decision; it tells you this rule's tolerance (0.05) is inside the noise. Step 31 shows what that means.

---

## Step 29 — Block a regression

**Do**
1. Generation → Pipeline runs: the **baseline** settings, but **Fault injection** `break generation` (the generator gets no context and must always answer), Name `broken-generator`, all 27 questions. Run.
2. Release Gate → Evaluate: candidate = `broken-generator`. **Evaluate**.

**Expect** — **BLOCK**, exit 1, from two rules:

| Rule | Candidate | Why it fails |
|---|---|---|
| faithfulness-floor | **0.56** [0.25, 0.75] < 0.80 | The answers it gives are guessed. |
| answer-ok-noninferior | 0.07; Δ −0.11, lower bound **−0.22** < −0.20 | A loss larger than the margin cannot be ruled out. |

`unanswerable-ceiling` still passes: the grader (still on) rejected the dog question before the broken generator saw it. False abstentions barely move (0.85 → 0.88) — the grader shields most questions from the broken stage.

**Learn** — The **floor** catches what averages hide. Only 3 questions reached the broken generator, and it invented answers for them; the correct-answer rate moved only 0.11. A floor on faithfulness blocks confident invented answers even when few of them get through.

---

## Step 30 — A comparison the gate refuses

**Do**
1. Generation → Pipeline runs: fault `none`, the `grader-off` settings, **Questions** `10`. Run.
2. Release Gate → Evaluate this 10-question run against the baseline.

**Expect** — **BLOCK** from a precondition: *At least 20 paired items — 10 items scored by both runs*. The rules below are not worth reading, but look anyway: `answer-ok-noninferior` *passes* (Δ +0.20 [0.00, +0.50], detects ±0.37), and `unanswerable-ceiling` fails because none of the 10 questions is unanswerable — the metric is missing. On a subset, rules measure a different thing or nothing at all.

**Optional** — If you pulled `llama3.2:3b` (Step 24): run the pipeline with **Metrics judge model** `llama3.2:3b` and evaluate it. **BLOCK**: *Same judge — judge 42ff… vs …*. The candidate may be identical to the baseline; the numbers are not comparable, so the gate refuses.

**Learn** — The most common way a gate lies is comparing things that are not comparable: a new dataset version, a new judge, a smaller subset. A good gate fails **closed** on these.

---

## Step 31 — A rule that cries wolf

**Do**
1. Generation: run the **baseline** settings again, name `baseline-rerun`. **Nothing changed.**
2. Evaluate `baseline-rerun` against the promoted baseline.
3. Policy tab: change `false-abstention-drop` to threshold `0.01`, severity **block**. Save (the version goes up). Evaluate `baseline-rerun` again.
4. Policy tab: change `answer-ok-noninferior` to margin `0.10`. Save. Evaluate again.
5. Put both rules back (**Default** button).

**Expect**

| Evaluation | Decision | What happened |
|---|---|---|
| 2. rerun, default policy | **WARN** | The rerun is *not* identical: Answer OK 0.19 → **0.11**, false abstention 0.85 → 0.92. `false-abstention-drop` (Δ −0.08 < −0.05) fails; *detects ±0.15*. |
| 3. tolerance 0.01, block | **BLOCK** | An unchanged system is blocked. |
| 4. margin 0.10 | **BLOCK** | Δ −0.07 [−0.19, 0.00]: the lower bound −0.19 cannot rule out a 10-point loss. |

Your rerun may be identical to the baseline — then 2 passes. Run it once more, or with **Generator temperature** `0.7`: in our run that gave Answer OK 0.07.

**Learn**
- **Temperature 0 is not bit-for-bit repeatable.** Ollama on a GPU batches requests; the floating-point order changes, and a borderline grader verdict flips. Here 2 of 27 questions changed, with no change to the system. That run-to-run noise is part of what the gate measures.
- A **max_drop** tolerance smaller than what the data can detect (±0.15 here) fires on noise. Teams then learn to ignore the gate, and an ignored gate protects nothing.
- A **non-inferiority** margin smaller than what the data can detect never passes — it blocks the unchanged system. With 27 questions, the strongest honest promise is *"not worse by more than about 20 points"*. To promise 10 points, you need roughly 4× the questions (the √n rule from Experiment 1, Step 12). That is why the default margin is 0.20.

---

## Step 32 — Record and override

**Do**
1. Evaluate `broken-generator` again. In **Record this decision**, click **Record decision** (no override).
2. Now type an override reason of fewer than 30 characters and click **Record override** → refused.
3. Type a real reason (for example *"Emergency hotfix for the login outage; generation quality re-tested manually on 20 questions."*) with your name. **Record override**.
4. Tab **Decisions**.

**Expect** — Two rows: one `block → block`, one `block → pass overridden` with your reason and name.

**Learn** — Sometimes you ship past a gate. The rule is not "never override"; it is **"never override silently"**. The verdict is never rewritten; the override is recorded next to it, with a name and a reason someone can review later.

---

## Step 33 — Wire it into CI

**Do**
1. Evaluate your best candidate (`grader-off`) against the baseline. Click **Export for CI**.
2. Tab **CI**. Read *What CI will decide* — the same Markdown summary GitHub will show.
3. In a terminal:

   ```bash
   cd infra && make evals-release-gate ; echo "exit code: $?"
   ```

4. Open `agent-hub/.github/workflows/eval-gate.yml` and read the header comment.

**Expect** — `infra` prints the Markdown table (*Eval release gate — ✅ PASS*); exit code **0**. The files `policy.json`, `baseline.json` and `candidate.json` (about 60 KB together — the export keeps per-question metrics and outcomes, not the full traces) now exist in `agent-hub/src/agent_hub/eval/release/`. Export `broken-generator` instead and the same command exits **1**.

**How the wiring works**

| Where | What runs | Cost |
|---|---|---|
| Your machine (GPU) | The pipeline, the judge, Release Gate → Export for CI | minutes, an 8B model |
| Git | The snapshot is committed **with the change** it measures | the reviewer sees the numbers in the diff |
| CI, every pull request | `eval-unit` (the metric, stats, attribution and gate tests, with fakes) → `release-gate` (the CLI on the snapshot) | seconds, no model |

Exit codes: `0` pass · `1` **block** · `2` the gate could not run (missing or invalid snapshot) — also a failure: a change to the reasoning path must carry its evidence. The workflow triggers only on changes that can move answer quality (CRAG graph, prompts, reasoning adapters, eval code, the snapshot).

**Learn** — The snapshot is not trusted blindly: the dataset hash, frozen status and judge fingerprint are preconditions, so a stale or hand-edited run is refused.

---

## Step 34 — Write your own policy

**Do** — Rewrite the policy for a real release of the Kestrel assistant. Answer each question in the rule's **rationale**:

1. Which failure would you roll back a release for? That — and only that — is **block**.
2. For each comparative rule: what is the smallest change that matters, and can 26 questions detect it? If not: warn, widen the margin, or grow the dataset (Experiment 1).
3. Which metric is judged by an LLM, and is that judge validated (Part C)? An unvalidated judge should not block.

**Expect** — Your policy's version goes up with each saved change. Evaluate `grader-off` and `broken-generator` again under your policy: your gate should pass the first (or warn) and block the second.

> **Mission D** — In five lines, write your team's release policy in plain words: what blocks, what warns, which preconditions, who may override and how. That text belongs at the top of the policy's description.

---

## Self-test

1. The archived-policy answer scored faithfulness 1.0. What does that tell you, and which metric catches the problem?
2. Why is "I don't know" 100 % faithful, and what must you report next to faithfulness?
3. Context recall is 1.0 but the answer is wrong. Which stage do you look at next?
4. Why does the lab mark a stage `skip` instead of `fail` when an earlier stage failed?
5. The fault matrix shows `masked` for some rerank questions. Is that a bug?
6. Your judge's pass rate equals the human pass rate. Is the judge validated?
7. Why include pairs of *identical* answers in a position-bias test?
8. The verbosity probe's mean change is not significant. Can the judge still have a verbosity problem?
9. Why does the gate block when the judge fingerprint changes, even if the system did not change?
10. Why is `max_drop 0.01` a bad blocking rule on 26 questions, and what would you use instead?
11. What does exit code 2 mean in CI, and why is it a failure?

<details>
<summary>Answers</summary>

1. The answer matches its context — but the context is wrong (stale). Faithfulness cannot see that. Answer correctness / the binary verdict (reads the reference) and context precision/recall (the chunk was not useful for the reference) catch it.
2. It has no claims, so none is unsupported (a convention). Report answer relevance or correctness, and the abstention rate, next to it.
3. The generator (and the grader, if on): the evidence reached the context, so ingestion and retrieval did their job.
4. A stage cannot be faulted for input it never received. Otherwise every failure would be counted against every later stage and the blame would be meaningless.
5. No. The fault did not break those questions — for example the evidence also sat in another chunk that survived. It tells you those questions do not depend on that stage.
6. No. Errors in both directions can cancel. Compare item by item: κ, fail recall, the confusion matrix, the per-kind table.
7. Any preference between an answer and itself is pure position bias — a test with a known correct result (tie).
8. Yes. A few decisive fail → pass flips can hide inside a mean change that is not significant.
9. The delta would mix the system change with the judge change. The numbers are not comparable, so the gate cannot certify anything.
10. 26 questions resolve changes of roughly ±0.1–0.2, so a 0.01 tolerance fires on noise. Use non-inferiority with a margin the data can resolve, and keep noisy rules at *warn*.
11. The gate could not run (snapshot missing or invalid). A gate that silently passes when its input is missing protects nothing.

</details>

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| Generation pages show an API error | eval-api not running or not reloaded | `cd infra && make evals-lab`; check `make evals-logs s=eval-api` |
| *index the corpus … before a generation run* | Index `off` not built | Golden Sets → Index → **Index corpus** |
| *Ad-hoc generation runs cannot read holdout items* | You tried split holdout | Generation runs use **dev** only. The holdout is for one registered look. |
| A metric card says *judge call failed — not scored* | The 8B model returned broken JSON twice | Run again; if it repeats, check `make evals-logs s=ollama` |
| Runs are very slow | CPU-only Ollama, or two jobs at once | Use **Questions** to limit the run; switch LLM context metrics off; run one job at a time |
| Judge Lab: *Nothing to compare* | No label from that rater on the set | Label first (Human labels), or pick the rater **guide** |
| Gate: *No promoted generation baseline* | Nothing promoted yet | Evaluate → pick a run → **Promote candidate as the new baseline** |
| `make evals-release-gate` exits 2 | No snapshot exported | Release Gate → Evaluate → **Export for CI** |
| Numbers far from this guide | Different model tag, or your own labels | Compare the judge fingerprint and the rater; small differences are normal |

## Start again

- **Soft reset:** new judges, sets and runs can sit next to the old ones; give them new names.
- **Delete a generation run:** `curl -X DELETE http://localhost:3005/api/eval/gen/runs/<run_id>`.
- **Full reset of this part only:** stop the lab and delete these folders under the `lab_dir` shown at `http://localhost:3005/api/eval/status`: `gen_runs/`, `judges/`, `judge_sets/`, `judge_runs/`, `gate_policies/`, `gate_decisions/` and the file `gate_baselines.json`. Your Experiment 1 datasets, runs and experiments are not touched.

---

*Data: fictional company, written for these guides. Every number in the Expect sections comes from a run of this lab on `kestrel-hr-policies-en` v1 with `llama3.1:8b-instruct-q4_K_M` at temperature 0.*
