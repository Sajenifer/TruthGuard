# TruthGuard

TruthGuard is a private, account-based document Q&A and adversarial verification application. It retains the existing FastAPI + React/Vite stack and uses MongoDB for user accounts, document metadata, extracted passages, saved summaries, conversations, and verification reports.

## Implementation plan

1. Add MongoDB-backed accounts, hashed passwords, HTTP-only sessions, and owner-scoped API access.
2. Add private PDF, DOCX, and TXT ingestion, source-backed metadata extraction/review, passage indexing, and safe file delivery.
3. Separate document summarization from question answering and adversarial verification.
4. Exercise authentication, ownership, ingestion, summary persistence, verification, and production frontend build.

## Local setup

Requirements: Python 3.12, Node.js/npm, and a running MongoDB server. The default MongoDB URI is `mongodb://127.0.0.1:27017`.

1. Create a Python environment and install backend dependencies:

   ```powershell
   py -3.12 -m venv .venv
   .\.venv\Scripts\Activate.ps1
   python -m pip install -r requirements.txt
   ```

2. Configure local environment values:

   ```powershell
   Copy-Item .env.example .env
   ```

   Set `MONGO_URI` for your MongoDB deployment. For production, use a restricted MongoDB user, a strong URI-held credential, HTTPS, and `COOKIE_SECURE=true`. Do not commit `.env`.

3. Install frontend dependencies:

   ```powershell
   cd frontend
   npm install
   cd ..
   ```

4. Start the backend and frontend in separate terminals:

   ```powershell
   .\.venv\Scripts\python -m uvicorn backend.app.main:app --reload --host 127.0.0.1 --port 8000
   ```

   ```powershell
   cd frontend
   npm run dev
   ```

5. Open `http://localhost:5173`, create an account, and upload your own documents. API documentation is at `http://localhost:8000/docs`.

## Configuration

| Variable | Purpose |
| --- | --- |
| `MONGO_URI` | MongoDB connection URI; keep credentials in the untracked `.env` |
| `MONGO_DATABASE` | MongoDB database name (`truthguard` by default) |
| `SESSION_COOKIE_NAME` | Name of the opaque session cookie |
| `SESSION_TTL_SECONDS` | Session lifetime (604800 seconds by default) |
| `COOKIE_SECURE` | Set `true` behind HTTPS; leave `false` for local HTTP development |
| `CORS_ORIGINS` | Explicit frontend origins permitted to send credentialed API requests |
| `UPLOAD_DIR` | Private local file directory |
| `MAX_UPLOAD_SIZE_MB` | Upload limit (10 MB by default) |
| `OCR_ENABLED` | Enable scanned-PDF OCR attempts (`false` by default) |
| `TESSERACT_CMD` | Optional absolute path to the Tesseract executable |
| `OCR_MIN_TEXT_CHARS` | Minimum extracted PDF characters before a page set is considered text-readable |
| `LLM_PROVIDER` | `mock` for local deterministic processing or `openai` for source-checked metadata extraction and optional semantic passage reranking |
| `OPENAI_API_KEY` | Optional provider key; used only on the backend and never sent to the frontend |
| `OPENAI_MODEL` | OpenAI model used for optional metadata extraction |
| `OPENAI_EMBEDDING_MODEL` | Embedding model used to rerank BM25 candidates when `LLM_PROVIDER=openai` and an API key is configured |
| `VITE_API_BASE_URL` | Optional frontend API URL; local development follows the UI hostname on port 8000, while deployments default to same-origin `/api` requests |

## Pages

- `/` landing page
- `/signup` and `/signin`
- `/dashboard`
- `/documents`, `/documents/upload`, and `/documents/:id`
- `/verify` and `/verification/:id`
- `/ask`, `/history`, and `/profile`

Application pages and APIs other than health and authentication are protected by an HTTP-only session and scoped to the signed-in user's MongoDB ID.

## API

