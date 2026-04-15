# Fullstack AI Marketing Platform

This repository contains a two-part application for turning uploaded assets into reusable marketing content.

- `nextjs/`: Next.js 14 App Router application for the product UI, authenticated user flows, billing, uploads, and content generation.
- `asset-processing-service/`: Async Python worker that polls processing jobs, extracts or transcribes asset content, and pushes results back into the app.
- `resources/`: Prompt and content seed files used as reference material.

## Architecture

The system is split across a web app and a background worker:

1. A signed-in user creates a project or template in the Next.js app.
2. The user uploads assets through Vercel Blob using `/api/upload`.
3. The upload completion hook creates an `assets` record and an `asset_processing_jobs` record.
4. The Python worker polls `/api/asset-processing-job` using the shared `SERVER_API_KEY`.
5. The worker fetches the asset, transcribes or reads it, and updates `/api/asset` with extracted content and token counts.
6. The user configures prompts or imports a template into a project.
7. The Next.js app generates final marketing copy from prompts plus processed asset content using OpenAI.

## Repo Layout

```text
.
├── asset-processing-service/
├── nextjs/
└── resources/
```

## Prerequisites

- Node.js 18+
- npm 9+
- Python 3.9+
- Poetry
- FFmpeg available on the machine for audio and video processing
- PostgreSQL database reachable through `POSTGRES_URL`
- Clerk application keys
- Stripe test or live keys
- Vercel Blob token
- OpenAI API key

## Installation

### 1. Clone and enter the repository

```bash
git clone <your-repo-url>
cd fullstack-ai-marketing-platform
```

### 2. Install the Next.js app dependencies

```bash
cd nextjs
npm install
cd ..
```

### 3. Install the Python worker dependencies

```bash
cd asset-processing-service
poetry install
cd ..
```

## Environment Variables

Do not commit real secrets. Use local `.env` files, and keep `.env.example` files limited to placeholder values only.

### Next.js app

Create `nextjs/.env` with the following values:

| Variable | Required | Purpose |
| --- | --- | --- |
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Yes | Clerk client-side authentication key |
| `CLERK_SECRET_KEY` | Yes | Clerk server-side key |
| `POSTGRES_URL` | Yes | Database connection string used by Drizzle and runtime queries |
| `BLOB_READ_WRITE_TOKEN` | Yes | Vercel Blob uploads |
| `SERVER_API_KEY` | Yes | Shared bearer token for service-only API routes |
| `STRIPE_PUBLISHABLE_KEY` | Yes | Stripe client-side billing setup |
| `STRIPE_SECRET_KEY` | Yes | Stripe server SDK configuration |
| `STRIPE_PRICE_ID` | Yes | Subscription price used for checkout |
| `STRIPE_WEBHOOK_SECRET` | Yes | Stripe webhook signature verification |
| `APP_URL` | Yes | Absolute base URL for redirects and billing callbacks |
| `OPENAI_API_KEY` | Yes | Used by generated-content route |

Notes:

- `POSTGRES_URL_NON_POOLING`, `POSTGRES_URL_NO_SSL`, `POSTGRES_PRISMA_URL`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_HOST`, and `POSTGRES_DATABASE` may exist in hosted environments, but this code directly requires only `POSTGRES_URL`.
- Non-`NEXT_PUBLIC_` values remain server-only in Next.js. Only expose values with the `NEXT_PUBLIC_` prefix when they are intended for the browser.
- Generate a strong shared `SERVER_API_KEY` with:

```bash
openssl rand -hex 32
```

### Asset-processing service

Create `asset-processing-service/.env` with the following values:

| Variable | Required | Default | Purpose |
| --- | --- | --- | --- |
| `SERVER_API_KEY` | Yes | None | Authenticates requests to protected Next.js service endpoints |
| `OPENAI_API_KEY` | Yes | None | Used implicitly by the Python OpenAI client for Whisper transcription |
| `API_BASE_URL` | No | `http://localhost:3000/api` | Base URL for the Next.js API |
| `STUCK_JOB_THRESHOLD_SECONDS` | No | `30` | Marks jobs stale if heartbeats stop |
| `MAX_JOB_ATTEMPTS` | No | `3` | Maximum retry count before terminal failure |
| `MAX_NUM_WORKERS` | No | `2` | Number of concurrent worker tasks |
| `HEARTBEAT_INTERVAL_SECONDS` | No | `10` | Heartbeat update interval |
| `MAX_CHUNK_SIZE_BYTES` | No | `25165824` | Audio chunk size cap before transcription |
| `OPENAI_MODEL` | No | `whisper-1` | Model used for audio transcription |

## Running Locally

Run the web app and worker in separate terminals.

### Terminal 1: Next.js app

```bash
cd nextjs
npm run dev
```

The app will start on `http://localhost:3000`.

### Terminal 2: Python worker

```bash
cd asset-processing-service
poetry run asset-processing-service
```

## Next.js Pages

These are the user-facing routes currently defined in `nextjs/app`:

| Route | Access | Purpose |
| --- | --- | --- |
| `/` | Public | Landing page with hero, FAQ, CTA, and footer |
| `/pricing` | Public | Pricing and subscription pitch |
| `/projects` | Authenticated | Project list and create-project entry point |
| `/project/[projectId]` | Authenticated | Project workspace with upload, prompts, and generation steps |
| `/templates` | Authenticated | Template list and create-template entry point |
| `/template/[templateId]` | Authenticated | Template prompt management |
| `/settings` | Authenticated | Subscription management and Stripe billing entry points |

