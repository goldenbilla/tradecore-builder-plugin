# tradecore-builder plugin

A local Claude Code plugin that teaches Claude how to create **DocTypes**,
**Reports**, and **Print Formats** in the TradeCore custom-app architecture,
so it doesn't have to re-read the codebase each time.

## What's inside

- **Skill `tradecore-doctype`** — the full recipe for authoring a doctype
  under a custom app's `doctypes/`: JSON schema, field types, controller
  hooks, seeding/discovery, and multi-tenant gotchas.
- **Skill `tradecore-report`** — how to add a report, both hand-coded
  (endpoint + React component) and Report Builder (metadata) kinds.
- **Skill `tradecore-print-format`** — how to author a print format
  (`print_formats/<name>.json`) for a doctype: block-layout JSON vs raw
  Jinja+CSS, the PrintFormat model, and how it installs with a custom app.
- **Command `/new-doctype`** — scaffold a doctype from a one-line request.
- **Command `/new-report`** — add a report from a one-line request.
- **Command `/new-print-format`** — add a print format from a one-line request.

The skills auto-trigger when you ask Claude to build/modify a doctype,
report, or print format. The commands are explicit entry points.

## Install (one-time)

This repo's root is a Claude Code marketplace (`.claude-plugin/marketplace.json`),
so it installs straight from GitHub:

```
/plugin marketplace add goldenbilla/tradecore-builder-plugin
/plugin install tradecore-builder@tradecore
```

(Or from a local clone: `/plugin marketplace add /path/to/tradecore-builder-plugin`.)

Then restart or reload. Verify with `/plugin` (should list
`tradecore-builder` as enabled) and `/help` (should show `/new-doctype`,
`/new-report`, and `/new-print-format`).

## Updating

After pushing changes here, run:

```
/plugin marketplace update tradecore
```

## Layout

```
tradecore-builder-plugin/
├── .claude-plugin/
│   └── marketplace.json          # marketplace "tradecore"
└── plugins/
    └── tradecore-builder/
        ├── .claude-plugin/plugin.json
        ├── skills/
        │   ├── tradecore-doctype/SKILL.md
        │   │   └── reference/field-types.md
        │   ├── tradecore-report/SKILL.md
        │   └── tradecore-print-format/SKILL.md
        │       └── reference/layout-blocks.md
        └── commands/
            ├── new-doctype.md
            ├── new-report.md
            └── new-print-format.md
```
