<div align="center">

# Zero-Gap-AI

### A resilient, edge-native AI platform built on a multi-provider inference layer

**React 19 · TanStack Start · Cloudflare Workers · Supabase · AMD MI300X + Groq**
*Status: in active development*

</div>

---

## Overview

**Zero-Gap-AI** is an edge-deployed, TypeScript-first web platform designed around a single core idea: **AI inference should never go down.** Instead of depending on one model provider, the system routes chat/completions through a **primary self-hosted inference cluster on AMD MI300X GPUs (via vLLM)**, and transparently fails over to **Groq Cloud** the moment the primary path degrades or errors — closing the "gap" between a provider outage and a broken user experience.

The application is built as a full-stack, type-safe React app on **TanStack Start**, deployed globally on **Cloudflare Workers** for low-latency, serverless edge execution, with **Supabase** handling authentication, database, and realtime data.

> This repository currently ships the production-grade foundation — architecture, tooling, dependency stack, deployment pipeline, and environment contracts — with feature UI actively being built out on top of it.

---

## Why This Project Stands Out

- **Provider-agnostic AI resilience** — an OpenAI-compatible inference contract lets the app swap between self-hosted (AMD MI300X / Llama 3.3 70B) and cloud (Groq) backends without touching application code, with automatic failover on 5xx errors.
- **Edge-first architecture** — deployed on Cloudflare Workers rather than a traditional server, meaning near-zero cold starts and requests served from the datacenter closest to the user.
- **Modern, type-safe full stack** — React 19, TanStack Router/Query, and Zod-validated forms give end-to-end type safety from the UI down to data fetching.
- **Production tooling from day one** — ESLint + Prettier, Vitest-ready config, Bun-first package management, and a documented secrets pipeline via Wrangler — not a throwaway prototype.
- **Data-rich UI foundation** — the dependency set (Recharts, Chart.js, shadcn/ui's full Radix primitive set, embla-carousel, cmdk) signals a dashboard-grade product: analytics views, command palettes, and document workflows (PDF parsing via `pdfjs-dist`, Word doc parsing via `mammoth`) are first-class citizens of the design.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         Client (Browser)                          │
│   React 19 UI · TanStack Router · TanStack Query · shadcn/ui      │
└───────────────────────────────┬───────────────────────────────────┘
                                 │ HTTPS
┌───────────────────────────────▼───────────────────────────────────┐
│                     Cloudflare Workers (Edge)                      │
│         TanStack Start server-entry · nodejs_compat runtime        │
│                                                                     │
│   ┌───────────────────────────────────────────────────────────┐   │
│   │                  AI Inference Router                       │   │
│   │                                                              │
│   │   Request → AMD MI300X vLLM endpoint (primary)              │   │
│   │                  │  OpenAI-compatible /v1/chat/completions   │   │
│   │                  │  model: llama-3.3-70b                     │   │
│   │                  ▼                                           │   │
│   │           5xx / timeout / unavailable?                       │   │
│   │                  │  yes                                       │   │
│   │                  ▼                                           │   │
│   │            Groq Cloud (fallback)                             │   │
│   └───────────────────────────────────────────────────────────┘   │
└───────────────────────────────┬───────────────────────────────────┘
                                 │
┌───────────────────────────────▼───────────────────────────────────┐
│                            Supabase                                 │
│         Auth · PostgreSQL · Realtime · Row-Level Security           │
└───────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer            | Technology                                                                 |
| ------------------ | ----------------------------------------------------------------------------- |
| Framework         | React 19 + TanStack Start (file-based routing via TanStack Router)            |
| Build tool        | Vite 7                                                                        |
| Deployment        | Cloudflare Workers (`@cloudflare/vite-plugin`, Wrangler, `nodejs_compat`)      |
| Styling           | Tailwind CSS 4 + shadcn/ui (Radix UI primitives, `tw-animate-css`)            |
| Server state      | TanStack Query v5                                                             |
| Forms/Validation  | React Hook Form + Zod + `@hookform/resolvers`                                 |
| Backend/Auth      | Supabase (`@supabase/supabase-js`) — Postgres, Auth, Realtime                 |
| AI inference      | AMD Developer Cloud (MI300X, vLLM, OpenAI-compatible) — primary · Groq Cloud — automatic fallback |
| Data viz          | Chart.js + `react-chartjs-2`, Recharts                                        |
| Document handling | `mammoth` (DOCX parsing), `pdfjs-dist` (PDF parsing)                          |
| UX/Interaction    | Framer Motion, `embla-carousel-react`, `cmdk` (command palette), `sonner` (toasts), `vaul` (drawers), `canvas-confetti` |
| Code quality      | ESLint 9 + Prettier + `typescript-eslint`                                     |
| Package manager   | Bun (`bun.lockb`, `bunfig.toml`) with npm lockfile compatibility              |
| Language          | TypeScript, strict mode                                                       |

---

## AI Inference Layer

The platform's defining engineering decision is its **dual-provider inference strategy**, configured entirely through environment secrets — no code changes needed to swap providers:

| Priority | Provider | Model | Use case |
|---|---|---|---|
| 1 — Primary | AMD Developer Cloud (MI300X GPU, vLLM server) | `llama-3.3-70b` | Self-hosted, cost-efficient, high-throughput inference |
| 2 — Fallback | Groq Cloud | Groq-hosted LPU inference | Kicks in automatically on AMD 5xx errors or downtime |

Both providers speak the **OpenAI-compatible chat completions API**, so the routing layer is a thin, swappable abstraction rather than provider-specific glue code.

---

## Prerequisites

- Node.js 18+ (or [Bun](https://bun.sh) — the repo is Bun-first)
- A [Supabase](https://supabase.com) project
- An AMD Developer Cloud vLLM endpoint (or a [Groq](https://groq.com) API key to run on fallback-only)
- A [Cloudflare](https://www.cloudflare.com) account + [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/) for deployment

---

## Getting Started

### 1. Install dependencies

```bash
bun install
# or
npm install
```

### 2. Configure environment variables

Copy `.env.example` to `.env` and fill in your values:

```env
# AMD Developer Cloud — PRIMARY (MI300X vLLM, OpenAI-compatible)
AMD_API_URL=http://<YOUR_DROPLET_IP>:8000/v1/chat/completions
AMD_API_KEY=your-vllm-api-key-here
AMD_MODEL=llama-3.3-70b

# Groq Cloud — FALLBACK (used when AMD is unavailable or errors 5xx)
GROQ_API_KEY=your-groq-api-key-here

# Supabase
SUPABASE_URL=
SUPABASE_PUBLISHABLE_KEY=
VITE_SUPABASE_URL=
VITE_SUPABASE_PUBLISHABLE_KEY=
VITE_SUPABASE_PROJECT_ID=
```

> `.env` is git-ignored by default — never commit real secrets.

### 3. Run the dev server

```bash
bun run dev
```

The app runs locally via Vite's dev server with the TanStack Start plugin.

---

## Deployment

Zero-Gap-AI ships ready for **Cloudflare Workers**. Push your secrets to the edge before deploying:

```bash
wrangler secret put AMD_API_URL
wrangler secret put AMD_API_KEY
wrangler secret put AMD_MODEL
wrangler secret put GROQ_API_KEY
```

Then build and publish:

```bash
bun run build
wrangler deploy
```

The Worker entry point is TanStack Start's `server-entry`, configured with the `nodejs_compat` compatibility flag for full Node API support at the edge.

---

## Available Scripts

```bash
bun run dev          # Start the Vite dev server
bun run build         # Production build
bun run build:dev     # Development-mode build (unminified, for debugging)
bun run preview        # Preview the production build locally
bun run lint            # Run ESLint across the project
bun run format           # Format the codebase with Prettier
```

---

## Project Structure

```
.
├── .env.example         # Documented environment contract (AMD, Groq, Supabase)
├── wrangler.jsonc        # Cloudflare Workers deployment + secrets config
├── vite.config.ts        # Vite + Cloudflare Workers + TanStack Start integration
├── components.json       # shadcn/ui generator config
├── eslint.config.js       # Linting rules (ESLint 9 flat config)
├── .prettierrc             # Formatting rules
├── package.json            # Scripts & dependency manifest
└── src/                     # Application code (routes, components, AI router, hooks)
```

---

## Roadmap

- [ ] Chat interface consuming the dual-provider AI router
- [ ] Supabase-backed authentication & user sessions
- [ ] Analytics dashboard (Recharts / Chart.js)
- [ ] Document upload & parsing pipeline (PDF/DOCX ingestion)
- [ ] Command palette (`cmdk`) for power-user navigation
- [ ] Automated test suite (Vitest)
- [ ] CI/CD pipeline for Cloudflare deployments

---

## Engineering Highlights (at a glance)

| Concern | Approach |
|---|---|
| **Availability** | Multi-provider AI failover eliminates single points of failure for inference |
| **Latency** | Edge deployment on Cloudflare Workers puts compute close to users globally |
| **Type safety** | TypeScript + Zod end-to-end, from form input to API boundary |
| **DX** | Bun-first tooling, strict linting/formatting, fast Vite HMR |
| **Scalability** | Serverless edge runtime scales automatically with traffic, no server management |

---

<div align="center">

Built with React 19 · TanStack Start · Cloudflare Workers · Supabase · AMD MI300X · Groq

</div>
