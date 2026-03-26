# Workspace Instructions

Use this file as a fast bootstrap for this monorepo. Keep area-specific behavior rules in scoped instruction files and link to them instead of duplicating.

## Repo At A Glance
- Monorepo with two runtime components:
  - `nextjs/`: Next.js 14 App Router app (UI + API routes).
  - `asset-processing-service/`: Async Python worker for media processing jobs.
- Shared prompt/content assets live in `resources/`.

## Architecture Boundaries
- Keep UI/API concerns in `nextjs/`.
- Keep long-running processing and transcription in `asset-processing-service/`.
- Service-to-app auth for processing routes uses `Authorization: Bearer <SERVER_API_KEY>` and is separate from Clerk user auth.

## Setup And Run Commands
- Next.js (run inside `nextjs/`):
  - `npm install`
  - `npm run dev`
  - `npm run build`
  - `npm run start`
  - `npm run lint`
- Python service (run inside `asset-processing-service/`):
  - `poetry install`
  - `poetry run asset-processing-service`
  - `poetry build`

## Environment Requirements
- `SERVER_API_KEY` is required by both Next.js middleware/routes and the Python worker.
- `POSTGRES_URL` is required for Drizzle/DB tooling.
- Never expose server secrets in client-executed code.

## Scoped Rules (Read These First)
- Next.js app code (`nextjs/**/*.{ts,tsx}`):
  - `.github/instructions/nextjs-app-guidelines.instructions.md`
- Python service code (`asset-processing-service/**/*.py`):
  - `.github/instructions/python-service-guidelines.instructions.md`
- Behavior changes in either runtime:
  - `.github/instructions/testing-requirements.instructions.md`

## Testing Policy
- Follow `.github/instructions/testing-requirements.instructions.md` for all behavior changes.
- This repo does not currently expose a fully standardized test command suite; when changing behavior, add or update focused tests with the change when possible, then run the smallest relevant lint/test commands and report what ran.

## Common Pitfalls
- Missing `SERVER_API_KEY` causes startup/runtime failures across both runtimes.
- Missing `POSTGRES_URL` breaks Drizzle config usage.
- Local Python media processing requires FFmpeg available on the machine.

## Reference Docs
- `nextjs/README.md` (currently default Next.js scaffold)
- `asset-processing-service/README.md` (currently empty)
- `resources/*.md` (prompt/content reference assets)