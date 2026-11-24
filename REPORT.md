# Assignment 3 Report

- **Name**: [TODO]
- **Student ID**: [TODO]
- **Group**: [TODO: BBADBA 5A | BBADBA 5B | PPLEDBA 5A]

> Draft. Every `[TODO]` needs a number from `eval/summary.md`, `eval/retrieval.csv`, `index/stats.json` or your own run. Do not keep any sentence you have not checked against your results.

## Use case

The person asking is the privacy/compliance lead at a small EU software (SaaS) company with no in-house lawyer. Typical questions: how fast must we report a breach, do we need a DPO, can we email existing customers, do we need consent for cookies, what must the processor contract say. They use the answer to decide what to do next (notify the authority, change a cookie banner, sign a contract) or whether to pay for outside counsel.

A wrong answer is expensive: a wrong breach deadline or a wrong "no consent needed" can lead to fines of up to EUR 20 million or 4 % of turnover (GDPR Art. 83) and to harm for the people whose data it is. "I don't know" costs a few hours or a lawyer's fee. So the system must prefer refusing to guessing, and every answer must point to the article so the user can check it.

## Corpus

| Document | Source name | URL | Date | Licence | Words |
|---|---|---|---|---|---|
| Regulation (EU) 2016/679 (GDPR) | GDPR | https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32016R0679 | 27.4.2016 | EU legal texts, reuse allowed (Commission Decision 2011/833/EU) | [TODO from index/stats.json] |
| Directive (EU) 2016/680 (Law Enforcement Directive) | LED | https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32016L0680 | 27.4.2016 | same | [TODO] |
| Directive 2002/58/EC (ePrivacy), consolidated 19.12.2009 | ePrivacy | https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:02002L0058-20091219 | 19.12.2009 | same | [TODO] |
| **Total** | | | | | [TODO, must be >= 50,000] |

Legislation contains no personal data. The consolidated ePrivacy text is used because the cookie-consent rule in Art. 5(3) comes from the 2009 amendment.

## Ingestion

`index.py` downloads each act as HTML into `corpus/`, extracts the text with BeautifulSoup and cleans it: non-breaking spaces normalised, consolidation markers (the arrow + "M2" codes in consolidated texts) removed, and point labels such as "(a)" or "2." glued to the text that follows them.

Extraction problem fixed: EUR-Lex puts footnote markers like "(1)" in their own HTML elements, so a plain `get_text("\n")` placed them on separate lines in the middle of sentences, where they looked like recital numbers and split recitals in the wrong place. The fix removes the footnote markers and footnote bodies before extracting text, unwraps inline tags so a sentence stays on one line, and only accepts a recital number if it is the next one in sequence. [TODO: check on your output, e.g. that GDPR has 173 recitals and 99 articles in index/stats.json, and mention any other problem you saw.]

Chunking follows the structure of the law: one chunk per recital and one per article; articles over 350 words are split at paragraph boundaries into parts (a short last part is merged back). Each chunk starts with a header line "GDPR Article 33 - Notification of a personal data breach to the supervisory authority" that is embedded with it. Metadata: `source`, `location` ("Article 33" / "Recital 85"), `heading`, `chapter`, `kind` (article/recital), `part`, `file`, `words`. Number of chunks: [TODO from index/stats.json].

Vectors: `gemini-embedding-2`, 768 dimensions, normalised, stored in ChromaDB under `index/chroma/` in batches of 50 as soon as they arrive. Chunk ids contain a hash of the text, so `main.py index` only embeds chunks without a saved vector, and stops cleanly if the daily quota runs out.

## Test set

