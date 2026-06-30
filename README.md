# Fintral

**DR Fiscal Compliance SaaS** — Automated e-CF, NCF, Reporte 606 & AI-powered tax assistant for the Dominican Republic.

[![Website](https://img.shields.io/badge/Website-fintral.app-38BDF8?style=flat-square)](https://www.fintral.app)

---

## Overview

Fintral is a comprehensive fiscal compliance platform built for businesses operating in the Dominican Republic. It automates the entire electronic invoicing (e-CF) workflow, NCF management, Reporte 606 filing, and provides an AI-powered tax assistant to help businesses stay compliant with DGII regulations.

## Key Features

- **📄 Electronic Invoicing (e-CF)** — Generate, sign, and submit electronic fiscal documents compliant with DGII requirements
- **🔢 NCF Management** — Track and manage NCF (Numeración Comprobante Fiscal) sequences across multiple entities
- **📊 Reporte 606** — Automated preparation and submission of the monthly fiscal report
- **🤖 AI Tax Assistant** — Smart chat interface powered by Gemini AI for tax-related queries and guidance
- **🏢 Multi-Entity Support** — Manage multiple businesses or subsidiaries under one account
- **📈 Dashboard & Analytics** — Real-time insights into fiscal compliance status, usage metrics, and document history

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | Next.js 15, TypeScript, Tailwind CSS, shadcn/ui |
| **Backend** | FastAPI (Python) |
| **Database** | PostgreSQL |
| **Cache** | Redis |
| **AI** | Google Gemini 2.5 Flash |
| **e-CF Backend** | Alanube API |
| **Auth** | Supabase + bcrypt dual authentication |
| **Infrastructure** | Docker, Nginx, Vercel |

## Architecture

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│   Next.js   │◄────►│   FastAPI    │◄────►│  PostgreSQL  │
│  Frontend   │     │   Backend    │     │   Database   │
└─────────────┘     └──────┬───────┘     └──────────────┘
                           │
                    ┌──────┴───────┐     ┌──────────────┐
                    │  Alanube API │◄────►│   DGII (RD)  │
                    │  (e-CF)     │     │  Tax Auth    │
                    └──────────────┘     └──────────────┘
```

## Pricing

- **Esencial** — $29/mo (50 e-CF, 300 AI queries, 2 users)
- **Profesional** — $69/mo (300 e-CF, 3K AI queries, 5 users)
- **Multi-Entidad** — $149/mo (5K e-CF pool, 10K AI queries, 10 entities)
- **Enterprise** — $299+/mo (custom limits)

15-day free trial available on all plans.

## Links

- **Website:** [https://www.fintral.app](https://www.fintral.app)
- **Documentation:** [https://docs.fintral.app](https://docs.fintral.app) *(coming soon)*

---

*Built with ❤️ for Dominican Republic businesses.*
