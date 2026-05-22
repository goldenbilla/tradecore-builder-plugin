# TradeCore field reference

Source of truth: `backend/app/models/doctype.py` (`FieldType` enum,
`DoctypeField` columns). Read this when authoring a doctype JSON.

## Valid `fieldtype` values

### Data field types (store a value)

| fieldtype | Use for | Notes |
| --- | --- | --- |
| `Data` | short text, codes, IDs | plain string |
| `Int` | counts, days | integer |
| `Float` | weight, percentage, qty | decimal; set `precision` |
| `Currency` | money | decimal, 2 places, money-formatted |
| `Date` | dates | `YYYY-MM-DD` |
| `Datetime` | timestamps | ISO with timezone |
| `Select` | dropdown | options in `options`, newline-separated: `"A\nB\nC"` |
| `Link` | reference another doctype | **must** set `options` to the target doctype `name` |
| `Table` | child table | **must** set `options` to the child doctype `name` |
| `Check` | boolean | renders as checkbox |
| `Text` | multi-line text | |
| `SmallText` | shorter multi-line text | |
| `Attach` | file upload | |
| `Password` | masked input | |
| `Rating` | star rating | |
| `Signature` | signature capture | |
| `Barcode` | barcode | |
| `HTML` | rendered HTML block | |

### Layout field types (no stored value)

| fieldtype | Purpose |
| --- | --- |
| `Section Break` | start a new section (give it a `label` for a header) |
| `Column Break` | split the current section into columns |
| `Tab Break` | start a new tab |

> Note: enum member names like `SectionBreak` map to the string values
> `"Section Break"` etc. In JSON always use the **string with the space**:
> `"fieldtype": "Section Break"`.

## Field-level keys

```jsonc
{
  "fieldname": "customer_id",      // snake_case identifier (required)
  "label": "Customer",             // UI label (required)
  "fieldtype": "Link",             // one of the above (required)
  "options": "Customer",           // Link target / child doctype / Select options
  "sort_order": 1,                 // display order; floats allowed (1, 1.5, 2)

  // requirement / editability
  "reqd": true,                    // required
  "read_only": true,               // computed; not user-editable
  "hidden": false,                 // keep in data, hide from UI
  "allow_on_submit": false,        // editable after submit
  "is_unique": false,              // enforce uniqueness
  "non_negative": false,           // reject negatives (numeric)
  "precision": 2,                  // decimal places (Float/Currency)

  // list & filtering
  "in_list_view": true,            // show as a column in the list/table
  "in_standard_filter": true,      // expose as a list filter

  // defaults & auto-population
  "default_value": "Exclusive",    // default
  "fetch_from": "party_id.name",   // auto-fill from a linked doc's field
  "fetch_if_empty": false,         // only fetch when blank

  // conditional behavior (expression strings, ERPNext-style)
  "depends_on": "...",             // show/hide condition
  "mandatory_depends_on": "...",   // conditional required
  "read_only_depends_on": "...",   // conditional read-only
  "collapsible": false,
  "collapsible_depends_on": "...",

  // meta
  "description": "help text",
  "bold": false,
  "no_copy": false,                // don't copy when amending/duplicating
  "print_hide": false,
  "search_index": false
}
```

The seeder fills boolean defaults (`reqd`, `in_list_view`,
`in_standard_filter`, `read_only`, `hidden`, `allow_on_submit`) to
`false` when omitted, so you only specify the ones you need.

## Common field snippets

```jsonc
// Required link, shown + filterable in list
{"fieldname": "customer_id", "label": "Customer", "fieldtype": "Link", "options": "Customer", "reqd": true, "in_list_view": true, "in_standard_filter": true}

// Computed currency, read-only
{"fieldname": "total_amount", "label": "Total Amount", "fieldtype": "Currency", "read_only": true, "in_list_view": true}

// Select with default
{"fieldname": "tax_type", "label": "Tax Type", "fieldtype": "Select", "options": "Exclusive\nInclusive", "default_value": "Exclusive"}

// Child table
{"fieldname": "items", "label": "Items", "fieldtype": "Table", "options": "Sales Invoice Item"}

// Auto-fill from a linked record
{"fieldname": "customer_name", "label": "Customer Name", "fieldtype": "Data", "read_only": true, "fetch_from": "customer_id.name"}

// Section header
{"fieldname": "sec_totals", "label": "Totals", "fieldtype": "Section Break"}
```
