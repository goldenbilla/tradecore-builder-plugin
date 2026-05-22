---
name: tradecore-print-format
description: |
  How to build a Print Format in TradeCore. Use whenever the user asks to
  create, add, or modify a print format / print template / printable
  document layout (invoice, quotation, delivery note, voucher, etc.) — for
  a core doctype or inside a custom app. Encodes the PrintFormat data
  model, the two authoring modes (block-layout JSON vs raw Jinja+CSS
  template_html), the on-disk `print_formats/<name>.json` shape that ships
  with a code-backed custom app, and how it installs — so a print format
  can be added without re-reading the codebase.
---

# Adding a Print Format in TradeCore

A **print format** is a printable layout bound to one doctype. It renders a
document to HTML/PDF. Pick where it lives and which authoring mode first.

## Where does it live?

| | Custom-app print format | Company print format |
| --- | --- | --- |
| **When** | Ships with a code-backed app, installs onto every company that installs the app | A one-off format for a single company |
| **Defined in** | `print_formats/<name>.json` in the app's repo folder | A `print_formats` DB row (created via UI/API) |
| **Travels via** | Custom-app install (the installer materialises it into the company's tenant DB) | Created directly in the tenant DB |
| **You build it** | Yes — author the JSON file | Usually the user, in the Print Designer UI |

This skill covers the **custom-app** case (the file you author). For a
one-off company format, point the user at the Print Designer UI, or POST a
`print_formats` row.

> Custom apps live in their own repo under `EXTERNAL_APPS_DIR`
> (`/app/external_apps/<namespace>/`), each with a `print_formats/` folder.
> Drop a `<name>.json` there; on **app install** the
> `app/custom_app/installer.py` writes a `PrintFormat` row (tenant-scoped,
> stamped with `app_id`) so uninstall is a clean reversal.

---

## The PrintFormat model (what a format becomes)

`backend/app/models/print_format.py`:

| Column | Meaning |
| --- | --- |
| `name` | format name (unique per `company_id` + `doctype_name`) |
| `doctype_name` | the doctype it prints (must exist) |
| `is_default` | the default format for that doctype |
| `template_html` | raw Jinja+CSS template (legacy / full-control mode) |
| `layout` | block-layout JSON (the Print Designer mode — preferred) |
| `margin_top/right/bottom/left` | page margins (mm) |
| `app_id` / `app_version` | provenance — set when installed by a custom app |

When `layout` is set, the block-stack renderer takes precedence; otherwise
`template_html` is rendered as a Jinja template.

---

## The file you author: `print_formats/<name>.json`

The custom-app manifest validator (`app/custom_app/manifest.py`) accepts
this shape. **Prefer the `layout` mode** — it's the visual-designer format
and survives schema changes better than hand-written HTML.

```json
{
  "name": "Biomass Voucher",
  "doctype": "Dropshipping Entry",
  "is_default": true,
  "margins": { "top": 14, "right": 12, "bottom": 14, "left": 12 },
  "layout": {
    "version": 2,
    "paper":   { "size": "A4", "orientation": "portrait" },
    "margins": { "top": 14, "right": 12, "bottom": 14, "left": 12 },
    "blocks": [ ...see reference/layout-blocks.md... ]
  }
}
```

Required keys: `name`, `doctype`. Provide **either** `layout` (preferred)
**or** `template_html` (raw mode). `is_default` and `margins` are optional.
`doctype` must match a doctype the app defines or an existing core doctype.

The full block catalog (header, title, bill_to/ship_to, items_table,
totals, terms, notes) and the field-reference syntax (`doc.name`,
`doc.data.<field>`, `item.<field>`, `loop.index`, formats `date`/`currency`)
are in **reference/layout-blocks.md** — read it before composing a layout.

---

## Raw `template_html` mode (full control)

Use only when the block layout can't express the design. It's a Jinja
template with inline CSS; `doc` is the document, `doc.data` its field
values, `company` the issuing company. Mirror an existing template in
`backend/app/services/print_format_seed.py` (e.g. the quotation template)
for the print-safe CSS conventions (pt units, `border-collapse`, zebra
rows, `.right` numeric columns).

---

## Recipe

1. **Confirm the target doctype exists** and note its field names
   (the layout/template references `doc.data.<fieldname>` and, for child
   tables, `item.<fieldname>`). For a custom app, the doctype is in the
   same app folder under `doctypes/`.
2. **Choose the mode** — `layout` (default) or `template_html`.
3. **Write** `print_formats/<name>.json` in the app folder. For `layout`,
   compose blocks per **reference/layout-blocks.md**, starting from the
   quotation/invoice default layout in `print_format_seed.py` as a template.
4. **Set `is_default: true`** only if it should be the doctype's default
   (at most one default per doctype per company).
5. **Bump the app's `version`** in `app.json` if you want a new installable
   version (installs/upgrades pick up the format).
6. **Install / re-install** the app onto a company to materialise the
   `PrintFormat` row, then verify: open a document of that doctype →
   Print → the format is listed and renders.

## Gotchas

- The format is **not visible until the app is installed** for that company
  (it's tenant-scoped). Re-install or roll the version forward to pick up
  changes.
- The custom-app **builder UI does not yet show a Print Formats tab** —
  print formats are authored as files / installed, not via that screen.
- `name` is unique per `(company, doctype)`. Re-using a name on the same
  doctype is a conflict the installer rejects.
- Empty layout sections auto-skip (a doc with no `terms` simply omits the
  terms block) — don't guard them manually.
