# PDPA Compliance Assistant (RAG)

A retrieval-augmented generation (RAG) assistant that answers questions about Singapore's **Personal Data Protection Act (PDPA)** using only official PDPC documents, and cites the document and page for every answer.

![Screenshot](screenshot.png)

## Why RAG?

A search engine returns links of mixed quality; a general AI chatbot answers from its training data, which may be outdated or from another country. A RAG assistant answers **only from an approved set of documents**, so an organisation gets:

- **Verifiable answers:** every answer cites the document and page.
- **Consistency:** every user gets answers from the same, current versions of the rules.
- **Safe refusals:** questions the documents don't cover are refused instead of guessed.
- **Privacy:** it runs locally, so internal policies could be added without uploading them to an outside service.

The same approach works for any rule-heavy domain, such as HR policies, MAS regulations or internal procedures.

## Documents used

5 official documents, **407 pages (about 3.6 MB)**, downloaded on 7 November 2026.

| Document | Pages | Source |
|---|---|---|
| Personal Data Protection Act 2012 | 125 | [Singapore Statutes Online](https://sso.agc.gov.sg/Act/PDPA2012?WholeDoc=1) |
| Advisory Guidelines on Key Concepts in the PDPA | 165 | [PDPC](https://www.pdpc.gov.sg/organisations/regulations-decisions/regulatory-guidance/advisory-guidelines-on-key-concepts-in-the-personal-data-protection-act) |
| Guide on Managing and Notifying Data Breaches | 40 | [PDPC](https://www.pdpc.gov.sg/organisations/resources/guidance-by-topic/data-breach-management-guide) |
| Advisory Guidelines for NRIC and other National Identification Numbers | 15 | [PDPC](https://www.pdpc.gov.sg/guidelines-and-consultation/2020/02/advisory-guidelines-on-the-personal-data-protection-act-for-nric-and-other-national-identification-numbers) |
| Advisory Guidelines on the PDPA for Selected Topics | 62 | [PDPC](https://www.pdpc.gov.sg/organisations/regulations-decisions/regulatory-guidance/advisory-guidelines-on-the-personal-data-protection-act-for-selected-topics) |

The PDFs are not included in this repository; file names are listed in `sources.csv`.

## How it works

```
PREPARE (once)
  5 PDFs ─► extract text per page ─► split into ~300-word chunks (50-word overlap, page number kept)
        ─► embed each chunk (nomic-embed-text, 768 numbers) ─► save as a NumPy matrix

ANSWER (per question)
  question ─► embed ─► cosine similarity with every chunk ─► top 4 chunks
           ─► best match below 0.69? ─► yes: refuse (no AI call)
                                     └► no: prompt = rules + chunks + question ─► llama3.2 answers
           ─► show answer + sources (document, page) + disclaimer
```

1. **Load:** read each PDF page by page with `pypdf` and clean the text.
2. **Chunk:** split pages into overlapping ~300-word pieces, keeping the page number for citations.
3. **Embed:** turn each chunk into a vector that represents its meaning (`nomic-embed-text` via Ollama).
4. **Retrieve:** embed the question and find the 4 most similar chunks with cosine similarity (NumPy).
5. **Refuse or answer:** if no chunk is similar enough, refuse; otherwise the model answers using only the retrieved chunks.
6. **Evaluate:** score retrieval, refusals and answers on a hand-checked test set.

**Stack:** Python, pypdf, NumPy, Ollama (`nomic-embed-text`, `llama3.2`), Jupyter. Runs fully on a laptop.

## Results

Tested on **30 hand-checked questions** (25 PDPA + 5 off-topic):

| Test | Result |
|---|---|
| Correct page retrieved (top 4 chunks) | **24 of 25 (96%)** |
| Off-topic questions refused cleanly | **5 of 5** (up from 4 of 5 before the similarity threshold) |
| Answers containing the key fact | **18 of 25 (72%)** |

**Key findings**

- **Search by meaning works:** a question about a "data leak" found the data breach guide, which never uses the word "leak".
- **The threshold is data-driven:** every PDPA question scored at least 0.729 similarity and every off-topic question at most 0.648, so 0.69 sits in the gap.
- **Retrieval right, answer wrong:** the model said "72 hours" for breach notification although its retrieved source says **3 calendar days**. A prompt rule to copy figures exactly did not fix it, which points to the small model's limits. This is why every answer shows its sources.

## How to run it

1. Install [Ollama](https://ollama.com) and run `ollama pull nomic-embed-text` and `ollama pull llama3.2`.
2. Put the PDFs in a `documents/` folder using the names in `sources.csv`.
3. Run `pdpa_assistant.ipynb` to build the index and ask questions, then `pdpa_score.ipynb` to reproduce the results.

## Limitations

- General information only, **not legal advice**.
- The small local model misses the key fact in about 1 in 4 answers; check the cited page.
- The threshold was tuned on the same 30 questions it was tested on.

## Future work

1. **Larger model:** test `llama3.1:8b` on the same questions to reduce answer errors.
2. **Better retrieval:** add `nomic-embed-text`'s query/document prefixes and keyword search.
3. **Web app:** a Streamlit interface with a public demo link.

*Personal learning project; not affiliated with the PDPC.*
