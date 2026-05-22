---
name: tradecore-doctype
description: |
  How to create or modify a DocType in the TradeCore backend
  (multi-tenant, ERPNext-style). Use whenever the user asks to add,
  change, or review a doctype, child table, controller, or document
  business logic (before_save / before_submit / GL / stock posting) in
  the TradeCore backend. Encodes the doctypes/ folder layout (bundled
  core apps vs external code-backed apps), the JSON field schema, the
  controller hooks, how dual-source discovery + per-company install work,
  and the multi-tenant gotchas — so a doctype can be authored without
  re-reading the codebase.
---

# Creating a DocType in TradeCore

TradeCore is a multi-tenant, ERPNext-style platform on a **single shared
database** (companies separated by `company_id`). DocTypes are
**code-authored as files** in an app's `doctypes/` folder and become
**GLOBAL definitions** (`company_id IS NULL`, defined once) — exactly like
Frappe. You almost never touch a database or write a migration to add one.

## Two kinds of app — pick first

| | **Bundled (system) app** | **External (custom) app** |
| --- | --- | --- |
| Lives in | the platform image: `backend/app/apps/<app>/` (e.g. `core`) | its **own git repo**, mounted at `EXTERNAL_APPS_DIR/<namespace>/` (default `/app/external_apps/<namespace>/`, e.g. `biomass`) |
| `app.json` `type` | `"system"` | `"custom"` |
| DocType definitions | **global** (`company_id NULL`), seeded once from code | **also global** (`company_id NULL`), seeded once from code |
| Reaching companies | auto-activated for every company | **opt-in** — an admin installs (activates) it per company |

