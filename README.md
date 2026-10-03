# ScrapeFlow

> A full-stack, visual web-scraping automation SaaS built with **Next.js 15 (App Router)**, **React Flow**, **Prisma**, **Clerk Auth**, and **Stripe**.

Users build scraping workflows by connecting task-nodes on a drag-and-drop canvas, schedule them with cron expressions, and pay for execution via a credit system.

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Architecture Overview](#architecture-overview)
- [Project Structure](#project-structure)
- [API Endpoints](#api-endpoints)
- [Server Actions](#server-actions)
- [Workflow Task Catalog](#workflow-task-catalog)
- [Database Schema](#database-schema)
- [Environment Variables](#environment-variables)
- [Getting Started](#getting-started)
- [Running Locally](#running-locally)
- [Deployment Notes](#deployment-notes)

---

## Features

| Feature | Details |
|---|---|
| 🔐 **Authentication** | Clerk — sign-in / sign-up with redirect to onboarding setup |
| 🎨 **Visual Workflow Editor** | Drag-and-drop canvas powered by `@xyflow/react` |
| ▶️ **Manual Execution** | Run any published workflow on-demand |
| ⏰ **Cron Scheduling** | Standard 5-field cron _and_ simple interval syntax (`2m`, `1h`, `3d`) |
| 💳 **Credit System** | Users start with 100 free credits; buy packs via Stripe Checkout |
| 🔒 **Secure Credentials** | AES-256-CBC encrypted storage for API keys, SMTP, and custom secrets |
| 🤖 **AI Extraction** | OpenAI-powered data extraction node |
| 📧 **Email Delivery** | Nodemailer `SEND_EMAIL` task using stored SMTP credentials |
| 📊 **Analytics Dashboard** | Period-based charts for execution stats and daily credit usage |
| 📥 **File Downloads** | Download scraped files directly from the execution log UI |
| 📸 **Screenshots** | Take and download full-page browser screenshots |
| 🌗 **Dark / Light Theme** | `next-themes` toggle in the sidebar header |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 15 (App Router, Server Components, Server Actions) |
| Language | TypeScript |
| Styling | Tailwind CSS v3 + shadcn/ui |
| Flow Canvas | `@xyflow/react` v12 |
| Database ORM | Prisma 5 → PostgreSQL |
| Auth | Clerk (`@clerk/nextjs` v6) |
| Payments | Stripe (`stripe` v17) |
| Browser Automation | Puppeteer Core + `@sparticuz/chromium-min` |
| AI | OpenAI SDK v4 |
| Email | Nodemailer |
| Scheduling | `cron-parser` + `cronstrue` |
| State / Data Fetching | TanStack React Query v5 |
| Charts | Recharts |

---

## Architecture Overview

```
Browser
  │
  ▼
Next.js App Router (Clerk Middleware → auth guard)
  │
  ├─ /app/(auth)                  # Public: Sign-in / Sign-up (Clerk-hosted)
  ├─ /app/(dashboard)             # Protected: all authenticated views
  │    ├─ /                       # Analytics home (stats + charts)
  │    ├─ /workflows              # Workflow list + CRUD
  │    ├─ /billing                # Credit balance + purchase packs
  │    └─ /credentials            # Encrypted credential store
  ├─ /app/workflow/editor/[id]    # React Flow visual editor
  ├─ /app/workflow/runs/[id]      # Execution history & phase viewer
  ├─ /app/setup                   # New-user onboarding (creates UserBalance)
  │
  ├─ GET  /api/workflows/cron     # Cron trigger (called by scheduler)
  ├─ GET  /api/workflows/execute  # Workflow executor (Bearer auth)
  └─ POST /api/webhooks/stripe    # Stripe payment webhook
```

### Execution Flow

```
Cron trigger (external scheduler or LocalCronRunner)
  │
  └─▶ GET /api/workflows/cron
        │  credit pre-check per workflow
        └─▶ GET /api/workflows/execute?workflowId=<id>  (Bearer API_SECRET)
              │  create WorkflowExecution + ExecutionPhase rows
              └─▶ ExecutionWorkflow()
                    │  reserve credits → iterate phases → run executors
                    │  decrement credits per task
                    └─▶ finalize + revalidate cache
```

---

## Project Structure

```
scraper-flow/
├── actions/
│   ├── analytics/
│   │   ├── getCreditUsageInPeriod.ts
│   │   ├── getPeriods.ts
│   │   ├── getStatsCardsValues.ts
│   │   └── getWorkflowExecutionsStats.ts
│   ├── billing/
│   │   ├── getAvailableCredits.ts
│   │   ├── getUserPurchaseHistory.ts
│   │   ├── purchaseCredits.ts        # Creates Stripe Checkout session
│   │   └── setupUser.ts              # Upserts UserBalance on first login
│   ├── credentials/
│   │   ├── createCredential.ts       # Encrypts & stores credential
│   │   ├── deleteCredential.ts
│   │   └── getCredentialsForUser.ts
│   └── workflows/
│       ├── createWorkflow.ts
│       ├── deleteWorkflow.ts
│       ├── duplicateWorkflow.ts
│       ├── getWorkflowExecutions.ts
│       ├── getWorkflowExecutionWithPhase.ts
│       ├── getWorkflowPhaseDetails.ts
│       ├── getWorkflowsForUser.ts
│       ├── PublishWorkflow.ts        # Validates plan, sets status=PUBLISHED
│       ├── UnpublishWorkflow.ts
│       ├── removeWorkflowSchedule.ts
│       ├── runWorkflow.ts            # Manual trigger → calls execute API
│       ├── updateWorkflow.ts         # Save definition (nodes + edges)
│       └── updateWorkflowCron.ts     # Set/update cron schedule
│
├── app/
│   ├── (auth)/
│   │   ├── sign-in/[[...sign-in]]/page.tsx
│   │   └── sign-up/[[...sign-up]]/page.tsx
│   ├── (dashboard)/
│   │   ├── layout.tsx                # Sidebar + header wrapper
│   │   ├── (home)/page.tsx           # Analytics dashboard
│   │   ├── billing/page.tsx          # Credit packs + purchase history
│   │   ├── credentials/page.tsx      # Credential manager
│   │   └── workflows/page.tsx        # Workflow list
│   ├── api/
│   │   ├── webhooks/stripe/route.ts  # Stripe webhook handler
│   │   └── workflows/
│   │       ├── cron/route.ts         # Scheduled execution trigger
│   │       └── execute/route.ts      # Workflow executor endpoint
│   ├── setup/page.tsx                # Onboarding: creates UserBalance
│   ├── workflow/
│   │   ├── editor/[workflowId]/page.tsx
│   │   └── runs/[workflowId]/[executionId]/page.tsx
│   ├── globals.css
│   ├── layout.tsx                    # Root layout (Clerk + providers)
│   └── not-found.tsx
│
├── components/
│   ├── Sidebar.tsx
│   ├── BreadcrumbHeader.tsx
│   ├── LocalCronRunner.tsx           # Client-side dev cron trigger
│   ├── Logo.tsx
│   ├── ThemeModeToggle.tsx
│   ├── UserAvailableCreditsBadge.tsx
│   └── ui/                          # shadcn/ui component library
│
├── lib/
│   ├── encryption.ts                 # AES-256-CBC encrypt/decrypt
│   ├── prisma.ts                     # Prisma client singleton
│   ├── log.ts                        # LogCollector factory
│   ├── credential/
│   │   ├── credentialHelper.ts
│   │   └── getCredentialValue.ts     # Decrypt credential by name
│   ├── cron/
│   │   └── scheduleParser.ts         # Parse cron + interval expressions
│   ├── helper/
│   │   ├── appUrl.ts
│   │   ├── dates.ts
│   │   ├── phases.ts
│   │   └── waitFor.ts
│   ├── stripe/
│   │   ├── stripe.ts                 # Stripe client instance
│   │   └── handleCheckoutSessionCompleted.ts
│   └── workflow/
│       ├── createFlowNode.ts
│       ├── creditCheck.ts            # Pre-check & reserve credits
│       ├── eligibility.ts
│       ├── executionPlan.ts          # Build phase plan from DAG
│       ├── executionWorkflow.ts      # Core execution engine
│       ├── executor/                 # 21 task executor implementations
│       └── task/                     # 21 task type definitions + registry
│
├── prisma/
│   ├── schema.prisma
│   └── migrations/
│
├── schema/
│   ├── credential.ts
│   └── workflow.ts
│
├── types/
│   ├── TaskType.ts
│   ├── analytics.ts
│   ├── appNode.ts
│   ├── billing.ts
│   ├── executor.ts
│   ├── log.ts
│   └── workflow.ts
│
├── .env.example
├── .puppeteerrc.cjs
├── middleware.ts
├── next.config.mjs
├── tailwind.config.ts
└── tsconfig.json
```

---

## API Endpoints

### `GET /api/workflows/cron`

**Purpose:** Scheduled execution trigger — called by an external cron job or the `LocalCronRunner` component during development.

**Logic:**
1. Queries all `PUBLISHED` workflows with `cron` expression where `nextRunAt <= now`.
2. For each workflow, calls `checkWorkflowCredits()` to verify sufficient credits.
3. If credits OK → calls `GET /api/workflows/execute?workflowId=<id>` with `Bearer API_SECRET`.
4. If insufficient → logs failure and skips.
5. Updates `nextRunAt` for all processed workflows.

**Response:**
```json
{
  "workflowsScheduled": 3,
  "workflowsRun": 2,
  "workflowsRunDetails": [{ "id": "...", "name": "...", "creditsCost": 5 }],
  "workflowsSkipped": 1,
  "skippedDetails": [{ "id": "...", "reason": "insufficient_credits" }],
  "timestamp": "2026-01-01T00:00:00.000Z"
}
```

---

### `GET /api/workflows/execute?workflowId=<id>`

**Purpose:** Execute a single published workflow. Called by `/api/workflows/cron` and the `runWorkflow` server action.

**Auth:** `Authorization: Bearer <API_SECRET>` (validated via `timingSafeEqual`).

**Logic:**
1. Validates Bearer token.
2. Fetches workflow + parses `executionPlan`.
3. Creates `WorkflowExecution` + `ExecutionPhase` rows.
4. Calls `ExecutionWorkflow()`:
   - Reserves credits upfront.
   - Runs each phase executor in order.
   - Decrements credits per completed task.
   - Finalizes status: `COMPLETED` or `FAILED`.

| Status | Reason |
|---|---|
| `400` | Missing `workflowId` |
| `401` | Missing/invalid Bearer token |
| `404` | Workflow or execution plan not found |
| `500` | Unexpected server error |

---

### `POST /api/webhooks/stripe`

**Purpose:** Receive Stripe events to fulfill credit purchases.

**Auth:** Stripe webhook signature verified via `STRIPE_WEBHOOK_SECRET`.

| Event | Handler |
|---|---|
| `checkout.session.completed` | Increments `UserBalance.credits` by pack amount; creates `UserPurchase` record |

---

## Server Actions

All server actions use `"use server"` and authenticate via Clerk's `auth()`.

### Analytics

| Action | Description |
|---|---|
| `GetPeriods()` | Year-month periods with execution data |
| `GetStatsCardsValues(period)` | Workflow executions, phase executions, credits consumed |
| `GetWorkflowExecutionsStats(period)` | Daily COMPLETED vs FAILED breakdown for charts |
| `GetCreditUsageInPeriod(period)` | Daily credit spend for billing chart |

### Billing

| Action | Description |
|---|---|
| `SetupUser()` | Upsert `UserBalance` (100 credits) → redirect to `/` |
| `GetAvailableCredits()` | Current credit balance |
| `GetUserPurchaseHistory()` | All `UserPurchase` records |
| `PurchaseCredits(packId)` | Create Stripe Checkout session → redirect |

### Credentials

| Action | Description |
|---|---|
| `CreateCredential(data)` | AES-256-CBC encrypt value → store in DB |
| `DeleteCredential(id)` | Delete by ID (ownership verified) |
| `GetCredentialsForUser()` | List credentials (values NOT returned) |

### Workflows

| Action | Description |
|---|---|
| `GetWorkflowsForUser()` | List all workflows with last run info |
| `CreateWorkflow(data)` | Create new DRAFT workflow |
| `DeleteWorkflow(id)` | Delete workflow + cascade executions |
| `DuplicateWorkflow(id)` | Clone as new DRAFT |
| `UpdateWorkflow(data)` | Save nodes/edges + recalculate `creditsCost` |
| `PublishWorkflow(id)` | Validate execution plan → set `PUBLISHED` |
| `UnpublishWorkflow(id)` | Set back to `DRAFT`, clear plan |
| `RunWorkflow(workflowId)` | Manual trigger via internal API call |
| `UpdateWorkflowCron(id, cron)` | Validate + set cron + `nextRunAt` |
| `RemoveWorkflowSchedule(id)` | Clear `cron` + `nextRunAt` |
| `GetWorkflowExecutions(id)` | Execution history list |
| `GetWorkflowExecutionWithPhase(id)` | Full execution + all phases |
| `GetWorkflowPhaseDetails(id)` | Phase detail with logs + I/O |
| `CheckWorkflowRunEligibility(id)` | Check PUBLISHED status + credit balance |

---

## Workflow Task Catalog

### 🌐 Browser Tasks

| Task | Credits | Description |
|---|---|---|
| `LAUNCH_BROWSER` | 5 | Launch Puppeteer browser instance with a start URL |
| `PAGE_TO_HTML` | 2 | Get full HTML content of the current page |
| `NAVIGATE_URL` | 2 | Navigate to a new URL |
| `SCROLL_TO_ELEMENT` | 1 | Scroll to a CSS selector |
| `TAKE_SCREENSHOT` | 3 | Capture full-page screenshot |

### 🖱️ Interaction Tasks

| Task | Credits | Description |
|---|---|---|
| `FILL_INPUT` | 1 | Fill a form input field |
| `CLICK_ELEMENT` | 1 | Click a CSS selector |
| `WAIT_FOR_ELEMENT` | 1 | Wait for an element to appear/disappear |
| `WAIT_DELAY` | 1 | Pause for N seconds |

### 📤 Extraction Tasks

| Task | Credits | Description |
|---|---|---|
| `EXTRACT_TEXT_FROM_ELEMENT` | 2 | Extract text from a CSS selector |
| `EXTRACT_TABLE_DATA` | 3 | Extract HTML table to JSON array |
| `EXTRACT_DATA_WITH_AI` | 4 | OpenAI-powered structured data extraction |

### 🔧 Data Processing Tasks

| Task | Credits | Description |
|---|---|---|
| `READ_PROPERTY_FROM_JSON` | 1 | Read a key from a JSON string |
| `ADD_PROPERTY_TO_JSON` | 1 | Add/update a key in a JSON string |
| `DATA_TRANSFORM` | 2 | Apply transformations to data |
| `FILTER_DATA` | 1 | Filter JSON array by field + condition |
| `CONDITIONAL_LOGIC` | 1 | Branch execution based on a condition |
| `LOOP` | 2 | Iterate over a JSON array |

### 📨 Delivery Tasks

| Task | Credits | Description |
|---|---|---|
| `DELIVER_VIA_WEBHOOK` | 2 | POST data to a webhook URL |
| `SEND_EMAIL` | 3 | Send email via Nodemailer SMTP credential |
| `DOWNLOAD_FILE` | 2 | Download a file from a URL |

---

## Database Schema

```prisma
model Workflow {
  id            String    @id @default(cuid())
  userId        String
  name          String
  description   String?
  definition    String    // JSON: { nodes[], edges[] }
  executionPlan String?   // JSON: WorkflowExecutionPlan
  creditsCost   Int       @default(0)
  status        String    // DRAFT | PUBLISHED
  cron          String?
  lastRunAt     DateTime?
  lastRunId     String?
  lastRunStatus String?
  nextRunAt     DateTime?
  @@unique([name, userId])
}

model WorkflowExecution {
  id              String    @id @default(cuid())
  workflowId      String
  userId          String
  trigger         String    // MANUAL | CRON | SCHEDULED
  status          String    // PENDING | RUNNING | COMPLETED | FAILED
  creditsConsumed Int       @default(0)
  definition      String    // snapshot at execution time
  phases          ExecutionPhase[]
}

model ExecutionPhase {
  id              String    @id @default(cuid())
  userId          String
  status          String
  number          Int
  node            String    // JSON: AppNode
  name            String
  inputs          String?   // JSON
  outputs         String?   // JSON
  creditsConsumed Int?
  logs            ExecutionLog[]
}

model ExecutionLog {
  id               String   @id @default(cuid())
  logLevel         String   // info | error
  message          String
  timestamp        DateTime @default(now())
  executionPhaseId String
}

model UserBalance {
  userId  String @id
  credits Int    @default(0)
}

model Credential {
  userId      String
  name        String
  value       String   // AES-256-CBC encrypted
  type        String   // smtp_email | api_key | custom
  description String?
  @@unique([userId, name])
}

model UserPurchase {
  userId      String
  stripeId    String
  description String
  amount      Int
  currency    String
}
```

---

## Environment Variables

Copy [`.env.example`](./.env.example) to `.env.local` and fill in your values.

| Variable | Required | Description |
|---|---|---|
| `NEXT_PUBLIC_APP_URL` | ✅ | Base URL — no trailing slash (e.g. `http://localhost:3000`) |
| `DATABASE_URL` | ✅ | PostgreSQL connection string |
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | ✅ | Clerk publishable key |
| `CLERK_SECRET_KEY` | ✅ | Clerk secret key |
| `NEXT_PUBLIC_CLERK_SIGN_IN_URL` | ✅ | `/sign-in` |
| `NEXT_PUBLIC_CLERK_SIGN_UP_URL` | ✅ | `/sign-up` |
| `NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL` | ✅ | `/` |
| `NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL` | ✅ | `/setup` |
| `STRIPE_SECRET_KEY` | ✅ | Stripe secret key |
| `STRIPE_WEBHOOK_SECRET` | ✅ | Stripe webhook signing secret |
| `STRIPE_SMALL_PACK_PRICE_ID` | ✅ | 1,000-credit pack price ID ($9.99) |
| `STRIPE_MEDIUM_PACK_PRICE_ID` | ✅ | 5,000-credit pack price ID ($39.99) |
| `STRIPE_LARGE_PACK_PRICE_ID` | ✅ | 10,000-credit pack price ID ($69.99) |
| `API_SECRET` | ✅ | Internal Bearer secret for cron→execute communication |
| `ENCRYPTION_SECRET` | ✅ | 64-char hex key for AES-256-CBC credential encryption |
| `OPENAI_API_KEY` | ✅ | OpenAI API key for the AI extraction task |

---

## Getting Started

### Prerequisites

- Node.js >= 18
- PostgreSQL database (Neon / Supabase / Railway / local)
- Clerk account
- Stripe account (test mode)
- OpenAI API key

### 1. Clone & Install

```bash
git clone https://github.com/kirtan597/ScrapeFlow.git
cd ScrapeFlow
npm install --legacy-peer-deps
```

### 2. Configure Environment

```bash
cp .env.example .env.local
# Fill in all values in .env.local
```

### 3. Set Up Database

```bash
npx prisma migrate dev
# Optional: npx prisma studio
```

### 4. Set Up Clerk

1. Create an app at [dashboard.clerk.com](https://dashboard.clerk.com).
2. Copy the publishable key and secret key into `.env.local`.
3. Set the **After sign-up URL** to `/setup` in the Clerk dashboard so new users get 100 free credits automatically.

### 5. Set Up Stripe

1. Create 3 one-time products in the Stripe dashboard:
   - **Small Pack** — $9.99
   - **Medium Pack** — $39.99
   - **Large Pack** — $69.99
2. Copy each Price ID into the corresponding env var.
3. Add a webhook for `checkout.session.completed` pointing to `https://<your-domain>/api/webhooks/stripe`.

For local webhook testing:
```bash
stripe listen --forward-to localhost:3000/api/webhooks/stripe
```

### 6. Generate Security Keys

```bash
# API_SECRET
openssl rand -hex 32

# ENCRYPTION_SECRET (must be exactly 64 hex chars for AES-256-CBC)
openssl rand -hex 32
```

---

## Running Locally

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

**First-time flow:**
1. Sign up → `/setup` → 100 credits provisioned → redirected to `/`.
2. Go to `/workflows` → create → open visual editor.
3. Connect tasks, save, publish, then run manually or set a cron schedule.

> **Dev Cron:** The `LocalCronRunner` component polls `/api/workflows/cron` every minute in development to simulate the production scheduler.

---

## Deployment Notes

### Vercel

1. Import the repo into Vercel and add all env vars.
2. Add `vercel.json` for cron scheduling:

```json
{
  "crons": [
    {
      "path": "/api/workflows/cron",
      "schedule": "* * * * *"
    }
  ]
}
```

3. The build script `prisma generate && next build` is pre-configured.

### Puppeteer on Serverless

`puppeteer-core` + `@sparticuz/chromium-min` handle headless Chrome on serverless platforms. Both packages are listed in `serverExternalPackages` in `next.config.mjs` to prevent bundling issues.

---

## Scripts

| Script | Description |
|---|---|
| `npm run dev` | Start development server |
| `npm run build` | `prisma generate` + `next build` |
| `npm run start` | Start production server |
| `npm run lint` | Run ESLint |
| `npm run prisma:migrate` | Run `prisma migrate dev` |
| `npm run prisma:generate` | Run `prisma generate` |
| `npm run prisma:studio` | Open Prisma Studio |

---

## License

MIT
