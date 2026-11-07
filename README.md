# PDPA Compliance Assistant (RAG)

A question-answering assistant for Singapore's **Personal Data Protection Act (PDPA)**. It answers only from official PDPC documents and cites the document and page for every answer.

> General chatbots answer PDPA questions from memory, without sources. This assistant retrieves the relevant paragraphs from official guidance first, so every answer can be checked.

![Screenshot](screenshot.png)
<!-- Take a screenshot of an answer in the notebook, save it as screenshot.png in this folder -->

## Results

| Test | Result |
|---|---|
| Correct page retrieved (top 4 chunks) | **9 of 9 questions (100%)** |
| Off-topic questions correctly refused | **2 of 2** |

Measured with `pdpa_score.ipynb` on a hand-checked answer key in `test_questions.csv` (607 chunks from 4 documents, 345 pages).

**What testing revealed**

- **Meaning-based search works:** a question about a "data leak" retrieved the Data Breach guide, although the documents use the word "breach".
- **The answer key needed a fix:** two first-run "misses" retrieved Key Concepts p.143, which also states the breach notification timeframes. The answer key now accepts a correct page from either document (`also_correct` column).
- **Retrieval correct, generation wrong:** asked when to notify the PDPC, the model answered "within 24 hours" although its retrieved source (Key Concepts p.143) says no later than **3 calendar days**. The small local model can misstate numbers even with the right source, so answers should be checked against the cited page.
- **Off-topic questions score much lower:** about 0.44–0.53 similarity, versus about 0.86–0.88 for PDPA questions.
- **Data quality:** the Act, saved from the web page as PDF, lost spaces between some words.

## How it works

```
4 official PDPA documents (PDF)
   → split into ~300-word chunks with page numbers
   → each chunk turned into an embedding (nomic-embed-text)
   → saved as a NumPy matrix (normalised vectors)
Question
   → embedded the same way
   → cosine similarity against every chunk (one matrix multiplication)
   → 4 most similar chunks retrieved
   → prompt: rules + numbered chunks + question
   → llama3.2 writes an answer citing [1], [2]...
   → disclaimer added by code on every answer
```

**Design choices**

- **Runs locally with Ollama:** free, and documents never leave the computer.
- **NumPy vector search instead of a vector database:** a few hundred pages fit easily in memory, so one matrix multiplication finds the closest chunks; simpler, with no extra dependencies.
- **Page numbers kept as metadata:** so answers cite the exact page.
- **Overlapping chunks (50 words):** sentences at chunk borders aren't lost.
- **"I don't know" rule:** the assistant refuses questions its documents don't cover.
- **Disclaimer added by code, not the AI:** guaranteed on every answer.

## Documents used

| Document | Source |
|---|---|
| Personal Data Protection Act 2012 | [Singapore Statutes Online](https://sso.agc.gov.sg/Act/PDPA2012?WholeDoc=1) |
| Advisory Guidelines on Key Concepts | [PDPC](https://www.pdpc.gov.sg) |
| Guide on Managing and Notifying Data Breaches | [PDPC](https://www.pdpc.gov.sg) |
| Advisory Guidelines for NRIC Numbers | [PDPC](https://www.pdpc.gov.sg) |
| Advisory Guidelines for Selected Topics (planned) | [PDPC](https://www.pdpc.gov.sg) |

The PDFs are not included in this repository; download them from the official links (file names in `sources.csv`).

## How to run it

1. Install [Ollama](https://ollama.com), then run `ollama pull nomic-embed-text` and `ollama pull llama3.2`.
2. Put the PDFs in a `documents/` folder with the names listed in `sources.csv`.
3. Open `pdpa_week1.ipynb` in Jupyter and run all cells.
4. To measure accuracy, fill in `test_questions.csv` and run `pdpa_score.ipynb`.

## Limitations

- General information only, **not legal advice**.
- Covers 4 documents; questions outside them are refused.
- Small local model: it can misstate details such as deadlines, even with the right source.
- Small test set (9 + 2 questions).

## Next steps

- Fix the generation error: a prompt rule to copy numbers exactly, or a larger model; then score answer accuracy, not only retrieval
- Add the Selected Topics guidelines
- Web interface with Streamlit and a public demo link
- Larger test set (30 questions) and chunk-size experiments
- Chunking by legal section for more precise citations
- Answers in Bahasa Indonesia / Chinese via multilingual embeddings

*Personal learning project; not affiliated with the PDPC.*
