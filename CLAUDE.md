# my-jlpt

Japanese text input → JLPT study material generator. Users paste text or upload a file and receive per-sentence study content: grammar analysis, vocabulary synonyms, example sentences, JLPT-frequent phrase highlights, overall JLPT level evaluation.
Processing is async: `POST /content` returns `202 + job_id` immediately. The pipeline runs as a background job. The client polls `GET /content/{id}` for the final result.

## Stack
- Frontend: Next.js (App Router) + TypeScript + shadcn/ui + Tailwind CSS
- Backend (orchestrator): NestJS — owns auth, job dispatch, content persistence, Gemini calls
- NLP service: FastAPI (Python) — owns tokenization, JLPT vocab lookup, grammar pattern matching
- LLM: Gemini API via `@google/genai` npm SDK — model `gemini-2.5-flash`
- Job queue: BullMQ + Upstash Redis (serverless managed Redis, free tier)
- Auth: Google OAuth 2.0 → JWT
- DB: PostgreSQL (Prisma ORM)
- SSE: postponed — future worktree. Jobs write progress to DB only.

## Processing Pipeline

```
POST /content → 202 { job_id }
  NestJS enqueues job to BullMQ
  BullMQ worker picks up job and runs:
    [1] HTTP POST nlp-service/segment   → sentence list
    [2] HTTP POST nlp-service/vocab     → JLPT vocab hits per sentence
    [3] HTTP POST nlp-service/grammar   → grammar pattern labels per sentence
    [4] Gemini batch call               → meaning, synonyms, examples, per-sentence level
    [5] Overall level evaluation        → computed from per-sentence levels
    [6] Prisma write                    → Content + Job records updated
  Job record: status → 'completed' | 'failed', progress 0–100

GET /content/{id}    → full result (only when job status = 'completed')
GET /content         → list (summary only, no sentences/vocab)
```

Each pipeline step updates `Job.progress` in the DB (0 → 16 → 33 → 50 → 66 → 83 → 100).
SSE streaming is NOT implemented yet. Frontend polls or checks on load.

## NLP Microservice (FastAPI / Python)

Internal service. NestJS calls it over HTTP.
- Dev: runs locally on port 8001
- Prod: deployed to Google Cloud Run (scale-to-zero). NestJS calls it via its Cloud Run URL.

### Endpoints
```
POST /segment   { "text": "string" }         → { "sentences": ["string"] }
POST /vocab     { "sentences": ["string"] }  → { "hits": VocabHit[][] }
POST /grammar   { "sentences": ["string"] }  → { "patterns": [["string"]] }
```

### Implementation
- Tokenizer: `fugashi` + `unidic-lite` (dev) / `unidic` (prod)
- JLPT vocab: loaded from JSON file bundled in Docker image into memory at startup
- Grammar patterns: regex rules over MeCab token POS stream. Returns matched pattern labels only (e.g. `"〜てしまう"`). Explanation delegated to LLM.
- Dataset: agents must create a small example dataset (50–100 vocab entries, all 5 JLPT levels, ≥5 per level) bundled at `data/vocab_example.json` inside the container. Do not reference external paths.

### Dev startup
```bash
cd nlp-service
pip install -r requirements.txt
uvicorn main:app --reload --port 8001
```

## Gemini Integration (NestJS `llm` module)

Package: `@google/genai` (NOT `@google/generative-ai` — that is the old SDK).

```typescript
import { GoogleGenAI } from '@google/genai';
const ai = new GoogleGenAI({ apiKey: configService.get('GEMINI_API_KEY') });
```

Model: `gemini-2.5-flash`

### Structured output
Use Gemini native structured output: set `responseMimeType: 'application/json'` and `responseSchema` in the config. Validate the parsed response with Zod as a safety net.

```typescript
const response = await ai.models.generateContent({
  model: 'gemini-2.5-flash',
  contents: prompt,
  config: {
    responseMimeType: 'application/json',
    responseSchema: { /* JSON Schema object matching Zod schema */ },
  },
});
const raw = JSON.parse(response.text);
const validated = MyZodSchema.parse(raw); // throws ZodError on schema violation
```

### Batch size
Process sentences in batches of `LLM_BATCH_SIZE` (env var, default 5). Never call Gemini once per sentence. Never send all sentences in one call (token limit risk).

## Job Queue (BullMQ + Upstash Redis)

NestJS uses `@nestjs/bullmq`. Redis provided by Upstash (serverless managed Redis — free tier covers dev and low-traffic prod).