- `POST /api/auth/signup`, `POST /api/auth/signin`, `POST /api/auth/logout`
- `GET/PATCH /api/auth/me`
- `GET /api/dashboard`
- `GET /api/documents`, `POST /api/documents/manual`, `POST /api/documents/upload`
- `GET/PATCH/DELETE /api/documents/{id}`; document detail supports `passage_page`, `passage_page_size` (up to 10), and `passage_search`
- `POST /api/documents/{id}/retry` and `POST /api/documents/{id}/confirm`
- `POST /api/documents/{id}/summary`
- `GET /api/documents/{id}/file` (owner-authorized)
- `POST /api/ask`, `POST /api/verifications`
- `GET /api/verifications/{id}`, `GET /api/history`

## Data and privacy

- MongoDB stores account hashes, metadata, processing status, provenance, extracted text as owner-scoped passages, saved summaries, conversations, and reports. Passwords are bcrypt-hashed; file paths and checksums are not returned to clients.
- Source files are stored under the configured upload directory using generated filenames. Downloads pass through an ownership-checked API.
- Upload processing is idempotent per owner and file checksum. Retries retain the same document ID and replace its passage index rather than making duplicate document records.
- Uploaded metadata is extracted deterministically from explicit labels, headings, and file properties. An optional OpenAI metadata pass only accepts values accompanied by a quoted source span found in the document. Users review and confirm values before the document becomes ready; missing values remain unknown.
- OCR is optional. Install the Tesseract executable, set `OCR_ENABLED=true` and (if needed) `TESSERACT_CMD` to enable scanned-PDF OCR. If Tesseract is missing or OCR finds no text, the uploaded file is retained and the UI displays the limitation and retry action.
- The prior SQLite database and uploaded files are left untouched. The new account-based library does **not** automatically adopt old SQLite documents because their owner cannot be established safely; re-upload them under your account when appropriate.
- No demo documents, demo accounts, or static reports are seeded.

## Verification behavior and limitations

Ask requests use automatic intent detection; factual questions are not routed through the summarizer. Retrieval ranks owner-scoped passages with BM25 and deterministic query-concept expansion, reranks by section heading, and returns only relevant evidence. If OpenAI embeddings are explicitly enabled, they rerank the top BM25 candidates and are cached with the passage records. If that provider call fails, the report identifies the BM25-only fallback. Dense semantic search is not used unless configured.

The synthesizer selects source sentences relevant to the exact question rather than returning the first retrieved passage. The Critic checks question relevance, verifies that each answer sentence occurs in its cited passage, and checks contradictions, expiry, and supersession before the Resolution Agent assigns a status. Reports retain page/passage citations, compared versions, resolution rationale, confidence notes, and uncertainty. The Critic does not treat old publication dates or newer uploads as authority.

Summaries are a separate action and an automatically detected Ask intent. They process all indexed passages in chunks (or the requested chapter scope), prioritize purpose and conclusion passages for the overview, exclude bibliography entries, and select source sentences for key points/requirements/warnings. Citations retain passage IDs and page numbers and summaries persist on the document and report. Extraction warnings prevent a summary from being labeled complete. This is a source-linked extractive summary, not an LLM-generated paraphrase.

Document details returns ten extracted passages per request, with backend search and pagination. It displays passage count separately from source page numbers and preserves expandable source text.

Limitations: without an explicitly configured OpenAI provider, retrieval uses BM25 and curated concept expansion rather than dense embeddings; factual answers and summaries are extractive. Contradiction and supersession checks are heuristic, not a substitute for human review. OpenAI embedding requests transmit only the question and candidate passage text when the provider is enabled; vectors are cached in MongoDB with those passage records.

## Tests

The backend tests use an isolated MongoDB database and clean up their generated records and files:

```powershell
$env:MONGO_DATABASE = "truthguard_test"
.\.venv\Scripts\python -m pytest -q backend/tests
```

Build the frontend:

```powershell
cd frontend
npm run build
npm run test:e2e
```

The Auto Call question evaluation expectations and source-page targets are recorded in [backend/tests/auto_call_qa_evaluation.json](./backend/tests/auto_call_qa_evaluation.json). The API suite also verifies question-specific answers, citation ownership/page metadata, critic execution, insufficient-evidence handling, and passage pagination/search.
