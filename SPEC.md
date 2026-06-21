# Money Manager — Project Spec

> A personal money manager that teaches new adults financial responsibility while
> quietly handling the analysis. Use-shaped, not demo-shaped.

**Status:** Locked (v1)
**Target user:** College students and new adults learning to manage money for the first time.

---

## 1. Core Principles

- **Use-shaped, not demo-shaped.** Runs on its own against real data; only surfaces
  something when there's an action worth taking. Intelligence shows up as *restraint*,
  not a stream of reasoning steps.
- **Human-in-the-loop on money.** The app reads everything and *recommends* or
  *executes with a human tap* — it never autonomously moves real money between bank
  accounts.
- **Math is trustworthy; AI is human.** Deterministic rules/optimization do the
  number-work. The LLM only handles fuzzy judgments, explanations, and drafting.
- **Educational by default.** Every recommendation teaches the underlying concept.

---

## 2. Feature Pillars

### 2.1 Auto Payment Analysis  *(first feature — build to completion first)*
- Detect recurring charges from the transaction stream (periodicity detection).
- Flag forgotten / unused subscriptions.
- Catch suspicious or scam-like charges.
- No money movement. Useful from day one.

### 2.2 Adaptive Budgeting
- **Start from reality:** initial target per discretionary category = rolling *median*
  of recent spend (median resists one-off large purchases).
- **Glide path:** each cycle, move the target toward the goal in capped steps
  (≤ ~5–8% tighter per period). Real change, never a shock.
- **Feedback loop:**
  - Hit target → take next step down.
  - Missed but closer → hold or take a smaller step (reward progress).
  - Blew past → ease target back toward actual so it stays believable.
- **Cold start:** first cycle is observe-only (target = baseline, no tightening).
- Fixed categories (rent, loans) do not adapt; only discretionary ones glide.

### 2.3 Savings Goals
- Save toward specific things (laptop, trip, emergency fund).
- "Execution" = a **virtual envelope ledger** that allocates and tracks goals, or
  sweeps surplus to the investing side. No unsupervised bank-to-bank transfers.
- A savings goal can *drive* the budgeting glide path ("to hit $600 by September,
  here's the gentle tightening across discretionary categories").

### 2.4 Auto-Invest  *(goal-aware optimizer, not naive surplus-dumping)*
- **Horizon-based allocation:** map each goal's amount + deadline to a risk-appropriate
  mix (near-term → cash/short-term treasuries; long-term → can take equity risk).
- **Goal feasibility math:** compute required rate of return; flag unrealistic goals and
  suggest saving more or extending the timeline instead of chasing risk.
- **Portfolio optimization:** mean-variance (Markowitz) optimization over a *safe, fixed
  universe*. Uses transparent long-run expected-return/volatility **assumptions per asset
  class** — principled diversification, NOT return forecasting.
- **Automatic glide-path de-risking:** shift toward bonds/cash as a goal's deadline nears.
- **Dollar-cost averaging:** invest on a schedule, never market-timing.
- Every trade gated behind a human tap. Paper trading first.

### 2.5 Bills & Payments
- Detect upcoming bills; warn ahead of shortfalls ("rent's due Friday, you're $40 short").
- Schedule / remind with one-tap approval. Rides bank autopay or a deliberate tap —
  never autonomous payment.

### 2.6 Student Layer
- Student-loan tracking and payment scheduling.
- Goal-saving for specific things; upcoming-obligation awareness.
- Framed to teach the underlying responsibilities.

### 2.7 The Auto-Engine (under all of the above)
- A scheduled background job that ingests new transactions, recategorizes, recomputes
  budgets, detects bills/shortfalls, runs analysis, and produces a short **actionable
  digest**. Feels agentic because it runs unattended, is right, and stays quiet otherwise.

---

## 3. Intelligence Approach
- **Rules + categorization** for structured number-work (deterministic, testable).
- **LLM** for fuzzy judgments, plain-language explanations, drafting recommendations.
- **pgvector (later):** "ask questions about my spending" (RAG).
- Optional small trained categorizer (scikit-learn) where rules fall short.

---

## 4. Data Sources
- **Plaid** (sandbox / Development tier, own accounts) — read-only transactions & balances.
- **Alpaca** — investing (paper first; individual API access enables real later).

---

## 5. Money-Movement Boundary (shapes the data model)
- ✅ Read accounts, balances, transactions.
- ✅ Recommend any action.
- ✅ Execute investments via Alpaca **on one-tap approval**.
- ✅ Track savings via virtual envelopes.
- ❌ Autonomously move real money between bank accounts.
- ❌ Predict ROI of specific assets / market-time.

---

## 6. Tech Stack
- **Language:** TypeScript end-to-end; Python for the analyzer service.
- **Web/product:** Next.js (App Router) + Tailwind, TanStack Query.
- **Data:** PostgreSQL + Prisma (migrations). pgvector later.
- **Jobs:** BullMQ + Redis (scheduled jobs, retries, idempotent webhook handling).
- **Analyzer:** Python + FastAPI (detection, categorization, cvxpy/scipy/numpy optimization).
- **Auth:** Clerk.
- **Quality:** Vitest + Playwright + pytest; GitHub Actions CI.
- **Infra:** Docker (worker/analyzer); one real AWS touch; Sentry for observability.
- **Security:** encrypted access tokens at rest, secrets management, rate limiting,
  input validation, no sensitive data in logs.

**Deliberately skipped:** Kubernetes (overkill for a solo app), Redux (legacy vs modern
React + TanStack Query).

---

## 7. Architecture
```
apps/web        Next.js — product + API routes
apps/worker     Node/TS — BullMQ jobs, scheduling, orchestration
apps/analyzer   Python/FastAPI — detection, categorization, portfolio optimization
packages/db     Prisma schema + client + migrations
packages/shared shared TS types
docker-compose  Postgres (+ Redis)
```
The three-service split (product / orchestration / analysis) is what creates the
"how services communicate, how failures propagate" story.

---

## 8. Build Order (no layer added until the one before it runs)
1. **Spine:** monorepo + Next app boots + Postgres + Prisma + one table rendered E2E.
2. Real schema (accounts, transactions, recurring_charges, budget_target, goals, loans).
3. Plaid sandbox ingestion into Postgres.
4. **Killer feature first:** recurring/subscription + scam detection → actionable digest.
   (Build logic in TS to move fast; extract to the Python analyzer once proven.)
5. BullMQ + Redis: run analysis on a schedule; idempotent webhook handling.
6. Auth (Clerk) — make it multi-user / yours.
7. Adaptive budgeting (glide path + feedback loop).
8. Savings goals (virtual envelopes).
9. Auto-invest (goal-aware optimizer in the analyzer service).
10. Hardening: tests, CI, token encryption, Docker, AWS touch, Sentry.

---

## 9. Non-Goals (v1)
- Autonomous movement of real money between bank accounts.
- Predicting returns of specific assets / market timing.
- Individual stocks, options, crypto, leverage in the investing universe.
- Public/multi-tenant scale concerns (Kubernetes, etc.).
