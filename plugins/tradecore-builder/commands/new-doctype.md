---
description: Scaffold a new TradeCore DocType (JSON + optional controller) in the custom-app architecture
argument-hint: <Doctype Name> [in <app>] [submittable] [fields...]
---

Create a new TradeCore DocType from this request: **$ARGUMENTS**

Follow the `tradecore-doctype` skill exactly. Steps:

1. Invoke the **tradecore-doctype** skill for the authoritative recipe,
   field-type reference, and gotchas. Do not re-derive conventions from
   the codebase.
2. Determine the target app (default `core` unless the user named a
   custom app, or the doctype is clearly an installable feature). If a
   new custom app is needed, scaffold it with
   `python -m app.scripts.new_app`.
3. Confirm the doctype `name`, whether it is submittable, whether it
   needs a child table, and the field list with the user **only if these
   are ambiguous** — otherwise infer sensible fields and proceed.
4. Create `backend/app/apps/<app>/doctypes/<snake_name>/<snake_name>.json`
   with correct field types, `options` for Links/Tables, `sort_order`,
   and list/filter flags. Create child-table doctype folders as needed.
5. If the doctype has logic (totals, validations, GL, stock), create
   `<snake_name>.py` with an `@register_controller("<Name>")` controller
   implementing only the needed hooks.
6. Verify: validate the JSON parses, confirm the controller registers,
   and (if a backend is running) confirm it seeds with no tracebacks.
7. Summarize what you created and any submittable/seed caveats from the
   skill's gotchas section.
