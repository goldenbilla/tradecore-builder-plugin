---
name: tradecore-report
description: |
  How to add a Report in TradeCore. Use whenever the user asks to build,
  add, or modify a report in the TradeCore app — financial reports,
  stock reports, ledgers, registers, or metadata/Report-Builder reports.
  Encodes the two report kinds (hand-coded backend endpoint + React
  component vs. metadata-driven Report Builder rows), how each is wired
  into the backend, the frontend route, and the sidebar — so a report can
  be added without re-reading the codebase.
---

# Adding a Report in TradeCore

There are **two kinds** of report. Pick the right one first.

| | Hand-coded report | Report Builder (metadata) |
| --- | --- | --- |
| **When** | Complex logic, cross-table joins, special aggregation (Trial Balance, P&L, ledgers, aged AR) | Ad-hoc single-doctype queries the user defines themselves |
| **Defined in** | Python endpoint in `backend/app/routers/reports.py` (or `stock_reports.py`) + a React component | A `report_definitions` DB row created via API/UI |
| **Run via** | `GET /api/v1/c/{company_id}/reports/<name>` | `POST /api/v1/c/{company_id}/report-builder/{id}/run` |
| **You build it** | Yes — code | No — the user clicks "Create Report" in the UI |
| **Seedable from files** | n/a | Only on **custom-app install** (see last section) |

If the user wants a *new analytical screen with bespoke logic*, build a
**hand-coded report** (the common case). If they want to *let users define
their own column/filter combos on an existing doctype*, that's the
**Report Builder**, which already exists — point them at `/reports`.

---

## Building a hand-coded report (the common case)

Five steps. Use an existing report (e.g. General Ledger / Trial Balance)
as the template.

### 1. Backend endpoint — `backend/app/routers/reports.py`

```python
@router.get("/my-report")
async def my_report(
    ctx: CompanyContext,
    db: AsyncSession = Depends(get_tenant_db),
    from_date: date = Query(...),
    to_date: date = Query(...),
    # ...other filters as Query params
):
    check_doc_permission(ctx, "Report", "read")
    company_id = ctx["company_id"]
    # ... SQLAlchemy queries against the tenant DB ...
    return {
        "success": True,
        "data": { "rows": [...], "totals": {...} },
        "meta": { "total": n },
    }
```

Conventions to copy from neighbors in the same file:
- `ctx: CompanyContext` + `db: AsyncSession = Depends(get_tenant_db)` give
  you the tenant-scoped session.
- Gate with `check_doc_permission(ctx, "Report", "read")`.
- Return `{"success": True, "data": {...}, "meta": {...}}`.
- For stock reports, add to `backend/app/routers/stock_reports.py` instead.

### 2. Frontend API method — `frontend/src/lib/api.ts`

Add to the `reportApi(companyId)` factory:

```ts
myReport: (params: Record<string, string>) =>
  api.get(`/c/${companyId}/reports/my-report`, { params }),
```

### 3. React component — `frontend/src/components/reports/MyReport.tsx`

Follow the existing pattern (`AccountLedgerReport.tsx` is a good model):
`"use client"`, `useCompanyStore` for the active company, filter state
(`useState`), `useQuery` keyed on `[name, company?.id, ...filters]` and
`enabled` only when required filters are set, then render filters + table
+ totals. Apply the **tradecore-ui** skill conventions for layout,
`PageHeader`, `DataTable`, `EmptyState`, etc.

### 4. Wire it into the reports page — `frontend/src/app/(app)/reports/page.tsx`

The `/reports` page is a hub that switches on a `?report=` URL param. Add:
- your id to the `ReportId` union,
- a view function (or import your component),
- a `case` in the switch that renders it.

For a standalone route instead, create
`frontend/src/app/(app)/reports/my-report/page.tsx` (mirror an existing
sub-route like `reports/stock-ledger/page.tsx`).

### 5. Sidebar link (optional) — `frontend/src/components/layout/Sidebar.tsx`

Add an entry to `REPORTS_GROUP.children`:

```ts
{ id: "reports-my", href: "/reports/my-report", icon: BarChart3, label: "My Report" },
```

### Verify

Run the dev server and load the report via the preview workflow: check
the network call returns `success: true`, the table renders, and filters
re-query. Fix and re-check before reporting done.

---

## Report Builder (metadata reports)

Already fully built — you usually **don't write code**, you tell the user
to use it, or you create a definition via the API.

- **UI:** the `/reports` page has Create / List / Run / Edit / Schedule
  controls for Report Builder reports.
- **API:** `backend/app/routers/report_builder.py`
  (prefix `/api/v1/c/{company_id}/report-builder`):
  - `GET /doctypes/{doctype_name}/fields` — fields available as columns
  - `GET ` / `POST ` / `PUT /{id}` / `DELETE /{id}` — CRUD report defs
  - `POST /{id}/run` — execute with runtime filters (`{filters, limit, offset}`)
  - `POST /{id}/export/{xlsx|pdf}` — export
  - `GET|POST /schedules`, `POST /schedules/{id}/run-now` — scheduling
- **Create body** (`ReportBody`): `name`, `doctype_name`, `columns`,
  `filters`, `group_by`, `sort_by`, `chart`, `disabled`.
- **Model:** `backend/app/models/report_builder.py` `ReportDefinition`
  (per-company `report_definitions`, unique on `company_id` + `name`).
  `report_type` is always `"query"` — **no raw SQL**, it's metadata-driven.

### File-based report seeding (custom apps only)

You can ship report definitions inside a custom app at
`backend/app/apps/<app>/reports/<name>.json`. They are discovered by
`app_manifest._load_reports()` and inserted into `report_definitions`
**when the custom app is installed** (`app/custom_app/installer.py`).

> **Gotcha:** the installer reads `rep["doctype"]` — **not**
> `doctype_name` — plus `report_type`, `columns`, `filters`, `group_by`,
> `sort_by`, `chart`. Match those exact keys or install fails:

```json
{
  "name": "Sales by Region",
  "doctype": "Sales Invoice",
  "report_type": "query",
  "columns": [],
  "filters": [],
  "group_by": [],
  "sort_by": [],
  "chart": {}
}
```

This path only runs on app install — it does **not** retro-seed already
installed tenants, and the system apps' `reports/` folders are currently
empty.

## Key files

| File | Role |
| --- | --- |
| `backend/app/routers/reports.py` | hand-coded financial reports |
| `backend/app/routers/stock_reports.py` | hand-coded stock reports |
| `backend/app/routers/report_builder.py` | metadata Report Builder API |
| `backend/app/models/report_builder.py` | `ReportDefinition` model |
| `frontend/src/lib/api.ts` | `reportApi`, `reportBuilderApi` |
| `frontend/src/app/(app)/reports/page.tsx` | reports hub (switches on `?report=`) |
| `frontend/src/components/reports/` | report components (e.g. `AccountLedgerReport.tsx`) |
| `frontend/src/components/layout/Sidebar.tsx` | `REPORTS_GROUP` nav |
