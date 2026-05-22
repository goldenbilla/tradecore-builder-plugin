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

TradeCore is a multi-tenant, ERPNext-style platform. DocTypes are
**code-authored as files** in an app's `doctypes/` folder and
auto-discovered at backend startup. You almost never touch a database or
write a migration to add one.

## Two kinds of app — pick first

| | **Bundled (system) app** | **External (custom) app** |
| --- | --- | --- |
| Lives in | the platform image: `backend/app/apps/<app>/` (e.g. `core`) | its **own git repo**, mounted at `EXTERNAL_APPS_DIR/<namespace>/` (default `/app/external_apps/<namespace>/`, e.g. `biomass`) |
| `app.json` `type` | `"system"` | `"custom"` |
| Distribution | **global** — doctypes seeded for every tenant, auto-installed everywhere | **managed** — registered in the catalog, **installed per company on demand** (no global rows, no auto-install) |
| Doctype rows | global (`company_id IS NULL`) | company-scoped, created by the installer when a company installs the app |

New business features should be **external custom apps** (their own repo,
opt-in) — that's the current model. Only truly universal objects go in
bundled `core`.

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
- **How rows get created:** bundled/system doctypes are seeded global on
  boot; external/managed doctypes are materialised into a company's tenant
  DB by `app/custom_app/installer.py` when an admin **installs** the app
  for that company (controller stays bound).

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

Restart the backend — startup log should show discovery + `Doctype
registry loaded` with no tracebacks.

- **Bundled/system doctype:** seeding is additive + idempotent and
  `seed_all_tenant_doctypes()` runs against every tenant DB, so existing
  tenants pick it up automatically.
- **External/managed doctype:** discovery only *registers* the app; the
  doctype rows are created when you **install** the app for a company
  (Custom Apps → Install, or `POST /custom-apps/{id}/install`). It won't
  appear for a company until installed.

You do **not** write an Alembic migration to add a doctype or field.

## Gotchas (verified, not guesses)

- **`is_submittable` is NOT read by the bundled core/system seed.**
  `app/services/doctype_seed.py` only copies `istable`, `autoname`,
  `title_field`, and a derived `controller_class` — it ignores
  `is_submittable`. The **custom-app installer** (`app/custom_app/installer.py`)
  and the **platform-app seed** *do* honor it. Since external/custom apps
  install via that path, a submittable doctype in an external app works
  correctly — which is another reason new submittable objects belong in a
  custom app, not bundled `core`. Always put `is_submittable: true` in the
  JSON regardless — it's the source of truth.
- **Field additions are additive only.** The seeder adds new fields and
  reconciles drifted `Select` options, but does not delete or rename
  fields on existing tenants. Renames need a deliberate migration.
- **`Link`/`Table` `options` must match an existing doctype `name`
  exactly** (spaces and case included), or seeding/validation breaks.
- **Multi-tenant DB drift** is handled by `seed_all_tenant_doctypes()` at
  startup — but if you ever seed manually, remember `alembic upgrade head`
  only touches the control DB; each tenant DB is seeded separately.

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
