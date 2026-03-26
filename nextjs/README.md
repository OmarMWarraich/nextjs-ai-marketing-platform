# Next.js App

This directory contains the main product application for the AI marketing platform. It handles authentication, project and template management, uploads, billing, asset orchestration, and final content generation.

## Stack

- Next.js 14 App Router
- React 18
- Clerk for authentication
- Drizzle ORM with PostgreSQL
- Stripe for subscriptions
- Vercel Blob for asset uploads
- OpenAI for generated marketing content

## Installation

```bash
cd nextjs
npm install
```

## Scripts

```bash
npm run dev
npm run build
npm run start
npm run lint
```

## Environment Variables

Create `nextjs/.env` and provide these values:

| Variable | Required | Purpose |
| --- | --- | --- |
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Yes | Clerk client-side auth key |
| `CLERK_SECRET_KEY` | Yes | Clerk server-side auth key |
| `POSTGRES_URL` | Yes | Primary database connection string |
| `BLOB_READ_WRITE_TOKEN` | Yes | Vercel Blob upload integration |
| `SERVER_API_KEY` | Yes | Shared token used by the Python worker on protected routes |
| `STRIPE_PUBLISHABLE_KEY` | Yes | Stripe client-side billing support |
| `STRIPE_SECRET_KEY` | Yes | Stripe server SDK configuration |
| `STRIPE_PRICE_ID` | Yes | Subscription price for checkout |
| `STRIPE_WEBHOOK_SECRET` | Yes | Stripe webhook signature verification |
| `APP_URL` | Yes | Absolute base URL for checkout and portal redirects |
| `OPENAI_API_KEY` | Yes | Used by the generated-content API route |

Notes:

- `POSTGRES_URL` is required by both runtime data access and `drizzle.config.ts`.
- `SERVER_API_KEY` must match the value used in `asset-processing-service/.env`.
- Only `NEXT_PUBLIC_` variables should be exposed to the browser.
- Keep `.env` files local and out of version control.

## Running Locally

```bash
cd nextjs
npm run dev
```

Open `http://localhost:3000`.

## Route Structure

### Pages

| Route | Access | Purpose |
| --- | --- | --- |
| `/` | Public | Landing page |
| `/pricing` | Public | Pricing page |
| `/projects` | Authenticated | Project list and create-project entry point |
| `/project/[projectId]` | Authenticated | Project workspace |
| `/templates` | Authenticated | Template list and create-template entry point |
| `/template/[templateId]` | Authenticated | Template workspace |
| `/settings` | Authenticated | Subscription management |

### Dashboard flow

The project detail page is organized into three steps:

1. Upload media
2. Configure prompts
3. Generate content

## Server Actions

Defined in `server/mutation.ts`:

| Action | Behavior |
| --- | --- |
| `createProject()` | Inserts a new project for the signed-in user and redirects to the project page |
| `createTemplate()` | Inserts a new template for the signed-in user and redirects to the template page |

## API Routes

### User-facing routes

| Method | Route | Purpose |
| --- | --- | --- |
| `POST` | `/api/upload` | Starts Vercel Blob uploads and persists uploaded assets/jobs |
| `POST` | `/api/webhooks/stripe` | Processes Stripe webhook events |
| `GET` | `/api/templates` | Lists templates |
| `PATCH` | `/api/templates/[templateId]` | Updates template title |
| `DELETE` | `/api/templates/[templateId]` | Deletes a template |
| `GET` | `/api/templates/[templateId]/prompts` | Lists template prompts |
| `POST` | `/api/templates/[templateId]/prompts` | Creates template prompts |
| `PATCH` | `/api/templates/[templateId]/prompts` | Updates template prompts |
| `DELETE` | `/api/templates/[templateId]/prompts?id=...` | Deletes template prompts |
| `PATCH` | `/api/projects/[projectId]` | Updates project title |
| `DELETE` | `/api/projects/[projectId]` | Deletes a project |
| `GET` | `/api/projects/[projectId]/assets` | Lists project assets |
| `DELETE` | `/api/projects/[projectId]/assets?assetId=...` | Deletes a project asset |
| `GET` | `/api/projects/[projectId]/asset-processing-jobs` | Lists processing jobs for a project |
| `GET` | `/api/projects/[projectId]/prompts` | Lists project prompts |
| `POST` | `/api/projects/[projectId]/prompts` | Creates project prompts |
| `PATCH` | `/api/projects/[projectId]/prompts` | Updates project prompts |
| `DELETE` | `/api/projects/[projectId]/prompts?promptId=...` | Deletes project prompts |
| `POST` | `/api/projects/[projectId]/import-template` | Copies template prompts into a project |
| `GET` | `/api/projects/[projectId]/generated-content` | Lists generated content |
| `POST` | `/api/projects/[projectId]/generated-content` | Generates content using prompts plus processed assets |
| `PATCH` | `/api/projects/[projectId]/generated-content` | Updates generated content |
| `DELETE` | `/api/projects/[projectId]/generated-content` | Deletes generated content |
| `POST` | `/api/stripe/create-checkout-session` | Starts Stripe checkout |
| `POST` | `/api/stripe/create-portal-session` | Starts Stripe billing portal session |

### Worker-only routes

Protected in `middleware.ts` with `Authorization: Bearer <SERVER_API_KEY>`:

| Method | Route | Purpose |
| --- | --- | --- |
| `GET` | `/api/asset-processing-job` | Lists non-terminal processing jobs |
| `PATCH` | `/api/asset-processing-job?jobId=...` | Updates a processing job |
| `GET` | `/api/asset?assetId=...` | Fetches one asset |
| `PATCH` | `/api/asset?assetId=...` | Saves extracted asset content |

## Core Data Model

Main tables defined in `server/db/schema.ts`:

- `projects`
- `assets`
- `asset_processing_jobs`
- `prompts`
- `templates`
- `template_prompts`
- `generated_content`
- `stripe_customers`
- `subscriptions`

## Authentication and Access Rules

- `/`, `/pricing`, `/api/upload`, and `/api/webhooks/stripe` are public in middleware.
- Most dashboard pages require a signed-in Clerk user.
- `/api/asset-processing-job` and `/api/asset` are reserved for the Python worker and require `SERVER_API_KEY`.

## Billing Flow

1. The user visits `/settings`.
2. `POST /api/stripe/create-checkout-session` creates a subscription checkout session.
3. Stripe redirects back to `/settings?success=true` or `/settings?canceled=true`.
4. Stripe webhooks persist the subscription in the database.
5. Existing subscribers can open the billing portal via `POST /api/stripe/create-portal-session`.

## Upload and Generation Flow

1. Upload files through `/api/upload`.
2. The upload callback inserts `assets` and `asset_processing_jobs` rows.
3. The Python worker processes those jobs and updates the asset content.
4. Project prompts are authored directly or imported from templates.
5. `/api/projects/[projectId]/generated-content` creates final marketing copy with OpenAI.