Both kinds define their DocTypes **once, globally** (the ERPNext "define
once" model). Install does **not** copy DocTypes per company — it just
**activates** the app for a company. New business features should be
**external custom apps** (their own repo, opt-in). Only truly universal
objects go in bundled `core`.

## Mental model

- **A doctype = a folder** at `<app>/doctypes/<snake_name>/` containing:
  - `<snake_name>.json` — field metadata (**required**)
  - `<snake_name>.py` — controller with business logic (**optional**)
  - `test_<snake_name>.py` — test stub (optional)
- An **app folder** has `app.json` + the standard subfolders
  `doctypes/ reports/ print_formats/ fixtures/ dashboards/`. (Print
  formats and fixtures install too — see the `tradecore-print-format`
  skill.)
- **Discovery is automatic and dual-source.** At startup
  `app/services/app_manifest.py:discover_apps()` scans **both** the
  bundled `app/apps/` dir **and** `EXTERNAL_APPS_DIR`, tagging each app
  `origin = bundled | external`. Controllers are file-loaded from both by
  `app/doctype_engine/controller.py:_discover_controllers()` so the
  `@register_controller` decorator fires regardless of where the app
  lives. **No import registration, no `SYSTEM_DOCTYPES` edit, no
  `__init__.py` needed.** (A bundled app wins on a namespace collision.)
- **How DocTypes get created:** ALL app DocTypes (bundled + external) are
  seeded as **global** rows from code by
  `platform_app_seed.py:_upsert_global_doctypes` (the same global path
  `doctype_seed.py` uses for `core`). **Install does NOT copy them** —
  `app/custom_app/installer.py:install_app` only records an *activation* and
  materialises per-company resources (print formats, reports, fixtures).
  Uninstall leaves the global DocType intact.
- **Applying changes (the `bench migrate` equivalent):** edit the JSON →
  `python -m app.migrate` (alembic upgrade + reconcile global defs + reload
  registry) **or** click **Reload** in Custom Apps (no restart). Controller
  `.py` changes need a worker restart, like Frappe's `bench restart`.

## Step 1 — Decide which app it belongs to

- A truly universal, always-present object → bundled `core`:
  `backend/app/apps/core/doctypes/<snake_name>/`.
- A feature companies opt into (the common case) → an **external custom
  app**: a folder `doctypes/<snake_name>/` inside that app's own repo,
  which is mounted at `/app/external_apps/<namespace>/`.

**Creating a new external custom app** = a new git repo whose **root is
the app folder**: `app.json` + the standard subfolders. Minimal `app.json`:

```json
{
  "type": "custom",
  "namespace": "myapp",
  "name": "My App",
  "version": "1.0.0",
  "author": "TradeCore Platform",
  "status": "published",
  "description": "…"
}
```

Then create `doctypes/ reports/ print_formats/ fixtures/` alongside it
(use the `biomass` repo as the reference layout). Deploy = clone the repo
on the host and mount it at `/app/external_apps/<namespace>` (`:ro`) on
the backend **and** celery_worker; restart → it registers as `managed`.
The legacy in-monorepo scaffolder (`app/services/app_scaffold.py`) still
exists for bundled apps, but new custom apps live in their own repo.

## Step 2 — Write `<snake_name>.json`

The folder name and JSON filename must be the **snake_case** form of the
doctype `name`. Full example (parent doctype with a child table):

```json
{
  "name": "Sales Invoice",
  "module": "Selling",
  "is_submittable": true,
  "autoname": "SI-.YYYY.-.####",
  "title_field": "customer_id",
  "controller_class": "SalesInvoiceController",
  "fields": [
    {"fieldname": "posting_date", "label": "Posting Date", "fieldtype": "Date", "sort_order": 0, "reqd": true, "in_list_view": true},
    {"fieldname": "customer_id",  "label": "Customer",     "fieldtype": "Link", "sort_order": 1, "options": "Customer", "reqd": true, "in_list_view": true, "in_standard_filter": true},
    {"fieldname": "due_date",     "label": "Due Date",     "fieldtype": "Date", "sort_order": 2, "in_list_view": true},
    {"fieldname": "items",        "label": "Items",        "fieldtype": "Table", "sort_order": 4, "options": "Sales Invoice Item"},
    {"fieldname": "tax_type",     "label": "Tax Type",     "fieldtype": "Select", "sort_order": 6.5, "options": "Exclusive\nInclusive", "default_value": "Exclusive"},
    {"fieldname": "total_amount", "label": "Total Amount", "fieldtype": "Currency", "sort_order": 10, "read_only": true, "in_list_view": true},
    {"fieldname": "terms_content","label": "Terms & Conditions", "fieldtype": "Text", "sort_order": 20}
  ]
}
```

Child table doctype — same shape but `"istable": true` and no `items`
field. Its `name` is what the parent's `Table` field references in
`options` (e.g. `"Sales Invoice Item"`), and its folder is
`sales_invoice_item/`.

### Doctype-level keys

| Key | Meaning |
| --- | --- |
| `name` | Human-readable doctype name (e.g. `"Sales Invoice"`). Referenced by `Link`/`Table` `options`. |
| `module` | Organizational module label (e.g. `"Selling"`, `"Stock"`). |
| `istable` | `true` for a child table doctype, else omit/`false`. |
| `is_submittable` | `true` to enable the draft → submit → cancel lifecycle + GL/stock posting. **See gotcha below.** |
| `autoname` | Naming pattern, e.g. `"SI-.YYYY.-.####"` → `SI-2026-0001`. |
| `title_field` | Field shown as the record title in lists/forms. |
| `controller_class` | Class name of the controller. Honored for custom apps; for core, the system seed derives it as `name` with spaces removed. Set it to match your class to be safe. |
| `fields` | Array of field defs (below). |

See [reference/field-types.md](reference/field-types.md) for the full
list of valid `fieldtype` values and every field-level key.

## Step 3 — Write the controller (optional)

Only needed if the doctype has logic: computed fields, validations, GL
posting, or stock movement. File: `<snake_name>.py`.

```python
from app.doctype_engine.controller import BaseController, register_controller


@register_controller("Sales Invoice")  # must match the JSON `name` exactly
class SalesInvoiceController(BaseController):

    async def before_save(self) -> None:
        """Compute derived fields / line + header totals. self.doc.data is the dict."""
        data = self.doc.data
        items = [i for i in data.get("items", []) if isinstance(i, dict)]
        for r in items:
            r["amount"] = float(Decimal(str(r.get("qty") or 0)) * Decimal(str(r.get("rate") or 0)))
        data["total_amount"] = float(sum(Decimal(str(r.get("amount", 0))) for r in items))

    async def before_submit(self) -> None:
        """Strict validations that drafts may bypass. Raise ValidationError to block."""

    def get_gl_entries(self) -> list[dict]:
        """Double-entry rows. Each: account_id, debit OR credit (one is 0), narration, optional party_id/party_type."""
        return []

    def get_stock_entries(self) -> list[dict]:
        """Stock specs: item_id, warehouse_id, qty (signed Decimal: + receipt / - issue), rate, optional voucher_detail_no."""
        return []
```

### Controller lifecycle hooks (all `async`, override what you need)

| Hook | When | Typical use |
| --- | --- | --- |
| `before_save` | before insert/update + GL/stock posting | compute totals, defaults, derived fields |
| `after_save` | after save + GL/stock complete | sync document_links, notifications |
| `before_submit` | before a draft is submitted | enforce strict validations |
| `before_cancel` | before a submitted doc is cancelled | block cancel if downstream state requires |
| `after_cancel` | after cancel + GL/stock reversed (same txn) | clean up links; raising rolls back |
| `before_amend` | before a cancelled doc is copied to a new draft | clean amendment-specific state |
| `before_delete` | before GL removal + soft delete | block delete if linked docs exist |
| `get_gl_entries` | on submit/repost | return double-entry GL rows |
| `get_stock_entries` | on submit/repost | return signed stock movements |

`BaseController` exposes: `self.doc` (the Document; `self.doc.data` is the
field dict, `self.doc.name`, `self.doc.id`), `self.meta`, `self.db`
(active tenant session), and an async `self.tenant_db()` context manager
yielding `(db, company)` for reads against the active company DB.

## Step 4 — Verify

```bash
# JSON is valid + parseable (path is the app folder, bundled or external)
python3 -c "import json; json.load(open('<app>/doctypes/<name>/<name>.json')); print('ok')"
# Controller registers (run inside the backend; the app must be discoverable)
python3 -c "from app.doctype_engine.controller import _controller_classes; print('<Doctype Name>' in _controller_classes)"
```

Then `python -m app.migrate` (or click **Reload** in Custom Apps) — the
startup/reload log shows discovery + `Doctype registry loaded` with no
tracebacks. The DocType becomes a **global definition immediately**, visible
to every company. It does NOT depend on a per-company install — install only
**activates** the app (and seeds per-company print formats/reports/fixtures).

You do **not** write an Alembic migration to add a doctype or field.

## Gotchas (verified, not guesses)

- **Definitions are GLOBAL, reconciled from code.** `_upsert_global_doctypes`
  (in `platform_app_seed.py`) creates/updates the single global DocType row
  (`company_id NULL`) on every migrate/reload, honoring `is_submittable`,
  `issingle`, `track_changes`, `autoname`, `title_field`, etc. Put the full
  truth in the JSON — it's authoritative and reconciled on each run.
- **Field additions are additive only.** New fields are added and drifted
  `Select` options reconciled, but fields are not deleted/renamed
  automatically. Renames need a deliberate migration.
- **`Link`/`Table` `options` must match an existing doctype `name`
  exactly** (spaces and case included), or validation breaks.
- **Layout-break fieldtypes:** the FieldType enum values have spaces
  (`"Section Break"`, `"Column Break"`). The loader also accepts the no-space
  member names (`"SectionBreak"`) and normalises them, but prefer the spaced
  form in new JSON.
- **Single shared DB.** All companies live in one database (separated by
  `company_id`); there are no per-tenant databases to keep in sync. One
  `alembic upgrade head` migrates everything.

## Key files (read only if you need to go deeper)

| File | Role |
| --- | --- |
| `backend/app/config.py` | `external_apps_dir` setting (default `/app/external_apps`) |
| `backend/app/services/app_manifest.py` | dual-source `discover_apps()` (bundled + external), `_load_doctypes/reports/print_formats/fixtures`, `AppSpec.origin` |
| `backend/app/services/doctype_seed.py` | `seed_system_doctypes()`, `seed_all_tenant_doctypes()` (bundled/global) |
| `backend/app/services/platform_app_seed.py` | registers apps in the catalog; `distribution` = `global` (bundled) vs `managed` (external) |
| `backend/app/doctype_engine/controller.py` | `BaseController`, `register_controller`, `_discover_controllers` (loads from both roots) |
| `backend/app/models/doctype.py` | `Doctype`, `DoctypeField`, `FieldType` enum |
| `backend/app/custom_app/installer.py` | materialises a managed app into one company's tenant DB (doctypes/fields/reports/print_formats/fixtures; honors `is_submittable`) |
| `backend/app/models/custom_app.py` | `CustomApp` (incl. `distribution`), versions, installs |
