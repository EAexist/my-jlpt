# Tech Decisions

> Add an entry here whenever a significant architectural or tooling decision is made.
> Format: decision, options considered, chosen, reason.

---

## LLM API
- **Candidates**: OpenAI GPT-4o-mini, Gemini 1.5 Flash
- **Chosen**: TBD
- **Reason**: —
- **Key consideration**: Japanese NLP quality vs. cost per call. Evaluate with sample JLPT texts.

## JLPT Data Source
- **Candidates**: JMdict, jlpt-vocab-list (github.com/olafjanssen/jlpt-vocab)
- **Chosen**: TBD
- **License confirmed**: TBD
- **Injection strategy**: TBD — full DB lookup vs. truncated list injected into prompt context

## Database
- **Dev**: PostgreSQL
- **Prod candidates**: PostgreSQL on Railway, Supabase
- **Chosen**: PostgreSQL

## Hosting
- **Frontend**: Vercel (recommended for Next.js)
- **Backend candidates**: Railway, Render, Fly.io
- **Chosen**: TBD

## Auth Strategy
- **Chosen**: Google OAuth 2.0 only — no email/password
- **Backend**: `authlib` for OAuth client, `python-jose` for JWT issuance
- **Frontend**: `next-auth` with Google provider
- **Reason**: Simplest secure auth for a tool-style app; reduces surface area

## API Design
- **Chosen**: REST, FastAPI auto-docs at `/docs`
- **Contract source of truth**: `docs/openapi.yaml`
- **Reason**: API-first to support future React Native client
