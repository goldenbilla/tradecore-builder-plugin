---
description: Add a new TradeCore print format (block-layout JSON or raw Jinja+CSS) for a doctype, shipped in a custom app
argument-hint: <Format Name> for <Doctype> [default] [in <app>]
---

Add a new TradeCore print format from this request: **$ARGUMENTS**

Follow the `tradecore-print-format` skill exactly. Steps:

1. Invoke the **tradecore-print-format** skill for the authoritative recipe,
   the PrintFormat model, and the two authoring modes. Do not re-derive
   conventions from the codebase.
2. Confirm the **target doctype** exists and gather its field names
   (header fields and, for the items table, child-row fields). For a custom
   app, the doctype is in the same app folder under `doctypes/`.
3. Choose the mode — **`layout`** (block JSON, the default) or
   **`template_html`** (raw Jinja+CSS, only when layout can't express it).
   Read `reference/layout-blocks.md` before composing a layout.
4. Write `print_formats/<name>.json` in the app's repo folder
   (`/app/external_apps/<namespace>/print_formats/`), starting from a
   default layout in `print_format_seed.py` and adapting title, party
   block, and items-table columns. Set `is_default: true` only if asked.
5. If a new installable version is wanted, bump `version` in the app's
   `app.json`.
6. Verify: (re)install the app onto a company, open a document of that
   doctype → **Print**, confirm the format is listed and renders correctly;
   fix and re-check before reporting done.
7. Summarize the file you added and which doctype/app it belongs to, and
   note that the format only appears once the app is installed for a company.
