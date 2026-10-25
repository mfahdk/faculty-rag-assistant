# AI-Based Faculty Information Management System
​
A faculty-record management platform centered on a **retrieval-augmented generation (RAG) assistant**. It combines uploaded-document context with structured activity records to answer questions and generate academic report drafts using Ollama-hosted language models.
​
## Contents
- [Overview](#overview)
- [Core features](#core-features)
- [System demo](#system-demo)
- [System architecture](#system-architecture)
- [RAG pipeline](#rag-pipeline)
- [Models and generation](#models-and-generation)
- [Supporting application](#supporting-application)
​
## Overview
​
Faculty activity data and supporting evidence often live in separate places. This system brings teaching, research, publication, and service records together with document-based retrieval in a conversational interface.
​
The assistant uses two complementary sources: **structured records** fetched from MySQL through the application API, and **document passages** retrieved from per-user FAISS indexes. These sources are assembled into the model's prompt alongside recent conversation history. The approach supplies context at inference time rather than fine-tuning a model on faculty data.
​
## Core features
- **Document-based questions:** retrieve relevant passages from uploaded files and use them as context for answers.
- **Faculty-record queries:** combine teaching, research, publication, and service summaries with record counts.
- **Report drafts:** use a dedicated Mistral model for report requests, with frontend DOCX/PDF export and report history.
- **Conversational record creation:** extract activity details into JSON and submit them to the application API.
- **Optional web search:** answer from DuckDuckGo results through a separate web-only path.
- **Supporting portal:** manage activities, documents, and dashboard views through a web interface.
​
## System demo
​
[Watch the system demo](https://example.com/faculty-system-demo)
​
*Replace this placeholder with the demo video URL.*
​
## System architecture
​
![High-level system design showing the web portal, AI platform, documents, and database](https://github.com/mfahdk/faculty-rag-assistant/blob/main/devl1.png)
​

*Conceptual system overview. The diagram's Llama 3.1 label predates the current Llama 3.2 3B setting; the SSO flow remains a design reference rather than a verified integration.*
​
```text
Web interface: PHP / JavaScript
    |
    |-- Application API: Node.js / Express
    |       |-- MySQL: faculty records and document metadata
    |       |-- Local files: supporting documents and reports
    |       |-- Internal endpoints: records and document access
    |
    |-- AI service: Python / FastAPI
            |-- Text extraction and overlapping chunks
            |-- Ollama embeddings --> per-user FAISS index
            |-- Retrieved passages + faculty records + history
            |-- Ollama language model --> answer / report draft
```
​
| Component       | Technologies                     |
| :-------------- | :------------------------------- |
| AI API          | Python, FastAPI, HTTPX           |
| LLM inference   | Ollama, LangChain Ollama         |
| Vector search   | FAISS, NumPy                     |
| Text extraction | PyMuPDF, Mammoth, python-pptx    |
| Backend         | Node.js, Express, MySQL, Multer  |
| Frontend        | PHP, JavaScript                  |
​
## RAG pipeline
### 1. Load the user's documents
​
For requests with a valid session token, the AI service derives the user ID from the token and fetches that user's uploads through the backend's shared-secret-protected internal API. Documents are loaded when the user's index is not already cached.
​
PDF, DOCX, and PPTX files have dedicated text extractors. Other files fall back to UTF-8 decoding; this does not provide OCR or reliable support for binary formats such as legacy DOC files. Generated-report categories are excluded from indexing to avoid reusing previous AI output as source evidence.
​
### 2. Split text into overlapping chunks
​
The service uses **700-character chunks with 100-character overlap**, retaining only chunks longer than 50 characters:
​
```python
def chunk_text(text: str, size: int = 700, overlap: int = 100) -> list:
    chunks = []
    for i in range(0, len(text), size - overlap):
        chunk = text[i:i + size].strip()
        if len(chunk) > 50:
            chunks.append(chunk)
    return chunks
```
​
Overlap preserves some context across boundaries. Each chunk is prefixed with its source filename before embedding.
​
### 3. Embed and index
​
Ollama converts document chunks into vectors. The service stores them in an exact-search **FAISS `IndexFlatL2`** index, with separate document and index dictionaries for each user:
​
```python
emb = np.array(model_emb.embed_documents(docs))
index = faiss.IndexFlatL2(emb.shape[1])
index.add(emb)
​
user_docs[uid] = docs
user_indexes[uid] = index
user_loaded.add(uid)
```
​
The index uses squared Euclidean distance: smaller distances indicate closer vector matches. Indexes live in process memory, and a per-user asynchronous lock prevents duplicate concurrent document loads within that process.
​
### 4. Retrieve passages for a question
​
The question is embedded with the same embedding model and searched against the user's index:
​
```python
q_emb = np.array(model_emb.embed_query(query)).reshape(1, -1)
distances, idxs = index.search(q_emb, min(k, len(docs)))
```
​
The answer path requests **up to ten matches** and includes the **first five** in the prompt. There is no relevance-distance cutoff or reranking stage.
​
### 5. Combine document and database context
​
The service fetches publication, teaching, research, and service records from the application API, then builds a prompt containing:
​
- Faculty-record summaries and total record counts.
- Retrieved document passages with source labels.
- The current date and September-to-August academic year.
- The latest six conversation messages and the current question.
​
Structured records are fetched directly rather than searched through FAISS. Prompt instructions tell the model to use the supplied counts, distinguish teaching from course enrollment, and avoid inventing facts. These instructions support grounding but do not guarantee factual answers or citations.
​
### 6. Generate the response
​
Ordinary questions use the chat model; report requests switch to the report model:
​
```python
active_model = report_model if query.report_mode else model
result = active_model.invoke(prompt)
clean_result = re.sub(r'<think>.*?</think>', '', result, flags=re.DOTALL).strip()
return {"answer": clean_result, "web_results": 0}
```
​
The frontend displays answers or converts report Markdown into downloadable documents. Report mode reuses the ordinary question-answering prompt rather than a separate report template.
​
### Retrieval boundaries
- **Cache freshness:** indexes are not persisted or automatically refreshed after uploads or deletions; cached content can become stale until rebuilt or the process restarts.
- **Extraction:** scanned PDFs and images require OCR, which is not implemented here.
- **OneDrive:** the backend separately stores OneDrive chunks and MiniLM embeddings in MySQL. The AI service does not read that store, so those files are not automatically part of this RAG pipeline.
- **Answer quality:** retrieved passages may be irrelevant, and long context or the report output limit can omit information. Answers and reports need review.
​
## Models and generation
| Role              | Ollama model                 |
| :---------------- | :--------------------------- |
| Chat / extraction | `llama3.2:3b`                |
| Embeddings        | `mxbai-embed-large`          |
| Reports           | `mistral:7b-instruct-q4_K_M` |​


**Generation settings**
​
- **Chat:** temperature `0.1`; context window `8192` tokens.
- **Reports:** temperature `0.0`; context window `4096` tokens; output limit `1024` tokens.
- **Embeddings:** the same model embeds both document chunks and questions.

​
The supplied environment settings select **Llama 3.2 3B** for chat and activity extraction and **mxbai-embed-large** for document and query embeddings. The report model is selected separately in the Python code. These settings identify the configured models, not a verified running deployment.
​
**Web-search mode** retrieves up to five DuckDuckGo results and passes their titles, snippets, and URLs to the chat model. It returns before the private-data answer path, so web results are not combined with faculty records or retrieved documents. Search questions are sent to an external provider.
​
**Activity creation** uses save-related phrases to trigger JSON extraction, checks the action and a required field, and sends the payload to the backend. Publication and teaching extraction have the clearest prompt support; research/service field mapping and follow-up handling need refinement. Successful writes occur without a separate confirmation step.
​
## Supporting application
​
![Backend subsystem architecture showing the API, activity modules, integrations, database, and file storage](https://github.com/mfahdk/faculty-rag-assistant/blob/main/subsystem_backend.png)
​
*Supporting backend design. Encryption applies only to `General` uploads; the identity-provider connection shown requires a verified trust boundary.*

​
The web portal handles chat, activity management, uploads, and report export. The Express API provides MySQL-backed records, a 25 MB upload limit, user-facing file ownership checks, and internal endpoints used by the AI service. Supporting integrations include Semantic Scholar publication import and separate OneDrive ingestion.
​
Private-data access in the AI service uses a token-derived user ID, but secure deployment still requires fixing the backend's caller-supplied-UID token issuance. Anonymous `/ask` requests can reach generation or web search without private records. This is a project implementation, not a claim of production readiness.
​
