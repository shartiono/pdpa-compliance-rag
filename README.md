# PDPA Compliance Assistant (RAG)

A question-answering assistant for Singapore's **Personal Data Protection Act (PDPA)**. It answers only from official PDPC documents and cites the document and page for every answer.

> General chatbots answer PDPA questions from memory, without sources. This assistant retrieves the relevant paragraphs from official guidance first, so every answer can be checked.

![Screenshot](screenshot.png)
<!-- Take a screenshot of an answer in the notebook, save it as screenshot.png in this folder -->

## Results

| Test | Result |
|---|---|
| Correct page retrieved (top 4 chunks) | **X of Y questions (Z%)** |
| Off-topic questions correctly refused | **X of Y** |

Measured with `pdpa_score.ipynb` on hand-checked test questions in `test_questions.csv`.

## How it works

```
5 PDPC documents (PDF)
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
| Advisory Guidelines for Selected Topics | [PDPC](https://www.pdpc.gov.sg) |

The PDFs are not included in this repository; download them from the official links (file names in `sources.csv`).

## How to run it

1. Install [Ollama](https://ollama.com), then run `ollama pull nomic-embed-text` and `ollama pull llama3.2`.
2. Put the PDFs in a `documents/` folder with the names listed in `sources.csv`.
3. Open `pdpa_week1.ipynb` in Jupyter and run all cells.
4. To measure accuracy, fill in `test_questions.csv` and run `pdpa_score.ipynb`.

## Limitations

- General information only, **not legal advice**.
- Covers 5 documents; questions outside them are refused.
- Small local model: answers can be less polished than large online models.

## Next steps

- Web interface with Streamlit and a public demo link
- Larger test set (30 questions) and chunk-size experiments
- Chunking by legal section for more precise citations
- Answers in Bahasa Indonesia / Chinese via multilingual embeddings

*Personal learning project; not affiliated with the PDPC.*
