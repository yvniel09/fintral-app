# Fintral

**Automated fiscal compliance for the Dominican Republic.** e-CF generation, NCF management, Reporte 606 filing, DGII validation, and AI-powered tax assistance.

[![CI](https://github.com/yvniel09/fintral-app/actions/workflows/ci.yml/badge.svg)](https://github.com/yvniel09/fintral-app/actions)
[![Website](https://img.shields.io/badge/fintral.app-38BDF8?style=flat-square&logo=nextdotjs&logoColor=white)](https://www.fintral.app)
[![Stack: Next.js + FastAPI](https://img.shields.io/badge/Stack-Next.js_15_•_FastAPI_•_PostgreSQL-09090b?style=flat-square)](https://github.com/yvniel09/fintral-app)

---

## The Problem

The Dominican Republic's **DGII** (Dirección General de Impuestos Internos) mandates electronic invoicing (e-CF) for all tax-registered businesses. The ecosystem is fragmented:

- **No unified API** — e-CF generation, NCF control, and Reporte 606 are separate systems with different formats
- **Proprietary XML signing** — requires XAdES-BES digital signatures compliant with DGII's specific schema
- **Web-scraped validation** — DGII's public portal is an ASP.NET WebForms app with `__VIEWSTATE` tokens and no documented API
- **Strict deadlines** — Reporte 606 must be filed monthly with penalties for late submissions
- **Multi-entity chaos** — accounting firms manage dozens of clients, each with unique NCF ranges and fiscal regimes

Fintral solves this by providing a **single platform** that abstracts all DGII complexity behind a clean SaaS interface.

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                     Nginx (Proxy)                      │
│    ┌──────────────┐            ┌──────────────────┐   │
│    │  Next.js 15   │◄─────────►│   FastAPI (Py)    │   │
│    │  App Router   │  REST/WS  │   Clean Arch      │   │
│    │  shadcn/ui    │           │   Repository      │   │
│    │  Tailwind     │           │   Service Layer   │   │
│    └──────┬───────┘           └────┬───┬───────────┘   │
│           │                        │   │               │
│           │                 ┌──────┘   └──────┐        │
│           │                 ▼                  ▼        │
│           │          ┌──────────┐      ┌──────────┐    │
│           │          │PostgreSQL│      │  Redis   │    │
│           │          │ (5440)   │      │ (6381)   │    │
│           │          └──────────┘      └──────────┘    │
└───────────┼────────────────────────────────────────────┘
            │
     ┌──────┴────────────────────────────────────┐
     ▼                    ▼                       ▼
┌──────────┐    ┌──────────────┐    ┌──────────────────┐
│ Alanube  │    │  DGII Portal │    │  Intuit / Xero   │
│ (e-CF    │    │ (web scrape) │    │  (accounting      │
│  backend)│    │  + QR scan   │    │   integration)    │
└──────────┘    └──────────────┘    └──────────────────┘
```

### Why FastAPI + Next.js?

- **FastAPI** provides Python's rich ML/OCR ecosystem (OpenCV, Tesseract, pyzbar) for document processing — critical for invoice data extraction from photos, PDFs, and scanned QR codes
- **Next.js 15 App Router** gives us React Server Components for SEO (public landing, docs) and client components for the dashboard — no context-switching between frameworks
- **Dual auth (Supabase + bcrypt)** — Supabase handles social login and session management; bcrypt fallback ensures zero external dependency for core authentication during development and self-hosted deployments

### DGII Integration Strategy

The DGII has no official API, so we use three complementary approaches:

| Approach | What | Technique | Anti-blocking |
|----------|------|-----------|---------------|
| **Alanube API** | e-CF generation & submission | REST API with JWT auth (reseller model, ~$0.0088/doc overage) | Telemetry per call |
| **DGII Web Scraper** | NCF physical validation | ASP.NET WebForms scraper (`__VIEWSTATE` extraction) | 500ms rate limit, on-demand only, result caching |
| **QR Code Validation** | e-CF authenticity check | Multi-strategy CV: CLAHE, upscaling, OTSU, pyzbar | 10 strategies cascade |

The QR validation engine alone uses **10 fallback strategies** (CLAHE contrast enhancement, multi-scale upscaling, adaptive thresholding, morphological closing, pyzbar with 7 preprocessing variants) to decode real-world QR codes from photos with poor lighting, blur, or angled capture.

## Key Features

### Electronic Invoicing (e-CF)
- Generates signed XML invoices compliant with DGII e-CF schema (XAdES-BES)
- Alanube reseller backend with pooled document quotas across entities
- Telemetry-tracked submission pipeline with automatic retries
- Supports NCF types: 01 (final consumer), 02 (credit note), 04 (debit note), 11 (minor expenses), 12 (e-CF unique), 14 (savings), 15 (guest)

### NCF Management
- Track NCF ranges per entity with automatic consumption tracking
- Real-time balance alerts when ranges approach depletion
- Integration with DGII portal for NCF status validation (active/cancelled/voided)
- Server Action pattern to bypass CORS on DGII web scrape calls

### Reporte 606 / 607 / 608
- Auto-generate monthly fiscal reports from invoice data
- Excel export with DGII's required column format
- NCF-by-NCF validation against DGII records before submission
- Support for credit notes and cancelled invoice reconciliation

### AI Tax Assistant
- Context-aware chat over the full fiscal compliance lifecycle
- Powered by Google Gemini 2.5 Flash (multi-provider abstraction via `LLMProvider` ABC)
- RAG over user's own invoice history for personalized responses
- Answers queries like "What NCF type do I need for professional services?" or "Why was my e-CF rejected?"

### Multi-Entity & Team Management
- Single account manages multiple RNCs (business entities)
- Granular RBAC: owner, admin, member roles
- Pooled e-CF quotas across entities under one subscription
- Audit log for all compliance actions (who submitted what, when)

### Accounting Integrations
- **QuickBooks Online** — Push vendor bills via OAuth 2.0 (`com.intuit.quickbooks.accounting` scope)
- **Xero** — API-based invoice synchronization
- **Odoo** — ERP connector module

### WhatsApp Invoicing
- Accept invoice photos via WhatsApp (Evolution API gateway)
- Multi-strategy OCR: OpenCV preprocessing → LLM vision extraction → structured data
- Automatic duplication detection via Redis
- Supabase Storage for uploaded file archival

## Technical Decisions & Trade-offs

### Server Actions as CORS Proxy
DGII's portal doesn't support CORS. Instead of a dedicated proxy service, we use **Next.js Server Actions** (`consultRncAction`, `searchByNameAction`) as the CORS bridge. This eliminates an extra hop while keeping secrets server-side.

### Invoice Processing Pipeline
A modular pipeline with pluggable stages — each stage can be maintained and tested independently:

```
Image Preprocessor → Classifier (e-CF / PDF / Photo / XML)
  → Normalizer → AI Extractor (LLM vision fallback)
  → Post-Extraction Validator → Categorizer
  → Pipeline Orchestrator
```

The pipeline includes OCR character corrections (`O→0`, `I→1`, `S→5`, etc.) tailored to invoice scans, and cascading from high-confidence structured extraction → LLM vision → full OCR.

### PWA Offline Support
Using **Serwist** (workbox fork) for service worker caching, plus **Dexie.js** (IndexedDB) for offline invoice data — the app works without connectivity and syncs when back online.

### Cost Control Architecture
Usage tracking is baked into every operation. AI chat has hard caps per plan; e-CF overage uses soft limits (never blocks — sends warning). Redis-backed rate limiting on WhatsApp ingestion prevents runaway costs.

## Engineering Practices

- **CI/CD**: GitHub Actions runs lint (ESLint + Ruff), TypeScript type-check, build, and pytest on every PR
- **Pre-push hooks**: Local CI via `act` before pushing, skippable with `--no-verify`
- **Testing**: 20+ backend test files covering pipeline processing, DGII validation, AI chat, webhooks, exports, and QR validation
- **Conventional commits**: `type(scope): description` format across all history
- **TypeScript strict mode**: `tsc --noEmit` in CI
- **React Doctor**: Automated component diagnostics via `react-doctor`
- **pnpm**: Fast, disk-efficient package management with frozen lockfiles

## Project Structure

```
frontend/                         # Next.js 15 App Router
├── app/
│   ├── dashboard/               # Main dashboard pages
│   ├── dgii-rnc/                # RNC validation & DGII consultation
│   ├── upload/                  # Invoice upload portal
│   ├── billing/                 # Subscription management
│   ├── actions/                 # Server Actions (CORS proxy)
│   └── auth/                    # Supabase + bcrypt login
├── features/
│   ├── dgii/                    # Reporte 606/607/608 features
│   ├── invoices/                # Invoice management
│   ├── store/                   # POS / store checkout
│   ├── chat/                    # AI tax assistant
│   └── dashboard/               # Analytics widgets
├── components/ui/               # shadcn/ui components
└── lib/
    ├── api/                     # API client layer
    └── services/                # Client services (DGII, etc.)

backend/                         # FastAPI
├── app/
│   ├── core/                    # Redis, auth, permissions, DI container
│   ├── models/                  # SQLAlchemy models
│   ├── services/
│   │   ├── alanube.py           # e-CF API reseller client
│   │   ├── dgii_scraper.py      # DGII WebForms scraper
│   │   ├── dgii_validation.py   # QR validation (10 strategies)
│   │   ├── pipeline/            # Invoice processing pipeline stages
│   │   ├── pipeline_orchestrator.py
│   │   ├── quickbooks_connector.py
│   │   ├── xero_connector.py
│   │   ├── whatsapp.py          # WhatsApp + OCR ingestion
│   │   ├── llm_providers.py     # Abstract LLM provider interface
│   │   └── ai_chat_service.py   # AI assistant backend
│   ├── routers/                 # API endpoints
│   └── utils/                   # Date parsing, etc.
├── tests/                       # 20+ test files
└── Dockerfile

infra/
├── compose.yml                  # PostgreSQL + Redis + Nginx
├── docker-compose.dev.yml
├── nginx/                       # Reverse proxy config
└── scripts/                     # Setup, invitations, DB seeding
```

## Getting Started (Development)

```bash
# Clone
git clone https://github.com/yvniel09/fintral-app.git
cd fintral-app

# Backend
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
cp .env.example .env          # Configure your environment
uvicorn app.main:app --reload --port 8000

# Frontend
cd frontend
pnpm install
pnpm dev                      # → http://localhost:3000

# Or use Docker
docker compose -f compose.yml up
```

## What Makes This Project Interesting

- **Real production SaaS** — not a demo or tutorial. Handles real DGII compliance for Dominican businesses
- **Solves a genuine infrastructure gap** — DR's tax authority has no API, so we built our own integration layer
- **Full-stack engineering** — from OpenCV image processing to multi-provider LLM abstraction to React Server Components
- **Pragmatic trade-offs** — chose Server Actions over a dedicated proxy, dual auth over lock-in, pooled NCF quotas over per-entity billing
- **Design-conscious** — the UI is inspired by Stripe's design system with deliberate typography, color, and spacing decisions documented in `DESIGN.md`

## Pricing

| Plan | Price | e-CF | AI Queries | Users |
|------|-------|------|------------|-------|
| **Esencial** | $29/mo | 50 | 300 | 2 |
| **Profesional** | $69/mo | 300 | 3K | 5 |
| **Multi-Entidad** | $149/mo | 5K pooled | 10K | 10 entities |
| **Enterprise** | $299+/mo | Custom | Custom | Custom |

15-day free trial on all plans.

---

*Built for Dominican Republic businesses navigating DGII fiscal compliance.*

[![Website](https://img.shields.io/badge/fintral.app-38BDF8?style=flat-square&logo=nextdotjs&logoColor=white)](https://www.fintral.app)
[![GitHub](https://img.shields.io/badge/GitHub-yvniel09-09090b?style=flat-square&logo=github)](https://github.com/yvniel09)
