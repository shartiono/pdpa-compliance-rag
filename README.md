# PDPA Compliance Assistant (RAG)

A question-answering assistant for Singapore's **Personal Data Protection Act (PDPA)**. It answers only from official PDPC documents and cites the document and page for every answer.

> General chatbots answer PDPA questions from memory, without sources. This assistant retrieves the relevant paragraphs from official guidance first, so every answer can be checked.

![Screenshot](screenshot.png)

## Results

30 hand-checked test questions (25 PDPA + 5 off-topic), comparing the original version with the version after two changes:

| Test | Original | After changes |
|---|---|---|
| Correct page retrieved (top 4 chunks) | **24 of 25** | **24 of 25** |
| Off-topic questions refused cleanly | **4 of 5** | **5 of 5** |
| Answers containing the key fact | **19 of 25** | **18 of 25** |

Measured with `pdpa_score.ipynb` against the answer key in `test_questions.csv`. Both versions are scored with the same answer key. Retrieval is identical in both; the changes affect only what happens after retrieval.

**What testing revealed**

- **Meaning-based search works:** a question about a "data leak" retrieved the Data Breach guide, although the documents use the word "breach".
- **The answer key needed fixes, not just the model:** some "misses" retrieved a second document that also answers the question (e.g. Key Concepts p.143 states the breach notification timeframes), and some "wrong" answers were correct but worded differently (e.g. "Yes, appoint a DPO under Section 11(3)"). The key was corrected only where the answer was genuinely right; the `also_correct` column accepts a correct page from another document.
- **Retrieval correct, generation wrong:** asked when to notify the PDPC, the model answered "within 24 hours", later "within 72 hours", although its retrieved sources say no later than **3 calendar days**. The small local model can misstate figures even with the right source in front of it.
- **Half-answers to off-topic questions:** asked "What is Singapore famous for?", the model wrote an unsupported sentence about Singapore's digital economy and *then* said it had no information. The first version of the test counted this as a pass because it only looked for the refusal phrase; the test now requires the answer to be **only** the refusal.
- **Off-topic questions score much lower:** every PDPA question's best match scored at least **0.729** similarity; every off-topic question scored at most **0.648**.
- **Remaining retrieval miss:** "Is anonymised data covered by the PDPA?" retrieves pages about *how* to anonymise data, not the page stating that anonymised data is not personal data.
- **Data quality:** the Act, saved from the web page as PDF, lost spaces between some words.

## Changes tested (before → after)

Two changes, both aimed at problems found in testing. One worked; one did not.

1. **Similarity threshold — kept.** If even the best-matching chunk scores below **0.69**, the code refuses *without asking the AI*, so it can't half-answer. 0.69 is the midpoint of the gap between the lowest PDPA question (0.729) and the highest off-topic question (0.648, "What is the GST rate in Singapore?"), leaving a margin on both sides. Clean refusals: **4 → 5 of 5**, with no PDPA question blocked. It also skips an unnecessary model call.
2. **Copy-numbers rule — no measurable benefit.** A prompt rule telling the model to copy numbers, deadlines and amounts exactly. It did not fix the notification deadline ("72 hours" persisted), fixed one answer (the NRIC lucky-draw answer now mentions a partial NRIC), and broke two that were correct before (it wrongly refused the physical-NRIC question and said the PDPA applies to business contact information). Net: **19 → 18 of 25**, which is no real change. Lesson: a small model is sensitive to prompt wording, and its main remaining errors are reasoning errors (anonymised data, employee consent) that a rule about numbers cannot fix.

## How it works

```
5 official PDPA documents (PDF)
   → split into ~300-word chunks with page numbers
   → each chunk turned into an embedding (nomic-embed-text)
   → saved as a NumPy matrix (normalised vectors)
Question
   → embedded the same way
   → cosine similarity against every chunk (one matrix multiplication)
   → 4 most similar chunks retrieved
   → if the best match is below 0.69 similarity: refuse (no AI call)
   → prompt: rules + numbered chunks + question
   → llama3.2 writes an answer citing [1], [2]...
   → disclaimer added by code on every answer
```

**Design choices**

- **Runs locally with Ollama:** free, and documents never leave the computer.
- **NumPy vector search instead of a vector database:** a few hundred pages fit easily in memory, so one matrix multiplication finds the closest chunks; simpler, with no extra dependencies. (ChromaDB was tried first but crashed on this Windows/Anaconda setup.)
- **Page numbers kept as metadata:** so answers cite the exact page.
- **Overlapping chunks (50 words):** sentences at chunk borders aren't lost.
- **Similarity threshold before the AI:** weak matches are refused in code, so off-topic questions never reach the model.
- **Disclaimer added by code, not the AI:** guaranteed on every answer.

## Documents used

| Document | Source |
|---|---|
| Personal Data Protection Act 2012 | [Singapore Statutes Online](https://sso.agc.gov.sg/Act/PDPA2012?WholeDoc=1) |
| Advisory Guidelines on Key Concepts | [PDPC](https://www.pdpc.gov.sg) |
| Guide on Managing and Notifying Data Breaches | [PDPC](https://www.pdpc.gov.sg) |
| Advisory Guidelines for NRIC Numbers | [PDPC](https://www.pdpc.gov.sg) |
| Advisory Guidelines for Selected Topics | [PDPC](https://www.pdpc.gov.sg) |

The PDFs are not included in this repository; download them from the official links (file names in `sources.csv`).

## How to run it

1. Install [Ollama](https://ollama.com), then run `ollama pull nomic-embed-text` and `ollama pull llama3.2`.
2. Put the PDFs in a `documents/` folder with the names listed in `sources.csv`.
3. Open `pdpa_week1.ipynb` in Jupyter and run all cells.
4. To measure accuracy, run `pdpa_score.ipynb` (set `IMPROVED` to `False` or `True` to compare versions).

## Limitations

- General information only, **not legal advice**.
- Covers 5 documents; questions outside them are refused.
- Small local model: about 1 in 4 answers misses or misstates the key fact, even when the right page is retrieved. Always check the cited page.
- Test set of 30 questions; the threshold is tuned on the same set, so a separate set of new questions would confirm it holds.
- The answer check looks for key phrases, so wrong answers were also reviewed by hand.

## Next steps

- Test a larger model (e.g. `llama3.1:8b`) on the same 30 questions, since prompt rules didn't fix the reasoning errors
- Improve retrieval for definition questions such as anonymised data
- Web interface with Streamlit and a public demo link
- A separate validation set for the threshold, and chunk-size experiments
- Answers in Bahasa Indonesia / Chinese via multilingual embeddings

*Personal learning project; not affiliated with the PDPC.*
