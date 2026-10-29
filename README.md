# OpsPilot: AI back-office workspace

One **FastAPI + SQLAlchemy + Supabase** web app that combines three project briefs:

| Module | Brief | Where |
| --- | --- | --- |
| **Invoice checking** | Koda (6b): read supplier invoices, compare with POs and existing records, flag mismatches for review | `/invoices` |
| **Order automation** | Spectra (11): extract purchase/sales orders, validate line items, approve and export for ERP | `/orders` |
| **Projects & tasks** | EM Services (12h): projects, task board, owners, deadlines, progress | `/projects` |

The modules are connected. A flagged invoice or an order that fails validation automatically becomes a task on the
**Invoice Exceptions** / **Order Exceptions** board, and approving or fixing the record closes that task. Everything is
written to a shared audit log.

## Stack

- **FastAPI**: server-rendered pages (Jinja2 templates) plus JSON endpoints under `/api/` (docs at `/api/docs`)
- **SQLAlchemy 2.0** ORM (`opspilot/models.py`) on **Supabase Postgres**
- **Supabase**: Auth (email/password), Storage (private `documents` bucket), Realtime (live task board)
- **OpenAI** (Responses API + Pydantic structured outputs) for document extraction and task suggestions; demo mode without a key
- UI: Tailwind CSS, Alpine.js and Lucide icons from CDNs (no Node build step); PDFs with ReportLab

## Setup

1. **Python 3.12+**, then from the project folder:
   ```bash
   python -m venv .venv
   ```
   ```bash
   .venv\Scripts\python -m pip install -r requirements.txt
   ```
2. **Database schema** (once): push the migrations and demo data to your Supabase project:
   ```bash
   npx supabase db push --include-seed
   ```
   (or paste `supabase/migrations/*.sql` in order, then `supabase/seed.sql`, into the Supabase SQL editor).
3. **`.env.local`** needs:
   | Variable | Where to find it |
   | --- | --- |
   | `NEXT_PUBLIC_SUPABASE_URL` / `SUPABASE_URL` | Supabase → Project Settings → API |
   | `NEXT_PUBLIC_SUPABASE_ANON_KEY` / `SUPABASE_ANON_KEY` | same page (anon / publishable key) |
   | `DATABASE_URL` | Supabase → **Connect** → *Session pooler* string, with your DB password filled in |
   | `SESSION_SECRET` | any long random string (signs the login cookie) |
   | `OPENAI_API_KEY` | optional; leave empty for **demo mode** (mock extraction built from the seeded POs) |
   | `OPENAI_MODEL` | optional, default `gpt-5.4-mini` |
4. For quick demos, turn off **Authentication → Providers → Email → Confirm email** in Supabase.
5. Run:
   ```bash
   .venv\Scripts\uvicorn opspilot.main:app --reload
   ```
   Open http://localhost:8000 and sign up. The **first account becomes admin**.

## CRUD by module

| | Create | Read | Update | Delete |
| --- | --- | --- | --- | --- |
| Invoices | upload up to 100 (AI), enter manually, or build a PDF in the **invoice generator** | review queue, compare page | edit fields/lines → re-check vs PO; approve/reject; reopen | any role |
| Purchase orders | "New purchase order" form (new suppliers created on the fly) | expandable list with lines | — | linked invoices are unlinked, not deleted |
| Product catalogue | add form (new categories on the fly), or from an order line | grouped by category, search | inline name, category, price, active | trash icon |
| Orders | upload up to 100 (AI) or enter manually | tabs by status | edit fields/lines → re-validate; approve; reopen; CSV export | reviewer/admin |
| Projects | new project form | cards with progress | name, brief, due date, status | deletes its tasks too |
| Tasks | add form or AI suggestions | board + filters (live updates) | drag between columns, title, details, owner, priority, due | from the task card |

## Roles

| Role | Can |
| --- | --- |
| staff | upload documents; edit, approve/reject and delete invoices; manage POs and the catalogue; correct orders; manage projects/tasks |
| reviewer | + approve, reopen and delete orders |
| admin | + change roles |

## Project layout

```
opspilot/
  main.py            FastAPI app, middleware, error pages
  config.py, db.py   settings and SQLAlchemy engine/session
  models.py          SQLAlchemy models (tables come from supabase/migrations)
  auth.py            Supabase Auth session + current_user dependency
  supabase.py        Supabase Auth/Storage REST client
  ai.py              OpenAI extraction + task planner (+ demo mocks)
  matching.py        invoice ↔ PO rules (duplicate, supplier, line not on PO, price drift > 1%, over-billed qty, arithmetic, low confidence)
  order_rules.py     order validation, catalogue matching, CSV export
  pdf.py             invoice PDF (ReportLab)
  workflow.py        audit log + exception tasks linking the modules
  routers/           one router per area (invoices, orders, projects, ...)
  templates/         Jinja2 pages; static/app.js for small shared helpers
supabase/            migrations (schema, RLS, storage, realtime) + seed data
```

> The earlier Next.js version of the app is still in `src/` for reference; the FastAPI app above does not use it.
