# Print Format layout — block reference

The `layout` object is interpreted by the block-stack renderer. Shape:

```json
{
  "version": 2,
  "paper":   { "size": "A4", "orientation": "portrait" },
  "margins": { "top": 14, "right": 12, "bottom": 14, "left": 12 },
  "blocks":  [ /* ordered list of blocks, top to bottom */ ]
}
```

Every block has an `id` (unique string) and a `type`. Blocks render in
order. Blocks whose source field is empty auto-skip.

## Field reference syntax

Used in block `field` / `meta[].field` / column `field`:

| Reference | Resolves to |
| --- | --- |
| `doc.name` | the document's name/number |
| `doc.data.<fieldname>` | a header field value on the document |
| `item.<fieldname>` | a child-row field (inside `items_table` columns) |
| `loop.index` | 1-based row number (inside `items_table` columns) |

Optional `format` on a field/column/meta entry:
- `"date"` — format as a date
- `"currency"` — format as money (right-aligned, monospace)
- omitted — raw value

Optional `fallback` on a meta entry — shown when the field is empty.

## Block catalog

### `header` — company branding
```json
{ "id": "header-1", "type": "header", "show_logo": true, "show_letter_head": false }
```

### `title` — document title badge + meta key/value table
```json
{
  "id": "title-1", "type": "title", "title": "Quotation", "align": "right",
  "meta": [
    { "label": "No.",        "field": "doc.name" },
    { "label": "Date",       "field": "doc.data.posting_date", "format": "date" },
    { "label": "Valid Till", "field": "doc.data.valid_till", "format": "date", "fallback": "—" },
    { "label": "Status",     "field": "doc.data.status", "fallback": "Draft" }
  ]
}
```

### `bill_to` / `ship_to` — party block
```json
{ "id": "billto-1", "type": "bill_to", "label": "Bill To", "name_field": "customer_name" }
```

### `items_table` — the child-row table
```json
{
  "id": "items-1", "type": "items_table",
  "field": "doc.data.items",
  "zebra": true,
  "header_background": "#1e3a5f", "header_color": "#ffffff",
  "columns": [
    { "field": "loop.index",  "header": "#",            "width": 10, "align": "center" },
    { "field": "item.name",   "header": "Description",  "width": 90, "align": "left" },
    { "field": "item.qty",    "header": "Qty",          "width": 20, "align": "right" },
    { "field": "item.rate",   "header": "Rate (PKR)",   "width": 30, "align": "right", "format": "currency" },
    { "field": "item.amount", "header": "Amount (PKR)", "width": 30, "align": "right", "format": "currency" }
  ]
}
```
- `field` points at the child-table fieldname (`doc.data.<table_field>`).
- `width` values are relative; keep them summing to ~100.
- `align`: `left` | `center` | `right`.

### `totals` — totals/grand-total block
```json
{ "id": "totals-1", "type": "totals", "currency": "PKR" }
```

### `terms` / `notes` — rich-text sections (auto-skip if empty)
```json
{ "id": "terms-1", "type": "terms", "label": "Terms & Conditions", "field": "doc.data.terms_content", "render_html": true }
{ "id": "notes-1", "type": "notes", "label": "Notes", "field": "doc.data.narration", "render_html": true }
```
- `render_html: true` renders stored rich-text (tiptap) HTML as-is.

## Starting point

Copy a default layout from
`backend/app/services/print_format_seed.py` (e.g. `DEFAULT_QUOTATION_LAYOUT`)
and adapt the title, party block, and `items_table` columns to the target
doctype's fields. That's the fastest correct path.
