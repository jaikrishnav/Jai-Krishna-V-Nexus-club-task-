# Nexus Knowledge

**Smart Document Knowledge Assistant** — upload your lecture PDFs and text files, ask questions in plain language, and get answers that are *grounded only in your documents*, each carrying a citation you can click through to the exact source passage.

No document → no answer. If nothing in your library is relevant, the app says so and shows the closest passages instead of inventing something.

---

## Table of contents

- [What it does](#what-it-does)
- [How it works](#how-it-works)
- [Tech stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Quick start](#quick-start)
- [Scripts](#scripts)
- [Environment variables](#environment-variables)
- [HTTP API](#http-api)
- [Project structure](#project-structure)
- [Testing](#testing)
- [Security & data](#security--data)
- [Troubleshooting](#troubleshooting)
- [Limitations](#limitations)

---

## What it does

| Area | Details |
| --- | --- |
| **Ingestion** | Upload up to 10 PDF/TXT files per request (10 MB each). PDFs are text-extracted with `pdfjs-dist` keeping page numbers intact; TXT files are cut into virtual "Section N" pages so citations always point at a stable location. |
| **Grounded chat** | Ask questions; answers come back as markdown with `[n]` citation chips wired to a sources panel. |
| **Citations** | Every claim maps to a stored chunk: document name, page/section, relevance score, and the verbatim snippet — shown side-by-side, never as a fabricated link. |
| **No-answer behavior** | A relevance gate (`MIN_SIMILARITY`) drops weak chunks *before* the LLM sees them. Below the gate you get a fixed "not found in your documents" response plus the closest passages, clearly labeled as non-answers. |
| **Answer controls** | Explain simpler (re-uses the *same* retrieved chunks, no new retrieval), Regenerate (full pipeline re-run), Copy, View sources. |
| **Conversation styles** | Selectable answer style per conversation (persisted). |
| **Study tools** | Summary, Flashcards, Quiz, Compare documents, Explain — all generated from the same retrieved chunks. |
| **Document library** | Live processing status with real progress (`chunksEmbedded/chunksTotal`), AI-generated summaries with topic tags, delete. |
| **Workspaces** | Every request carries a random `X-Workspace-Id`; all SQL is filtered by it, so one visitor can never see another visitor's documents. |

### Screens

- `/` — landing + upload dropzone
- `/app` and `/app/c/:id` — the three-panel workspace: document library · chat/study · sources
- `/how-it-works` — plain-language explainer of the pipeline

Responsive: three columns ≥1280 px, drawer + slide-over on tablet, single column with bottom nav on mobile.

---

## How it works

```
PDF / TXT
  └─ extract text (page-aware, header/footer stripping)
      └─ chunk (~900 chars, ~150 overlap, sentence-aware)
          └─ embed each chunk  ──────────────┐
                                             │
question ─► rewrite with history ─► embed ───┤
                                             ▼
                              cosine similarity over stored vectors
                                             │
                              gate: drop chunks < MIN_SIMILARITY
                                             │
                              keep top-k (3–5) → LLM prompt
                                             │
                              ┌──────────────┴──────────────┐
                              ▼                             ▼
                     grounded answer + citations    "not found" + closest passages
```

Key tuning constants live in [`server/src/config.ts`](server/src/config.ts) (chunk size, overlap, gate, caps) with the trade-offs explained inline.

---

## Tech stack

**Server** — Node 20+, Express 4, better-sqlite3 (WAL), Zod validation, multer uploads, Helmet + CORS + `express-rate-limit`, `pdfjs-dist` + `pdf-lib`, Gemini *or* OpenAI for chat + embeddings, TSX to run TypeScript directly.

**Client** — Vite 6, React 18, TypeScript (strict), Tailwind CSS 3, React Router 6, `react-markdown` + `remark-gfm`, lucide-react icons, self-hosted Inter / Source Serif 4 via Fontsource.

**Single process**: in production Express serves the built client from the same origin — no separate Node server, no microservices.

---

## Prerequisites

- **Node.js ≥ 20** (`engines` is enforced)
- An API key for **Gemini** (default) or **OpenAI**

---

## Quick start

```bash
# 1. install (npm workspaces: client + server)
npm install

# 2. configure the AI provider
cp server/.env.example server/.env
#    then edit server/.env and set GEMINI_API_KEY=...  (or OPENAI_API_KEY=...)

# 3. develop — runs API on :3001 and Vite on :5173 with /api proxied
npm run dev
```

Open **http://localhost:5173**, drop in a PDF, wait for the status bar to reach 100 %, then ask a question.

Optional helpers:

```bash
npm run seed        # build 3 sample lecture PDFs and ingest them through the real pipeline
npm run seed:pdf    # just generate the sample PDF files
npm run calibrate   # measure in/out-of-scope similarity scores -> recommended MIN_SIMILARITY
```

### Production build

```bash
npm run build       # typechecks + builds client, typechecks server
npm start           # one process on :3001 serving API + client/dist
```

If `client/dist` is missing, `/` returns a friendly hint instead of a blank page.

---

## Scripts

Run from the repository root:

| Script | What it does |
| --- | --- |
| `npm run dev` | Server (`tsx watch`) + client (`vite`) concurrently |
| `npm run build` | `client`: `tsc --noEmit && vite build`, then `server`: `tsc --noEmit` |
| `npm start` | Run the API (serves the built client if present) |
| `npm test` | Server test suite (Vitest) |
| `npm run seed` / `seed:pdf` / `calibrate` | Sample data and retrieval calibration |

---

## Environment variables

All read by [`server/src/config.ts`](server/src/config.ts) from **`server/.env`** (copy `server/.env.example`). Invalid values fail fast with a readable message instead of crashing later.

| Variable | Default | Notes |
| --- | --- | --- |
| `AI_PROVIDER` | `gemini` | `gemini` or `openai` |
| `GEMINI_API_KEY` | — | Free key: https://aistudio.google.com/apikey |
| `GEMINI_CHAT_MODEL` | `gemini-3.8-flash` | |
| `GEMINI_EMBED_MODEL` | `gemini-embedding-001` | One vector per input — required by chunk batching |
| `OPENAI_API_KEY` | — | Used when `AI_PROVIDER=openai` |
| `OPENAI_CHAT_MODEL` | `gpt-4o-mini` | |
| `OPENAI_EMBED_MODEL` | `text-embedding-3-small` | |
| `TOP_K` | `5` | Chunks sent to the model; clamped to **3–5** on purpose |
| `MIN_SIMILARITY` | `0.55` | Relevance gate; model-dependent → run `npm run calibrate` after changing the embedding model |
| `MAX_FILE_MB` | `10` | Per-file upload cap |
| `MAX_FILES_PER_UPLOAD` | `10` | |
| `MAX_DOCS_PER_WORKSPACE` | `20` | |
| `PORT` | `3001` | |
| `CLIENT_ORIGIN` | `http://localhost:5173` | CORS origin in development |
| `DATA_DIR` | `./data` | SQLite DB + uploads, relative to `server/` |

Without a key the server still starts: uploads fail at the embedding step with a clear 503, and the UI shows a configuration banner.

---

## HTTP API

Base URL `http://localhost:3001/api` (the Vite dev server proxies `/api` for you).

Every endpoint **except `GET /api/health`** requires the header `X-Workspace-Id: <uuid>` — the client generates one on first visit.

### Health

| Method | Path | Description |
| --- | --- | --- |
| GET | `/health` | Liveness + `aiConfigured`, provider and model names |

### Documents

| Method | Path | Description |
| --- | --- | --- |
| POST | `/documents` | Multipart upload (field `files`, up to 10); extraction/embedding runs asynchronously and the client polls `GET /documents` for progress |
| GET | `/documents` | List documents + status/progress/summary |
| POST | `/documents/seed` | Generate the sample lectures and ingest them (background, so the UI shows staged processing) |
| GET | `/documents/:id` | One document |
| DELETE | `/documents/:id` | Delete document + its chunks |
| GET | `/documents/:id/chunks/:chunkId` | Raw stored chunk (source viewer) |

### Chat

| Method | Path | Description |
| --- | --- | --- |
| GET | `/conversations` | List conversations |
| POST | `/conversations` | Create |
| GET | `/conversations/:id` | Conversation + messages |
| PATCH | `/conversations/:id` | Update title, selected doc ids, style |
| DELETE | `/conversations/:id` | Delete (messages cascade) |
| POST | `/conversations/:id/messages` | **Run the full RAG pipeline**; rolls back the optimistic user message if it fails |
| POST | `/messages/:id/explain-simpler` | Same stored chunks, Simple style, no new retrieval |
| POST | `/messages/:id/regenerate` | Re-run the whole pipeline for the same question |

### Study

| Method | Path | Description |
| --- | --- | --- |
| POST | `/study/summary` | Summary from sampled chunks |
| POST | `/study/flashcards` | Flashcard deck |
| POST | `/study/quiz` | Quiz questions |
| POST | `/study/compare` | Compare selected documents |
| POST | `/study/explain` | Targeted explanation |

### Response conventions

- Success: plain JSON payload (e.g. `{ conversation, messages }`).
- Errors: always `{ "error": { "code", "message" } }` — never HTML.
- Rate limits: **60 req/min** per IP overall (`GET /api/health` and `GET /api/documents` exempt so UI polling doesn't starve), **20 req/min** on chat + study because those spend tokens.

---

## Project structure

```
├── client/                     # Vite + React SPA
│   ├── index.html
│   ├── tailwind.config.js      # design tokens (paper/ink/primary/amber…)
│   └── src/
│       ├── main.tsx            # BrowserRouter mount
│       ├── index.css           # CSS variables + Tailwind layers
│       ├── app/                # App (routes), AppContext (store), useConversation
│       ├── components/ui/      # Button, Badge, Tooltip, Skeleton, …
│       ├── features/
│       │   ├── landing/        # first screen
│       │   ├── upload/         # dropzone + upload hook
│       │   ├── library/        # left panel: document cards, summaries
│       │   ├── chat/           # messages, markdown + citation chips, composer
│       │   ├── sources/        # right panel: citations, "how this was generated"
│       │   ├── study/          # flashcards, quiz runner
│       │   ├── howItWorks/     # /how-it-works explainer (HowItWorksPage.tsx)
│       │   └── Workspace.tsx   # three-panel layout + breakpoint switching
│       └── lib/                # api client, types, format helpers, constants
└── server/                     # Express API (also serves client/dist in prod)
    ├── .env.example
    └── src/
        ├── index.ts            # middleware, rate limits, routes, static client
        ├── config.ts           # every env var + tuning constant, validated
        ├── http.ts             # JSON errors, X-Workspace-Id guard, zod parsing
        ├── db/                 # better-sqlite3 schema + queries
        ├── routes/             # health, documents, chat, study
        ├── rag/                # extract → clean → chunk → embed → retrieve → generate
        ├── llm/                # provider clients (Gemini / OpenAI)
        ├── study/              # summary / flashcards / quiz / compare generators
        └── seed/               # sample documents, PDF generator, calibration
```

---

## Testing

```bash
npm test        # server suite via Vitest
```

Vitest is wired up for the server workspace; the suite targets the pure pipeline pieces (chunking, similarity search, citation validation, the relevance gate, upload validation). **Note: no test files have been written yet**, so `npm test` currently reports "no test files found" until they land.

Independently of tests, the builds double as typechecks: `npm run build` runs `tsc --noEmit` for the server and `tsc --noEmit && vite build` for the client.

---

## Security & data

- **Secrets** live only in `server/.env` (git-ignored). Keys are never sent to the browser; `GET /api/health` exposes only non-secret metadata (configured flag, provider, model names).
- **Workspace isolation**: a random UUID header scopes every query; there are no accounts and no cross-workspace reads.
- **Headers**: Helmet enabled; CORS restricted to `CLIENT_ORIGIN`.
- **Uploads**: extension + size + count validation, per-workspace document cap; original files are stored under `server/data/uploads/`.
- **Rate limiting** as described above, keyed per IP (`trust proxy` set for one reverse proxy).
- **Data**: everything lives in `server/data/` (SQLite + uploads) — delete that folder to reset.

---

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| Banner "AI provider is not configured" | Set `GEMINI_API_KEY` (or `OPENAI_API_KEY`) in `server/.env`, restart the server |
| Uploads stuck / fail at embedding | Same as above — extraction works, embedding needs the key |
| `Missing or invalid X-Workspace-Id header` | You're calling the API directly; send a UUID header, or go through the client |
| Every answer says "not found" | Documents may still be processing, or `MIN_SIMILARITY` is too high for your embedding model → run `npm run calibrate` |
| 429 `RATE_LIMITED` | 60/min general, 20/min chat+study — wait a minute |
| `NO_DOCUMENTS` on send | No document with status `ready`; check the library panel |
| Port already in use | Change `PORT` in `server/.env` (client proxy targets `3001` — keep them in sync) |
| Blank page in production | Run `npm run build` so `client/dist` exists |

---

## Limitations

- **Text-based PDFs only** — scanned/image-only PDFs are detected (fewer than ~100 extracted chars) and rejected with a clear message rather than silently producing nothing.
- **Brute-force cosine search** over stored vectors is fine to ~500 chunks/document (capped); this is not a vector database.
- **No auth** — workspaces are anonymous and scoped by a client-held UUID; anyone with the same UUID sees the same data.
- **Single-node** — SQLite and local file storage; horizontal scaling would need a real database and object storage.
- **Relevance gate is model-dependent** — recalibrate when you switch embedding models.
