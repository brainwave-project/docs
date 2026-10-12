# Arabic normalization diagnostic fixture

This is a deliberately small, authored test set: **16 passages, 12 queries, 192 binary judgments**. All organizations and policies are fictional. These are engineering checks and explanatory examples; they cannot establish how often normalization helps Arabic search. The text is original to this repository and covered by `docs/LICENSE` (MIT).

Use `../diagnostic-bundle.json` in Evaluation → Arabic Norm for a single-file import. The separate files support the command-line importer and other retrieval tools:

- `corpus.jsonl`: one `{passage_id, title, text}` record per line; empty titles are intentional.
- `queries.jsonl`: one `{qid, query, tags}` record per line.
- `qrels.tsv`: `qid 0 passage_id relevance`, separated by tabs. Every pair is judged; `1` means relevant to the intended sense below, `0` means irrelevant.
- `provenance.json`: source, scope, counts, and labeling policy.

The short single-word queries intentionally make ambiguity visible. Their relevance labels encode the intended sense, just as a human searcher has an intent that an isolated string may not fully express. Review or replace those labels for your own application; do not present them as independent human benchmark annotations.

| Query ID | Intended meaning and relevant passage | What to inspect |
|---|---|---|
| diag-q01 | Annual leave policy; diag-01 | Unchanged query can benefit when Alef/diacritics in documents change. |
| diag-q02 | Travel procedures; diag-02 | Tatweel inside document words; inspect token matching before and after. |
| diag-q03 | Working hours; diag-03 | Bare Alef in query against hamzated Alef in passage. |
| diag-q04 | Public library; diag-04 | Query contains no vowels; the relevant document does. |
| diag-q05 | Club subscription; diag-05 | Plain spelling control. |
| diag-q06 | The 2026 grant; diag-06 | Digits are outside the six primary presets; leave this as a scope check. |
| diag-q07 | The person Ali; diag-07 | Aggressive ى→ي conflates the preposition على with the name علي. |
| diag-q08 | The person Huda; diag-09 | Aggressive ى→ي also conflates هدى with هدي in a different sense. |
| diag-q09 | Knowledge/science, عِلْم; diag-11 | Removing diacritics conflates عِلْم with عَلَم (a flag), diag-12. Even the preset called conservative can erase meaning. |
| diag-q10 | Water, ماء; diag-13 | Standalone ء is preserved by every primary preset; do not silently delete it. |
| diag-q11 | A school, مدرسة; diag-15 | ة→ه conflates it with مدرسه (“his teacher”), diag-16. |
| diag-q12 | Identity card requirement; diag-05 | Another plain spelling control. |

Expected behavior is about exact transformations, preserved passage identities, valid scores, and inspectable rankings. **There is no expected positive aggregate gain.** A collision can create a tie without changing nDCG for a particular stable tie-break order. That still demonstrates information loss, but does not establish a population-level retrieval regression.

With only 16 passages, Recall@100 is largely a mechanics check. Publish no headline based on that number. Do not mix these 12 authored queries into MIRACL averages or use them to select the best normalization recipe before claiming confirmatory MIRACL results.
