---
description: Add a new TradeCore report (hand-coded endpoint + React component, or Report Builder definition)
argument-hint: <Report Name> [for <Doctype>] [filters...]
---

Add a new TradeCore report from this request: **$ARGUMENTS**

Follow the `tradecore-report` skill exactly. Steps:

1. Invoke the **tradecore-report** skill for the authoritative recipe and
   the two report kinds. Do not re-derive conventions from the codebase.
2. Decide the kind:
   - **Hand-coded** (bespoke logic, joins, aggregation, a dedicated
     screen) → build the backend endpoint + API method + React component
     + reports-page wiring + sidebar link.
   - **Report Builder** (user-defined columns/filters on one doctype) →
     this already exists; either point the user to `/reports` or create
     the definition via the report-builder API.
   Ask the user which only if it is genuinely ambiguous.
3. For a hand-coded report, implement all five wiring steps from the
   skill, and apply the **tradecore-ui** skill for the component's layout.
4. Verify with the preview workflow: the endpoint returns
   `success: true`, the component renders, and filters re-query. Fix and
   re-check before reporting done.
5. Summarize the files you added/changed and the route it lives at.
