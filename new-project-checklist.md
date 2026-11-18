# Project Development & Production Readiness Checklist

## Table of Contents

- [How to Use This Doc](#how-to-use-this-doc)
- [Part 1: The Stack](#part-1-the-stack)
  - [1.0 At a Glance](#10-at-a-glance)
  - [1.1 Tooling Foundation](#11-tooling-foundation)
  - [1.2 Framework: Next.js (App Router)](#12-framework-nextjs-app-router)
  - [1.3 UI Layer](#13-ui-layer)
    - UI Workflow
  - [1.4 API Layer](#14-api-layer)
    - API Packages
    - API Conventions
  - [1.5 Database and ORM](#15-database-and-orm)
    - ORM Options
    - Database Connections
    - Migrations
    - Database Layering
    - Database Environments
    - Database Schema Rules
  - [1.6 Authentication and Authorization](#16-authentication-and-authorization)
    - Authentication Options
    - Authentication and Authorization Rules
  - [1.7 Rate Limiting and Abuse Protection](#17-rate-limiting-and-abuse-protection)
    - Rate Limiting Tools
    - Starter Rate Limits
  - [1.8 Caching](#18-caching)
    - Caching Layers
    - Caching Rules
  - [1.9 Services You Will Eventually Need](#19-services-you-will-eventually-need)
    - [Email](Email)
    - File Uploads
    - [Payments](Payments)
    - Background Jobs
    - [Realtime](Realtime)
    - [Search](Search)
  - [1.10 Testing](#110-testing)
  - [1.11 Observability and Performance](#111-observability-and-performance)
  - [1.12 Security Baseline](#112-security-baseline)
  - [1.13 CI/CD and Environments](#113-cicd-and-environments)
- [Part 2: The Reusable Checklist](#part-2-the-reusable-checklist)
  - [Phase 0: Define (Day 0)](#phase-0-define-day-0)
  - [Phase 1: Blueprint](#phase-1-blueprint-plan-to-about-70-then-stop)
  - [Phase 2: Setup (Day 1)](#phase-2-setup-day-1)
  - [Phase 3: Walking Skeleton (Day 1 to 2)](#phase-3-walking-skeleton-day-1-to-2)
  - [Phase 4: Vertical Slices](#phase-4-vertical-slices-repeat-for-each-feature)
    - Suggested Slice Order
    - Per-Slice Checklist
  - [Phase 5: Harden](#phase-5-harden)
    - [Security](Security)
    - [Reliability](Reliability)
    - [Performance](Performance)
    - UX and Accessibility
    - [Testing](Testing)
  - [Phase 6: Pre-Launch](#phase-6-pre-launch)
  - [Phase 7: Ship and Learn](#phase-7-ship-and-learn)
- [Part 3: Templates](#part-3-templates)
  - [3.1 Folder Structure](#31-folder-structure)
    - Dependency Direction
  - [3.2 Starter Commands](#32-starter-commands-a-menu-not-a-to-do)
    - Project Initialization
    - Forms, Validation and UI Extras
    - Database Setup
    - Authentication Setup
    - Security and Plumbing
    - Tooling and Tests
  - [3.3 Decision Log](#33-decision-log)
  - [3.4 Per-Endpoint Checklist](#34-per-endpoint-checklist-every-server-action-and-route-handler)
  - [3.5 Per-Table Checklist](#35-per-table-checklist)
  - [3.6 Debugging Routine](#36-debugging-routine)
  - [3.7 Beginner Traps](#37-beginner-traps)
- [The 6 Highest-Leverage Moves](#the-6-highest-leverage-moves)

---
## How to use this doc

- **Part 1** says what to use for each concern and which package does which job.
- **Part 2** is the checklist. Copy it into a fresh doc or repo issue for every new project.
- **Part 3** holds templates: folder structure, starter commands, decision log, per-endpoint and per-table checks.
- **Rule:** use the *Default* unless you can write down why not. The *Add when* column stops you from installing things too early. Every new tool must solve a problem you have today.

**The sequence that prevents most beginner failures:** Define → Blueprint → Walking skeleton → Vertical slices → Harden → Ship.

---

## Part 1: The stack

### 1.0 At a glance

| Concern | Default | Add when |
| --- | --- | --- |
| Package manager | pnpm | Day 0 |
| Language | TypeScript (strict) | Day 0 |
| Framework | Next.js (App Router) | Day 0 |
| Styling | Tailwind CSS | Day 0 |
| Components | shadcn/ui | Day 0 |
| Validation | Zod | Day 0 |
| Forms | React Hook Form | First form |
| Mutations from your UI | Server Actions | First write |
| Public API / webhooks | Route Handlers | External callers exist |
| Client-side server state | TanStack Query | Polling, infinite scroll, optimistic UI |
| Database | Postgres on Neon | Day 0 |
| ORM | Prisma 7 | Day 0 |
| Auth | Better Auth | First user-owned data |
| Rate limiting | Upstash Ratelimit | Before public launch |
| Caching | Next.js caching, then Upstash Redis | After you measure slowness |
| Email | Resend + React Email | Verification and reset emails |
| File uploads | UploadThing | First upload feature |
| Payments | Stripe | First revenue |
| Background jobs | Inngest | Task over \~10s, retries, schedules |
| Unit tests | Vitest | First pure logic |
| E2E tests | Playwright | Critical path works |
| Error tracking | Sentry | Walking skeleton |
| Analytics | PostHog | Launch |
| CI | GitHub Actions | Day 1 |
| Hosting | Vercel | Day 1 |

### 1.1 Tooling foundation

- **Node.js (LTS) + pnpm:** fast, disk-efficient installs. Commit `pnpm-lock.yaml`.
- **TypeScript with `strict: true`:** catches whole classes of bugs before runtime. Never use `any` to silence an error; use `unknown` and narrow it.
- **ESLint + Prettier** (or **Biome** as a single tool): pick one setup and do not mix them.
- **Husky + lint-staged:** lint and format on commit so broken code never reaches Git.
- **@t3-oss/env-nextjs (with Zod):** validates environment variables at build time, so a missing key fails loudly on your machine instead of silently in production.

### 1.2 Framework: Next.js (App Router)

- Components are **Server Components by default**. Add `use client` only for state, effects, event handlers, or browser APIs, and push it as far down the tree as you can.
- Fetch data in Server Components by calling your service functions directly. No API hop needed.
- **Next.js 16 renamed `middleware.ts` to `proxy.ts`** (export a function named `proxy`). Many older tutorials and library docs still show the old name.
- **Never make `proxy.ts` your only auth check.** A 2025 vulnerability (CVE-2025-29927) showed that middleware-only protection can be bypassed. Use it for cheap redirects, and verify the session in the server code that actually touches data.
- **Caching is opt-in** in current versions. Do not assume pages are cached; read the caching docs for your exact version, because this area changes fastest.
- Next.js ships regular security releases. Keep `next` updated with Dependabot or Renovate.
- `NEXT_PUBLIC_` env vars are **inlined into the browser bundle**. Never put a secret behind that prefix.

### 1.3 UI layer

| Package | Job |
| --- | --- |
| **Tailwind CSS** | Utility-first styling. Theme tokens (colors, spacing, fonts) live in your CSS config |
| **shadcn/ui** | Copy-in, accessible components (Button, Dialog, Dropdown, Table, Form) built on Radix UI primitives. You own the code |
| **lucide-react** | Icons |
| **clsx + tailwind-merge** | The `cn()` helper for conditional, conflict-free class names (shadcn installs it) |
| **class-variance-authority** | Component variants such as size and intent |
| **React Hook Form + @hookform/resolvers** | Forms with Zod validation. Reuse the same schema on the server |
| **TanStack Table** | Sorting, filtering, pagination for data tables (shadcn’s Data Table wraps it) |
| **Recharts** (via shadcn Charts) | Charts |
| **sonner** | Toast notifications |
| **next-themes** | Dark mode without flicker |
| **motion** (Framer Motion) | Animations. Use sparingly |
| **date-fns** | Date formatting and math |
| **nuqs** | Type-safe URL query state (filters, tabs, pagination) so views are shareable |
| **TanStack Query** | Client cache for server data: polling, infinite scroll, optimistic updates. Skip it if Server Components cover the need |
| **Zustand** | Small global client state, rarely needed. Do not reach for Redux |
| **next/font, next/image** (built in) | Font loading and image optimization with no extra packages |

**UI workflow:**

- Design flows in Figma first, then components. Define a tiny design system: colors, two fonts, spacing scale, border radius.
- Use shadcn components before building your own.
- Build all four states for every data view: loading, empty, error, success.
- Mobile first. Use semantic HTML (`button`, `nav`, `main`, `label`) and keep keyboard navigation working.

### 1.4 API layer

Choose by **who is calling**:

| Situation | Use |
| --- | --- |
| Your own UI submits a form or mutates data | **Server Action** (wrap with `next-safe-action` for validation and auth middleware) |
| A page needs data | Call a service function from the Server Component |
| External callers: webhooks, mobile app, public API | **Route Handler** (`app/api/.../route.ts`) |
| Many interactive client queries and you want end-to-end types without codegen | **tRPC** (v11) |
| Separate backend, edge-first API, or non-Next clients | **Hono** |

**Packages:**

- **Zod:** validates every input (body, params, query, form data) with `safeParse`. One schema, shared by client and server.
- **next-safe-action:** typed Server Actions with built-in input validation and auth middleware.
- **nanoid:** short, unguessable IDs for share links and slugs.
- **ky or ofetch:** calling third-party APIs with timeouts and retries.

**Conventions:**

- **Server Actions are public HTTP endpoints.** Treat them exactly like API routes: authenticate, authorize, validate.
- One response shape everywhere: success returns `{ data }`, failure returns `{ error: { code, message } }`.
- Use correct status codes (400 bad input, 401 not signed in, 403 not allowed, 404 missing, 409 conflict, 429 too many requests).
- Never leak stack traces or internal messages to the client. Log them server-side instead.
- Paginate every list. Prefer cursor pagination for large or fast-changing data.
- Make webhooks and payment calls **idempotent** (safe to run twice).
- Version the API (`/api/v1`) only once outsiders depend on it.

### 1.5 Database and ORM

- **Postgres** is the default for almost every app. Hosts: **Neon** (serverless Postgres with instant branching per PR) or **Supabase** (Postgres plus bundled auth, storage, realtime).
- **ORM (pick one):**
  - **Prisma 7:** readable schema language, built-in migrations, Prisma Studio GUI. Best beginner experience. Version 7 dropped the old Rust engine, so it is lighter, but setup changed (driver adapters, `prisma.config.ts`). Follow the current official guide, not 2024 tutorials.
  - **Drizzle ORM:** schema and queries in TypeScript that feel like SQL. Smallest footprint. Best if you want to learn SQL properly or deploy to edge runtimes.
- **Connections:** in serverless, use your host’s **pooled** connection string for the app (Neon’s has `-pooler` in the hostname) and the direct string for migrations.
- **Migrations only:** `prisma migrate dev` locally and `prisma migrate deploy` in CI or prod (Drizzle: `drizzle-kit generate` then `migrate`). Never edit prod by hand. Use `push` only for throwaway prototypes.
- **Layering:** only the data layer imports the ORM. UI components never touch the database.
- **Environments:** separate dev and prod databases (or Neon branches), a seed script for dev data, automated backups on prod.
- **Schema rules:**
  - Every table has `id`, `createdAt`, `updatedAt`.
  - Enforce uniqueness (email, slug) in the database, not just in code.
  - Add foreign keys and decide `onDelete` behavior deliberately.
  - Index every foreign key and every column you filter or sort by.
  - Store money as integer minor units (paise, cents), never floats. Store timestamps in UTC.
  - Store files in object storage and keep only the URL in the database.

### 1.6 Authentication and authorization

| Option | Pick when | Trade-off |
| --- | --- | --- |
| **Better Auth** (default) | You want a TypeScript library inside your app, users in your own Postgres, plugins for 2FA, passkeys, organizations, magic links | You wire up email sending and own the setup |
| **Clerk** | You want it working in an hour with prebuilt UI | Hosted, vendor lock-in, cost grows with users |
| **Supabase Auth** | You are already on Supabase and want row-level security tied to users | Couples auth to Supabase |
| **Neon Auth** | You are on Neon and want the shortest path (used in the source video) | Tied to Neon; check current docs |
| **Auth.js (NextAuth v5)** | OAuth-only apps and existing codebases | Docs can lag Next.js 16 changes; most 2026 comparisons recommend Better Auth for new projects |

**Rules that matter more than the library:**

- **Never hand-roll auth or crypto.** Use a library or service.
- **Authentication is not authorization.** Knowing who someone is does not mean they may touch this row. Put the owner check **inside the query**: `where: { id, userId: session.user.id }`.
- Create two helpers early: `getSession()` (returns user or null) and `requireUser()` (throws or redirects). Call `requireUser()` at the top of every protected action, handler, and page.
- Do role checks (admin, member) on the server, never by hiding buttons.
- Start with Google OAuth plus email/password. Add email verification and password reset before launch (needs an email provider).
- Use generic error messages on login (“Invalid email or password”) to avoid revealing which accounts exist.
- Sessions live in httpOnly, secure cookies (the library handles this). Never store tokens in `localStorage`.
- Rate-limit login, signup, and reset endpoints (see 1.7).

### 1.7 Rate limiting and abuse protection

**Why:** stops brute-force logins, scraping, and runaway costs on anything that sends email, SMS, or calls an AI API. Also protects you from your own buggy retry loops.

- **Default: `@upstash/ratelimit` + `@upstash/redis`.** HTTP-based Redis that works in serverless and edge, with a generous free tier. Algorithms: *sliding window* for APIs, *token bucket* for bursts, *fixed window* for simplicity.
- **Arcjet:** SDK that bundles rate limiting, bot detection, and email validation.
- **Vercel WAF or Cloudflare rules:** infrastructure-level limits with no code. Good as a first shield.
- **Do not use an in-memory counter** in serverless. Each instance has its own memory, so the limit is meaningless.
- **Cloudflare Turnstile** (CAPTCHA alternative) on signup and public forms.

**Starter limits (tune later):**

| Endpoint | Limit | Key by |
| --- | --- | --- |
| Login, signup, password reset | 5 to 10 per 15 min | IP + email |
| Public forms (contact, waitlist) | 5 per minute | IP |
| Authenticated API | 60 per minute | User ID |
| Costly calls (AI, email, SMS) | Much lower, plus a daily cap | User ID |

Return `429` with a `Retry-After` header and show a friendly message in the UI.

### 1.8 Caching

**Rule:** do not cache until you have measured. Fix queries and indexes first. A stale-cache bug is worse than a slow page.

| Layer | Tool | Use for |
| --- | --- | --- |
| Browser and CDN | `Cache-Control` headers, Next.js static assets, `next/image` | Static files, images, fully static pages |
| Server (framework) | Next.js opt-in caching (`use cache` and tag-based revalidation; confirm syntax in your version’s docs) | Shared, read-heavy data such as a public product list |
| Client | TanStack Query (`staleTime`) | Avoiding refetches while the user navigates |
| Shared store | **Upstash Redis** with a TTL | Expensive aggregates, third-party API responses, rate-limit counters |
| Database | Indexes, then materialized views | Slow queries |

**Rules:** always set a TTL. Invalidate on write. Cache read-heavy, rarely changing, non-user-specific data first. Include the user ID in the cache key for any private data so users never see each other’s results.

### 1.9 Services you will eventually need

- **Email:** **Resend** + **React Email** (templates as components). Verify your sending domain (SPF, DKIM) before launch or emails land in spam.
- **File uploads:** **UploadThing** (easiest), **Vercel Blob**, or **S3 / Cloudflare R2** with presigned URLs so files never pass through your server. Validate type and size server-side; never trust the filename.
- **Payments:** **Stripe** (global) or **Razorpay** (India). Create the order on the server, confirm via **webhooks** (verify the signature, handle duplicates), never trust the client redirect. Use Lemon Squeezy or Paddle if you want a merchant-of-record to handle tax.
- **Background jobs:** **Inngest** or **Trigger.dev** for retries, delays, and workflows; **Vercel Cron** for simple schedules. Never run long work inside a request handler.
- **Realtime:** start with polling or Server-Sent Events. Move to Pusher, Ably, or Supabase Realtime only if needed.
- **Search:** Postgres full-text search first. Add Meilisearch or Typesense only when it falls short.

### 1.10 Testing

- **Vitest:** unit tests for pure logic and services. This is where most of your tests belong.
- **Playwright:** 1 to 3 end-to-end tests for the critical path (sign up, do the core action, see the result).
- **React Testing Library:** only for tricky interactive components.
- **MSW:** mock third-party APIs in tests.
- Keep the core algorithm a pure function (data in, data out) so it is trivial to test. Skip chasing coverage percentages.

### 1.11 Observability and performance

- **Sentry:** error tracking with stack traces and release tags. Install it in the walking skeleton.
- **Structured logs:** JSON logs with a request ID (Pino). Never log passwords, tokens, or personal data.
- **Uptime monitor:** Better Stack or UptimeRobot hitting `/api/health`.
- **PostHog:** product analytics, session replay, feature flags. Vercel Analytics or Plausible are lighter options.
- **Performance:** Lighthouse and Web Vitals, `@next/bundle-analyzer`, `next/image`, lazy-load heavy components.

### 1.12 Security baseline

- Secrets only in env vars, never in Git. Rotate any key that ever leaked.
- Security headers and a Content Security Policy (configure in `next.config`). HTTPS everywhere.
- ORM parameterization prevents SQL injection; never concatenate user input into raw SQL.
- React escapes output by default. Avoid `dangerouslySetInnerHTML`; if you must, sanitize with DOMPurify.
- Configure CORS to a specific allow-list, never `*` on authenticated endpoints.
- Verify webhook signatures. Validate uploads. Use a least-privilege database user.
- Never expose service-role or admin keys to the browser.
- Turn on Dependabot or Renovate, and run `pnpm audit` in CI.

### 1.13 CI/CD and environments

- **GitHub Actions on every PR:** install → lint → typecheck → test → build. Protect `main` so it only accepts green PRs.
- **Vercel preview deployments** per PR; a Neon branch per preview if you want isolated data.
- **Migrations run in the deploy pipeline**, never by hand on prod.
- Three environments: local, preview, production, each with its own env vars and database.
- Keep a rollback plan: know how to redeploy the previous version in one click.

---

## Part 2: The reusable checklist

*Copy this section for each new project. Do the phases in order. Do not start a phase until the previous one is done.*

### Phase 0: Define (Day 0)

- [ ] One-sentence spec written: “\[User\] can \[core action\] so that \[outcome\].”
- [ ] Target user and the one problem named
- [ ] Must-have / Later / Never lists written
- [ ] v1 scope = one core feature + at most two supporting features
- [ ] Deadline set (2 to 4 weeks)

### Phase 1: Blueprint (plan to about 70%, then stop)

- [ ] User flows sketched (paper, then low-fidelity Figma)
- [ ] Screen list written, including empty, loading, and error states
- [ ] **ERD drawn:** entities, fields, relationships, unique constraints
- [ ] Core algorithm pseudocoded with 3 to 5 hand-computed test cases
- [ ] API contract listed: endpoint or action, input shape, output shape
- [ ] Permission matrix: who can see, create, edit, delete what
- [ ] Third-party services listed and free-tier limits checked
- [ ] Decision log started (see template in Part 3)
- [ ] Top 3 unknowns identified, each tested with a half-day spike

### Phase 2: Setup (Day 1)

- [ ] Git repo created, `.gitignore`, README stub, `main` protected
- [ ] `create-next-app` with TypeScript, Tailwind, ESLint, App Router, `src/`
- [ ] `strict` TypeScript on; Prettier, Husky, lint-staged configured
- [ ] shadcn/ui initialized; base components added (Button, Input, Form, Dialog)
- [ ] Folder structure created (see Part 3)
- [ ] `env.ts` validation in place; `.env.example` committed; `.env` ignored
- [ ] Database created (dev and prod, or Neon branches); ORM initialized; first migration applied
- [ ] CI pipeline green (lint, typecheck, test, build)
- [ ] Vercel connected with preview deployments

### Phase 3: Walking skeleton (Day 1 to 2)

- [ ] One page reads one row from the DB through the service layer
- [ ] Deployed to a production URL
- [ ] `/api/health` endpoint responds
- [ ] Sentry installed and a test error received
- [ ] Everything above reachable from a fresh clone using only the README

### Phase 4: Vertical slices (repeat for each feature)

**Suggested slice order:** (1) core feature, ugly but working, no auth → (2) persistence and share links → (3) auth and ownership → (4) supporting features → (5) polish.

**Per-slice checklist (tasks of 2 to 4 hours each):**

- [ ] Pure logic written with unit tests first
- [ ] Schema change and migration reviewed
- [ ] Zod validator written (shared client and server)
- [ ] Service function written (business logic, no framework imports)
- [ ] Server Action or Route Handler: authenticated, authorized, validated
- [ ] UI built with loading, empty, error, success states
- [ ] Manually tested including a bad-input case
- [ ] Small commits, PR opened, CI green
- [ ] Preview deployment verified
- [ ] Anything new that came to mind added to “Later”, not built now

### Phase 5: Harden

**Security**

- [ ] Every query scoped to the owner (test with a second account)
- [ ] All inputs validated; all Server Actions treated as public endpoints
- [ ] Rate limits on login, signup, reset, public forms, costly endpoints
- [ ] Security headers and CSP set; CORS allow-list only
- [ ] No secrets in Git history or `NEXT_PUBLIC_` vars
- [ ] Webhook signatures verified; uploads validated
- [ ] `pnpm audit` clean; Dependabot on

**Reliability**

- [ ] Error boundaries and a friendly 404 and 500 page
- [ ] Failed external calls handled with timeouts and retries
- [ ] Structured logging in place; Sentry alerts routed to you
- [ ] DB backups confirmed and a restore tested once

**Performance**

- [ ] Indexes on filtered, sorted, and foreign-key columns
- [ ] Lists paginated; no N+1 queries
- [ ] Images optimized; bundle size checked
- [ ] Caching added only where measurement justified it

**UX and accessibility**

- [ ] Works on a real phone and on slow network
- [ ] Forms show inline errors and disable the button while submitting
- [ ] Keyboard navigation and focus states work; contrast checked
- [ ] Page titles, meta descriptions, and Open Graph image set

**Testing**

- [ ] Unit tests cover the core logic
- [ ] 1 to 3 Playwright tests cover the critical path

### Phase 6: Pre-launch

- [ ] Custom domain with HTTPS
- [ ] Production env vars set; secrets rotated if they were ever shared
- [ ] Email domain verified (SPF, DKIM); verification and reset emails tested
- [ ] Privacy policy and terms if you collect personal data
- [ ] Analytics installed; uptime monitor watching `/api/health`
- [ ] Production smoke test: sign up, core action, sign out, sign in
- [ ] Rollback plan written (how to revert a deploy and a migration)
- [ ] README complete: what it is, how to run it, architecture in 5 lines

### Phase 7: Ship and learn

- [ ] Put it in front of **5 real people**; watch silently and take notes
- [ ] Turn every confusion into a backlog item; fix the top 3
- [ ] Weekly: merge dependency updates, check Sentry, review analytics
- [ ] Retrospective: what slowed me down? Update this checklist

---

## Part 3: Templates

### 3.1 Folder structure

```
src/
  app/
    (marketing)/        public pages
    (app)/              logged-in pages
    api/                route handlers (webhooks, public API)
  components/
    ui/                 shadcn components
    features/           feature-specific components
  lib/
    auth.ts             auth config, getSession, requireUser
    db.ts               ORM client (singleton)
    env.ts              validated environment variables
    ratelimit.ts        limiter instances
    actions/            server actions
    services/           business logic (pure where possible)
    validators/         Zod schemas shared by client and server
  types/
tests/
  unit/
  e2e/
prisma/                 schema and migrations (or drizzle/)
```

**Dependency direction (one way only):** UI → actions / route handlers → services → data layer. Nothing imports upward.

### 3.2 Starter commands (a menu, not a to-do)

Install each package when you reach the slice that needs it.

```bash
pnpm create next-app@latest my-app --ts --tailwind --eslint --app --src-dir --import-alias '@/*'
cd my-app
pnpm dlx shadcn@latest init

# forms, validation, UI extras
pnpm add zod react-hook-form @hookform/resolvers sonner next-themes

# database (follow the current Prisma 7 setup guide after installing)
pnpm add -D prisma
pnpm add @prisma/client

# auth
pnpm add better-auth

# safety and plumbing
pnpm add @t3-oss/env-nextjs nanoid next-safe-action
pnpm add @upstash/ratelimit @upstash/redis

# tooling and tests
pnpm add -D vitest @playwright/test prettier husky lint-staged
```

### 3.3 Decision log

| Date | Decision | Alternatives considered | Why | Revisit when |
| --- | --- | --- | --- | --- |
|  | e.g. Postgres on Neon | Supabase, SQLite | Relational data, branching for previews | Traffic exceeds free tier |
|  |  |  |  |  |

### 3.4 Per-endpoint checklist (every Server Action and Route Handler)

- [ ] Session verified on the server (`requireUser()`)
- [ ] Ownership or role checked inside the query
- [ ] Input validated with Zod (body, params, query)
- [ ] Rate-limited if public or costly
- [ ] Consistent response shape and correct status code
- [ ] No secrets or internal errors leaked
- [ ] Safe to retry (idempotent) if it handles money or webhooks
- [ ] Failures logged; test covers the happy path and one failure path

### 3.5 Per-table checklist

- [ ] `id`, `createdAt`, `updatedAt` present
- [ ] Owner column (`userId`) if the data belongs to a user
- [ ] Foreign keys with a deliberate `onDelete` choice
- [ ] Unique constraints for real-world uniqueness (email, slug)
- [ ] Indexes on foreign keys and filtered or sorted columns
- [ ] Money stored as integer minor units; timestamps in UTC
- [ ] Nullable columns are intentional
- [ ] Migration reviewed before it touches production

### 3.6 Debugging routine

| Type of block | Action |
| --- | --- |
| **Memory** (syntax) | Read the docs or ask AI |
| **Knowledge** (concept gap) | Learn the minimum needed, then return to the project |
| **Design** (how to structure) | Sketch two approaches yourself before asking anyone |
| **Bug** | Reproduce → isolate → read the actual error → hypothesize → change **one thing** at a time |

Time-box solo struggle to 20 to 30 minutes, then ask with a minimal reproduction. Use AI to *explain* code, not only to paste it. If you cannot explain a line, you do not own it yet.

### 3.7 Beginner traps

- **Tutorial hell:** you learn by building, not watching.
- **Stack-hopping** in the middle of a project.
- **Premature abstraction and optimization:** repeat yourself three times before abstracting.
- **Perfectionist design** before anything works.
- **Rolling your own auth or crypto.**
- **Scope creep:** every new idea goes to “Later”.
- **Skipping the schema:** bad data models are the hardest thing to fix later.
- **Building horizontally** (all DB, then all API, then all UI) instead of thin end-to-end slices.

---

## The 6 highest-leverage moves

1. **Schema before code.**
2. **Deploy a walking skeleton on day 1.**
3. **Build vertical slices, not horizontal layers.**
4. **Keep core logic pure and tested.**
5. **Check ownership inside every query.**
6. **Ship to real users early.** Feedback beats any plan.