```
Queue name: 'content-processing'
Job payload: { jobId: string, text: string }
Worker: processes the 6-step pipeline, updates Job.progress + Job.status at each step
Retry: 3 attempts, exponential backoff
On failure: set Job.status = 'failed', Job.error = error message
```

Upstash provides a `REDIS_URL` in `rediss://` format (TLS). Use this directly in BullMQ connection config.
Note: use Upstash Fixed plan (not pay-per-request) for BullMQ — BullMQ issues many Redis commands per job and pay-per-request becomes costly at volume.

## Prisma Schema (current)

```prisma
model Job {
  id        String   @id @default(uuid())
  status    String   // 'pending' | 'processing' | 'completed' | 'failed'
  progress  Int      @default(0)  // 0–100
  error     String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  content   Content?
}

model Content {
  id        String   @id @default(uuid())
  jobId     String   @unique
  job       Job      @relation(fields: [jobId], references: [id])
  data      Json     // ContentFull shape per openapi.yaml
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

## Backend Module Structure (NestJS)

```
src/modules/
  auth/       → Google OAuth, JWT issuance
  content/    → POST /content (enqueue), GET /content, GET /content/:id
  llm/        → GeminiService: structured batch calls
  nlp/        → NlpService: HTTP client to FastAPI nlp-service
  job/        → BullMQ worker, pipeline orchestration
```

## Environment Variables

```bash
# NestJS backend
FRONTEND_URL=http://localhost:3000
JWT_SECRET=changeme
JWT_EXPIRE_MINUTES=10080
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GEMINI_API_KEY=          # from Google AI Studio — aistudio.google.com/apikey
DATABASE_URL=postgresql://...
REDIS_URL=rediss://:password@...upstash.io:6380   # Upstash Redis URL (TLS)
NLP_SERVICE_URL=http://localhost:8001             # Cloud Run URL in prod
LLM_BATCH_SIZE=5

# FastAPI nlp-service (no external secrets needed — vocab bundled in image)
```

## Deployment (prod — cheapest viable stack)

| Service | Platform | Est. monthly cost |
|---|---|---|
| NestJS backend | Railway (Hobby plan, usage-based) | ~$5–8 |
| FastAPI NLP service | Google Cloud Run (scale-to-zero) | ~$0–2 |
| PostgreSQL | Railway or Neon (free tier) | ~$0–5 |
| Redis | Upstash (free tier / Fixed plan) | ~$0 |
| **Total** | | **~$5–15/mo** |

Cloud Run cold starts (~1–2s) are acceptable because the NLP service is called from a background worker, not from a user-facing request.

## SSE Event Schema (future — not implemented yet)

```json
{ "event": "progress", "data": { "step": 2, "total": 6, "label": "...", "pct": 33 } }
{ "event": "done",     "data": { "content_id": "<uuid>" } }
{ "event": "error",    "data": { "detail": "..." } }
```

Documented here for future reference only. `GET /jobs/{job_id}/stream` is not in scope.

## Boundary Rules
- Frontend never calls LLM, NLP service, or DB — backend API only.
- NLP service is internal — NestJS is the only caller.
- NestJS is the only caller of both the NLP service and Gemini.
- API contract source of truth: `docs/openapi.yaml`.
- Never hardcode secrets — `.env` / `.env.local`.
- Read existing files before editing; never assume structure.

## Contracts & Docs
- API shape: `docs/openapi.yaml`
- Error shape: `{ "detail": "string" }`
- Auth: Bearer JWT in `Authorization` header
- Decisions: `docs/decisions.md`
- UI wireframe: `frontend/docs/frontend-wireframe.md`

## Resolved Decisions
- LLM: Gemini API, model `gemini-2.5-flash`, SDK `@google/genai`
- JLPT dataset: JSON bundled in Docker image, loaded into memory at startup
- DB: PostgreSQL via Prisma (Railway or Neon)
- Job queue: BullMQ + Upstash Redis (Fixed plan)
- NLP: FastAPI microservice on Cloud Run, called over HTTP by NestJS
- Tokenizer: `fugashi` + UniDic
- Grammar: rule-based (regex over MeCab tokens), explanations via LLM
- Async flow: POST /content → 202, pipeline in BullMQ worker
- SSE: postponed to future worktree
- Deployment: Railway (NestJS) + Cloud Run (NLP) — no ECS, no ElastiCache

## Git
- Feature branches off main; never commit directly to main.
- Read `frontend/CLAUDE.md` before touching frontend code.
- Read `backend/CLAUDE.md` before touching backend code.