### Project workspace flow

The project detail UI uses three tabs:

- `upload`: upload and manage project assets
- `prompts`: create and edit project prompts
- `generate`: generate and edit final marketing copy

## Server Actions

The app currently defines two server actions in `nextjs/server/mutation.ts`:

| Action | Used From | Behavior |
| --- | --- | --- |
| `createProject()` | `/projects` | Creates a project for the current Clerk user and redirects to `/project/[id]` |
| `createTemplate()` | `/templates` | Creates a template for the current Clerk user and redirects to `/template/[id]` |

## Core Queries

The app currently exposes these server-side data helpers in `nextjs/server/queries.ts`:

- `getProjectsForUser()`
- `getProject(projectId)`
- `getTemplatesForUser()`
- `getTemplate(id)`
- `getUserSubscription()`

## API Routes

### Public or user-facing routes

| Method | Route | Auth | Purpose |
| --- | --- | --- | --- |
| `POST` | `/api/upload` | Clerk user enforced during upload token generation | Creates Vercel Blob upload tokens and persists uploaded asset + processing job |
| `POST` | `/api/webhooks/stripe` | Stripe signature | Handles Stripe webhook events for new subscriptions |
| `GET` | `/api/templates` | Clerk user | Lists templates for the signed-in user |
| `PATCH` | `/api/templates/[templateId]` | Clerk user | Renames a template |
| `DELETE` | `/api/templates/[templateId]` | Clerk user | Deletes a template |
| `GET` | `/api/templates/[templateId]/prompts` | Clerk user | Lists prompts for a template |
| `POST` | `/api/templates/[templateId]/prompts` | Clerk user | Creates a template prompt |
| `PATCH` | `/api/templates/[templateId]/prompts` | Clerk user | Updates a template prompt |
| `DELETE` | `/api/templates/[templateId]/prompts?id=...` | Clerk user | Deletes a template prompt |
| `PATCH` | `/api/projects/[projectId]` | Clerk user | Renames a project |
| `DELETE` | `/api/projects/[projectId]` | Clerk user | Deletes a project |
| `GET` | `/api/projects/[projectId]/assets` | Clerk user | Lists assets attached to a project |
| `DELETE` | `/api/projects/[projectId]/assets?assetId=...` | Clerk user | Deletes an asset and removes its blob file |
| `GET` | `/api/projects/[projectId]/asset-processing-jobs` | Clerk user | Lists asset-processing jobs for a project |
| `GET` | `/api/projects/[projectId]/prompts` | Clerk middleware | Lists project prompts |
| `POST` | `/api/projects/[projectId]/prompts` | Clerk user | Creates a project prompt |
| `PATCH` | `/api/projects/[projectId]/prompts` | Clerk user | Updates a project prompt |
| `DELETE` | `/api/projects/[projectId]/prompts?promptId=...` | Clerk user | Deletes a project prompt |
| `POST` | `/api/projects/[projectId]/import-template` | Clerk user | Copies template prompts into a project |
| `GET` | `/api/projects/[projectId]/generated-content` | Clerk middleware | Lists generated content records for a project |
| `POST` | `/api/projects/[projectId]/generated-content` | Clerk middleware | Generates marketing content from prompts and processed assets |
| `PATCH` | `/api/projects/[projectId]/generated-content` | Clerk middleware | Updates a generated content record by id |
| `DELETE` | `/api/projects/[projectId]/generated-content` | Clerk middleware | Deletes all generated content for a project |
| `POST` | `/api/stripe/create-checkout-session` | Clerk user | Starts Stripe subscription checkout |
| `POST` | `/api/stripe/create-portal-session` | Clerk user | Opens Stripe customer portal |

### Service-only routes protected by `SERVER_API_KEY`

These routes are enforced in `nextjs/middleware.ts` and are intended for the Python worker only.

| Method | Route | Purpose |
| --- | --- | --- |
| `GET` | `/api/asset-processing-job` | Returns jobs in non-terminal states |
| `PATCH` | `/api/asset-processing-job?jobId=...` | Updates job status, attempts, error state, and heartbeat |
| `GET` | `/api/asset?assetId=...` | Fetches a single asset record |
| `PATCH` | `/api/asset?assetId=...` | Stores processed asset content and token count |

## Worker Lifecycle

The Python service does the following:

1. Polls `/api/asset-processing-job` for `created`, `failed`, or `in_progress` jobs.
2. Marks stale `in_progress` jobs as failed when the heartbeat threshold is exceeded.
3. Fetches the target asset record and raw file.
4. Processes text directly, splits audio for transcription, or extracts audio from video before transcription.
5. Updates the asset content and token count through `/api/asset`.
6. Marks the job as `completed`, `failed`, or `max_attempts_exceeded`.

## Data Model Overview

Primary tables defined in `nextjs/server/db/schema.ts`:

- `projects`
- `assets`
- `asset_processing_jobs`
- `prompts`
- `templates`
- `template_prompts`
- `generated_content`
- `stripe_customers`
- `subscriptions`

## Helpful Commands

### Next.js app

```bash
cd nextjs
npm run dev
npm run build
npm run start
npm run lint
```

### Python worker

```bash
cd asset-processing-service
poetry install
poetry run asset-processing-service
poetry build
```

## Notes

- `SERVER_API_KEY` must be identical in both runtimes.
- `POSTGRES_URL` is required for both runtime database access and Drizzle config.
- FFmpeg must be installed locally for video and audio ingestion to work.
- If secrets were previously committed, rotate them and rewrite the affected Git history before pushing to a protected repository.
