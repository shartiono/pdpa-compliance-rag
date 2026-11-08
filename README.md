# PDPA Compliance Assistant (RAG)

A question-answering assistant for Singapore's **Personal Data Protection Act (PDPA)**. It answers only from official PDPC documents and cites the document and page for every answer.

> General chatbots answer PDPA questions from memory, without sources. This assistant retrieves the relevant paragraphs from official guidance first, so every answer can be checked.

## Objective

To build and evaluate a **retrieval-augmented generation (RAG)** assistant for compliance questions, using Singapore's PDPA as the example domain. The goal is to show how an organisation could give staff quick, **sourced** answers from approved regulatory documents, and to measure honestly how well it works:

- answer only from official, approved documents
- cite the exact document and page, so every answer can be verified
- refuse questions the documents don't cover, instead of guessing
- measure retrieval and answer quality on a hand-checked test set, and test whether changes actually help

The same approach applies to any rule-heavy domain: HR policies, financial regulations (e.g. MAS notices), workplace safety, or a company's internal procedures.

![Screenshot](screenshot.png)

## Sample answers

Real outputs from the notebooks, showing the three behaviours that matter.

**1. A correct answer with citations**

> **Q:** Can a shop collect a customer's full NRIC number for a lucky draw?
>
> **A:** No, a shop cannot collect a customer's full NRIC number for a lucky draw. [1] [4] According to the guidelines, a shop should not collect the NRIC number of customers for purposes such as lucky draws, unless it is necessary to accurately establish or verify the identity of the individual to a high degree of fidelity. The shop should consider using alternative identifiers, such as the customer's mobile number, email address, or partial NRIC number (up to the last 3 numerical digits and checksum).
>
> **Sources:** [1] NRIC Guidelines, p.12 (similarity 0.839) · [4] NRIC Guidelines, p.13 (0.700)

**2. An off-topic question, before and after the similarity threshold**

> **Q:** What is Singapore famous for?
>
> **Before:** "Singapore is famous for being a digital economy hub, with a strong focus on innovation and technology. [1] I don't have information on that in my PDPA documents." *(an unsupported half-answer, then a refusal)*
>
> **After:** "I don't have information on that in my PDPA documents." *(refused in code: the best match scored 0.617, below the 0.69 threshold, so the AI was never asked)*

**3. A known error: right source, wrong figure**

> **Q:** When must an organisation notify the PDPC about a notifiable data breach?
>
> **A:** "...within 72 hours of determining that the breach is notifiable." *(an earlier run said "24 hours")*
>
> **Correct:** as soon as practicable, and **no later than 3 calendar days** after determining the breach is notifiable (Data Breach Guide p.25; Key Concepts p.143). Both pages were retrieved; the small model misstated the figure. This is why every answer shows its sources.

## Why RAG, instead of a search engine or a general AI chatbot?

For one person asking an occasional question, a search engine or a general AI assistant (ChatGPT, Claude, Gemini) with an uploaded PDF works well. RAG matters when an **organisation** needs answers it can control, share and trust.

| | Search engine | General AI chatbot | This RAG assistant |
|---|---|---|---|
| **Gives a direct answer** | No, a list of links to read | Yes | Yes |
| **Sources it uses** | Mixed: official pages, forums, ads, other countries' rules | Training data, plus whatever files a user uploads or pages it searches | Only the approved document set |
| **Same sources for every user** | No | No, depends on what each user uploads or enables | Yes, documents are loaded once, centrally |
| **Exact citation (document + page)** | The link only | Varies | On every answer |
| **When the documents don't cover it** | Shows something anyway | May answer from general knowledge | Refuses (in code, below a similarity threshold) |
| **Accuracy measured** | No | Not for your specific documents | Yes, on a hand-checked test set |
| **Keeping up to date** | Depends on the websites | Depends on the model's training date or the user re-uploading | Replace the document and rebuild the index |
| **Confidential internal documents** | Not possible | Must be uploaded to an outside service | Can run fully locally (this project uses Ollama) |

**Where an organisation benefits most**

- **Consistency:** every staff member gets answers from the same approved, current versions of the rules.
- **Verifiability:** compliance and audit teams can check each answer against the cited page.
- **Combining public and internal rules:** a company could add its own data-handling policy next to the PDPA, which it may not be allowed to upload to a public chatbot.
- **Control:** the organisation decides the documents, the refusal behaviour, the disclaimer, and where the system runs.

**Honest trade-off:** a large online chatbot usually writes more fluent answers than the small local model used here; this project's answer accuracy is about 72%. The value of RAG is not a smarter model but **controlled sources, citations and measurable behaviour**, and the retrieval pipeline works the same with a larger model.

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

## Project files

| File | What it is |
|---|---|
| `pdpa_week1.ipynb` | The assistant: loads the PDFs, chunks, embeds, retrieves and answers, step by step with explanations |
| `pdpa_score.ipynb` | The evaluation: retrieval, off-topic and answer-accuracy tests, plus the similarity report |
| `test_questions.csv` | The hand-checked answer key: 30 questions with correct pages and expected key facts |
| `test_results_baseline.csv` | Results for every question, original version |
| `test_results_improved.csv` | Results for every question, after the changes |
| `sources.csv` | Titles, official links and download dates of the documents |
| `requirements.txt` | Python libraries needed |

**Tech stack:** Python, NumPy, pypdf, Ollama (`nomic-embed-text` embeddings, `llama3.2` answers), Jupyter.

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
