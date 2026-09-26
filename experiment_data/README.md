# Experiment Data — English Corpus

A ready-to-upload corpus for the Eval Lab (Evaluation → Dataset Construction).
English only, on purpose — no Arabic traps here, so it stays out of the way
while you focus on retrieval metrics, dataset construction, and statistical
reasoning.

## What's here

`corpus/` — 16 Markdown files, in 8 topic pairs. Each pair is one **answer
document** (a real policy with concrete, quotable facts) and one **distractor**
(topically adjacent, but it answers a different question):

| Answer document | Distractor pair | Answer document | Distractor pair |
|---|---|---|---|
| `leave-policy` | `leave-request-form` | `onboarding-policy` | `onboarding-welcome-email` |
| `refund-policy` | `refund-faq-marketing` | `data-retention-policy` | `records-management-guide` |
| `remote-work-policy` | `remote-work-equipment` | `password-policy` | `it-security-overview` |
| `expense-policy` | `expense-report-guide` | `procurement-policy` | `vendor-code-of-conduct` |

The pairing is deliberate: `leave-policy` and `leave-request-form` share almost
all their vocabulary ("leave", "request", "HR portal"), but only one of them
answers "how many days of annual leave do I get?" A retriever with weak
ranking will pull the wrong one to the top — that's the point.

**"Distractor" is not a filename convention.** In the lab, a document becomes
a distractor the moment no golden item lists it as relevant, and an answer
document the moment one does. Nothing here marks that in advance — it emerges
from how you write your questions.

## Every document has a countable fact

Each one carries at least one specific number (days, dollars, hours) so a
question about it has one unambiguous answer — e.g. "twenty (20) working
days," "thirty (30) days," "ninety (90) days." When you write golden items,
quote that sentence verbatim as the evidence span.

## How to use it

1. Evaluation → Dataset Construction → **Create** a dataset (or fork one),
   language `en`.
2. **Corpus** tab → **Upload .md/.txt** → select all 16 files in `corpus/`.
   The filename (minus `.md`) becomes the `doc_id`; it's already
   lab-safe (letters, digits, `-` only).
3. **Golden items** → write questions the way a user would ask, e.g. *"How
   many annual leave days do full-time employees get?"* → grade
   `leave-policy` a 3, add the matching sentence as an evidence span.
4. **Index** the corpus (normalization `off` — there's nothing to normalize
   in English).
5. **Pool & judge** once you have ~15+ questions, to catch anything a
   retriever surfaces that you didn't think to label.
6. **Data Lab** → split (holdout ~30%, grouped by primary document) → clear
   blockers → **freeze**.

## Sizing

16 documents, ~75 words each — enough for one real chunk per document (well
under the 512-token/384-word chunk window), so chunk-level and document-level
scoring should agree closely. Write **20–30 questions** against this corpus
before you split: enough for a first Data Lab verdict, not enough yet for a
tight confidence interval (see the Power panel in Data Lab / Stats Lab for
what "enough" would actually take).

## Extending it

Add more topic pairs the same way — one factual policy, one adjacent document
that shares its vocabulary but not its answer. Keep filenames as the intended
`doc_id` (letters, digits, `.`, `_`, `-` only) and keep every fact a single,
quotable sentence.