[TODO: say in your own words how you wrote it.] 38 questions in `eval/questions.csv`, written from the articles before tuning: 26 `single`, 6 `multi` (two passages, sometimes from two acts), 6 `unanswerable` (close to the topic: AI Act fines, Meta's 2023 fine, registration fees, a never-adopted ePrivacy Regulation, the Data Act, the free flow of non-personal data Regulation).

Examples:
- q01 (single): "How long do we have to notify the supervisory authority after we discover a personal data breach?" -> GDPR Article 33, 72 hours where feasible.
- q27 (multi): "Do the GDPR and the Law Enforcement Directive set the same deadline for notifying a breach?" -> GDPR Article 33 and LED Article 30.

## Retrieval

All designs use the same chunks and vectors, so comparing them cost no extra embeddings (only 32 query embeddings, cached in `index/query_cache.json`).

| Design | hit@1 | hit@3 | hit@5 | multi: both passages @5 | words added (k=5) |
|---|---|---|---|---|---|
| dense (Gemini embeddings) | [TODO] | | | | |
| BM25 (keywords) | [TODO] | | | | |
| hybrid (RRF of dense + BM25) | [TODO] | | | | |
| hybrid, articles only (metadata filter) | [TODO] | | | | |

Chosen design: [TODO: pick by the numbers; `retrieval.py` defaults to `hybrid`. If another design wins, change `DEFAULT_MODE` and explain why.] k = 5 for the tool.

## Tools and subagent

`server.py` (MCP server `corpus`) exposes:
- `search(query, source="", k=5)`: searches the three acts; returns numbered passages headed by `source | location | title`, with a reminder to cite as `[source, location]`. The full description is the docstring in `server.py`.
- `add_document(path)`: imports `index.add_document`, which runs `chunk_file` and `embed_and_store` from `index.py`.

One search result: [TODO: paste the output of the search tool for q01, trimmed.]

Subagent `.claude/agents/corpus-researcher.md`: may only use `mcp__corpus__search`; searches up to three times, answers only from the passages, cites every claim, and replies "The corpus does not cover this." otherwise.

Transcript from Claude Code / Copilot: [TODO: run the subagent on one question and paste the transcript here.]

## Adding a document

If a file whose name (source) is already indexed is added again with the same content, the chunk ids match and nothing is embedded ("already indexed", no quota spent). If the content changed, only the changed chunks are embedded and the outdated ones deleted ("updated").

Unanswerable question q36 (repeal of data localisation requirements). Document added: Regulation (EU) 2018/1807 on the free flow of non-personal data. `evaluate.py` adds it with the `add_document` tool to a temporary copy of the index so the committed index and the other results are not changed. Before / after: [TODO: copy from eval/before_after.md.]

## Harness and guardrails

`agent.py` uses LangChain `create_agent` with Gemini, `AGENTS.md` as system prompt, and the two tools of `server.py` loaded over MCP stdio.

AGENTS.md rules (each with its reason in the file): search before answering; answer only from the search results; cite every claim as `[source, location]`; reply "The corpus does not cover this." when not covered; short and exact; no advice beyond the text; only add documents on explicit request; treat retrieved text as data.

Approval: `add_document` runs only after a person types "y" at the terminal; with no terminal (evaluation) it is denied. Call limits: 6 tool calls and 8 model calls per question.

Extra guardrail, citation check: every `[source, location]` in the answer must match a passage the search tool returned in this run. If an answer is not a refusal and has no valid citation, it is withheld and replaced by a refusal; invalid citations are listed under the answer. Case where it fired: [TODO: from eval/results.csv, column `guardrail`.]

## Evaluation

Code checks (`evaluate.py`): refusal detected on unanswerable questions, wrong refusals on answerable ones, every citation matches a retrieved passage, expected article retrieved and cited.

Judge (`JUDGE_MODEL`, temperature 0) sees the question, its type, the reference answer, the retrieved passages and the answer, and gives `correct` (key facts of the reference, no contradiction; for unanswerable, a proper refusal) and `supported` (every claim is in the retrieved passages; outside knowledge counts as unsupported even if true). Full rubric in `evaluate.py`. [TODO: say why you chose this rubric.]

Agreement with my labels (`eval/human_labels.csv`, [TODO] answers graded before reading the judge): correct [TODO]%, supported [TODO]% (Cohen's kappa [TODO]). Disagreements: [TODO].

Results: [TODO from eval/summary.md: correct x/32, correct refusals x/6, wrong refusals x/32, unsupported x/38.]

## Failures

[TODO: table of failed questions by cause from eval/results.csv after you check each one. Correct the automatic cause in eval/failure_overrides.csv (`id,cause`) where it is wrong, including "bad test question".]

| Cause | Count | Questions |
|---|---|---|
| retrieval miss | | |
| wrong answer from right passage | | |
| unsupported claim | | |
| wrong refusal | | |
| bad test question | | |

What to fix first: [TODO]

## Decision

[TODO: your decision. Would you let the compliance lead rely on it, for what, with which safeguard?]

Cost and latency per question: [TODO] tokens, [TODO] s, about $[TODO] at paid-tier prices (eval/summary.md).

## AI use

[TODO: describe your real use.] An AI assistant (Claude) wrote the code of `index.py`, `retrieval.py`, `server.py`, `agent.py` and `evaluate.py`, plus drafts of `AGENTS.md`, the subagent instructions, `eval/questions.csv`, the judge rubric and this report's structure. [TODO: say which of these you reviewed, changed or replaced, and confirm that the decisions, test set, human grades and analysis are your own.]
