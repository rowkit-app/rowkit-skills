---
name: rowkit-tables
description: Create tables, edit schema, and read/write records in a Rowkit workspace using its scoped agent API key — including derived formula fields that compute themselves from other columns (totals, flags, cross-table lookups). Use whenever the user wants an AI agent to maintain structured data in their Rowkit workspace — a CRM, invoice log, monitoring feed, inventory tracker, waitlist, or any other table-shaped tracking task — instead of doing it by hand in the UI.
---

# Rowkit — agent table access

This skill lets an agent create and maintain tables of data in a Rowkit
workspace on the user's behalf: define schema, then create/read/update/delete
records. It talks to one app only — the app the user's API key was minted for.

## Before you do anything

You need one thing from the user, once:

- **`ROWKIT_API_KEY`** — a key starting with `gb_live_`, created by the app
  owner in Rowkit under **Settings > API keys > New API key**. It is shown to
  them exactly once at creation time.

`ROWKIT_API_BASE` defaults to `https://api.rowkit.app` — Rowkit's hosted API.
Only ask for a different base URL if the user tells you they're running a
self-hosted or local instance (e.g. `http://localhost:8080`).

If you don't have the key, ask the user for it before calling anything. Do
not invent one.

## Handling the key securely

- Treat the key exactly like a password. **Never** print it in full back to the
  user, log it, write it into a file you create, or include it in any output
  that might be shared, committed, or displayed. It's fine to reference it as
  `$ROWKIT_API_KEY` / an env var, or by its last 4 characters.
- Send it only as `Authorization: Bearer <key>` over HTTPS, only to
  `ROWKIT_API_BASE`. Never send it anywhere else.
- The key is already scoped: it can only touch the one app it was minted for,
  and it can only do schema + record operations (see "What this key cannot do"
  below) — you don't need to add your own extra restrictions on top.
- If a request ever returns `401 Unauthorized`, the key is invalid, expired, or
  was revoked. Don't retry with a guessed variant of it — tell the user and ask
  them to check **Settings > API keys** (they may need to mint a new one).
- If you ever suspect the key leaked (e.g. it appeared somewhere it shouldn't
  have), tell the user immediately and recommend they revoke it in
  **Settings > API keys** and create a replacement.

## Base request shape

Every call below is relative to `ROWKIT_API_BASE`, with:

```
Authorization: Bearer <ROWKIT_API_KEY>
Content-Type: application/json   (on POST/PATCH)
```

There is no `{appID}` in any path — the key already fixes which app you're
working in.

## Routes

### Tables (schema)

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/v0/agent/tables` | List all active tables in this app. |
| `POST` | `/v0/agent/tables` | Create a new table with an initial set of fields. |
| `GET` | `/v0/agent/tables/{tableID}` | Get one table's full field list. |
| `POST` | `/v0/agent/tables/{tableID}/fields` | Add a new field to an existing table. |
| `PATCH` | `/v0/agent/tables/{tableID}/fields/{fieldID}` | Rename/reconfigure an existing field. |

**Create a table** — `POST /v0/agent/tables`:

```json
{
  "name": "Leads",
  "key": "leads",
  "fields": [
    { "name": "Full Name", "key": "full_name", "kind": "text", "required": true },
    { "name": "Email", "key": "email", "kind": "email" },
    { "name": "Stage", "key": "stage", "kind": "single_select", "options": ["new", "contacted", "qualified", "won", "lost"] }
  ]
}
```

`key` must be lowercase `snake_case`. Response: `{ "table": {...}, "defaultGrid": {...} }` —
`table.ID` is what you use as `{tableID}` in every route below. `table.Fields[].ID`
is what you use as `{fieldID}`. `GET /v0/agent/tables/{tableID}` returns that
same `{ "table": {...}, "defaultGrid": {...} }` shape.

**Field `kind` values**: `text`, `long_text`, `email`, `phone`, `number`,
`currency`, `checkbox`, `date`, `datetime`, `single_select` (needs `options`),
`link_record` (needs `targetTableId`, the `ID` of the table it points to —
that table must already exist, so create referenced tables first),
`json`, `formula` (needs `formula` and usually `resultKind` — see "Formula
fields" below).

**Add a field** — `POST /v0/agent/tables/{tableID}/fields` — same shape as one
entry in `fields` above.

**Update a field** — `PATCH /v0/agent/tables/{tableID}/fields/{fieldID}`:

```json
{ "name": "New Label", "key": "new_label", "required": true, "options": ["a", "b", "c"] }
```

Only send the keys you want to change. Supported patch keys are `name`, `key`,
`required`, `options`, `targetTableId`, and `formula`. `options` only applies
to `single_select` fields, `targetTableId` only to `link_record` fields, and
`formula` only to formula fields (see below). Renaming a field (`key`) that an
active formula reads is rejected rather than silently breaking the formula;
create a replacement stored field, update the formula to use it, backfill, and
leave the old field in place if that migration is needed.

There is no route to delete a table. If a table is no longer needed, leave it
(or ask the user to remove it from the product UI) rather than trying to work
around this.

### Formula fields

A `formula` field's value is **computed automatically from other fields on
write** — you never set it in `values` when creating/updating a record (and
if you try, the API rejects it with `FIELD_READ_ONLY`). Use it whenever the
user wants a value that's *derived* from other columns instead of entered by
hand: totals, discounts, flags, labels built from other fields, status text
that depends on a threshold, and so on.

**Creating one** — same shape as any other field in `fields`/`POST .../fields`,
plus two extra keys:

```json
{
  "name": "Total After Discount",
  "key": "total_after_discount",
  "kind": "formula",
  "formula": "round(subtotal * (1 - coalesce(discount_pct, 0) / 100), 2)",
  "resultKind": "currency"
}
```

- `formula` — the expression source, as a plain string (see grammar below).
- `resultKind` — which formula-compatible output kind the computed value
  materializes as (`text`, `long_text`, `email`, `phone`, `number`,
  `currency`, `checkbox`, `date`, `datetime`, `single_select`). `formula`,
  `link_record`, and `json` are not valid formula outputs: a formula can't
  hold a reference and the expression language has no JSON-producing value.
  Defaults to `text`
  if omitted, but always set it explicitly to match what the expression
  actually produces — a numeric expression with `resultKind: "text"` will
  still get stringified, which is rarely what you want. **`number` stores
  integers only** — a result of `9/4` (2.25) fails the write with
  `non-integer result`. If the expression can produce a fraction (any
  division, weighted averages, percentages), use `currency` instead; it keeps
  decimals.
- `required` is always ignored/forced off for formula fields — a derived
  value can't be "required," there's nothing for the user to fill in.

**Editing the formula on an existing field** — `PATCH .../fields/{fieldID}`
with just the new source:

```json
{ "formula": "round(subtotal * (1 - coalesce(discount_pct, 0) / 100), 2)" }
```

`resultKind` (the underlying stored type) cannot change after creation — if
the result type needs to change, create a new formula field instead.

**Formula grammar** — a small expression language, not a general scripting
language:

- **Literals**: numbers (`42`, `3.5`), strings (`'text'` or `"text"`),
  `true`, `false`, `null`.
- **References**: a bare field key (`price`) reads that field on the *same
  record*. A dotted path (`customer.name`) follows **exactly one** hop
  through a `link_record` field to read a field on the record it points at —
  no deeper chains (`customer.company.name` is not allowed).
- **Arithmetic**: `+ - * /` on numbers. `+` concatenates whenever either
  operand is text (`'qty: ' + qty` produces text), so it doubles as a
  two-argument `concat`. Division by zero at write time fails the write with
  a `400` — guard it (see `if`/`coalesce` below) unless you're certain the
  divisor can never be zero.
- **Comparison**: `== != > >= < <=`, producing a boolean. Numbers compare
  numerically, booleans and same-kind text-ish values compare directly; dates
  are stored as ISO `YYYY-MM-DD` strings, so `<`/`>` on them orders correctly.
  `==`/`!=` handle `null` safely (`x == null` works), but ordering comparisons
  with a `null` side produce `null`, not a boolean — wrap in `coalesce` or an
  explicit `== null` guard if you need a definite true/false.
- **Boolean logic**: `and`, `or`, `not` (as words, not `&&`/`||`/`!`).
- **Parentheses** for grouping: `(a + b) * c`.
- **Functions**: `if(cond, thenExpr, elseExpr)`, `concat(a, b, ...)`,
  `lower(text)`, `upper(text)`, `trim(text)`, `length(text)`, `abs(number)`,
  `round(number)` / `round(number, decimals)`, `coalesce(a, b, ...)` (first
  non-null argument), `today()`, `now()` (evaluated and snapshotted at write
  time — not a live clock).

**How the value actually gets set** — a formula field is not evaluated on
read. It's compiled once (when you create/update the field — a bad
expression, a typo'd field name, a reference deeper than one hop, or a
dependency cycle between two formula fields all fail right there with
`400`), then evaluated and written into the record like any other stored
value every time that record is created or updated. Two consequences worth
knowing:

- Adding a formula field to a table with existing records does **not**
  retroactively backfill them — a row only picks up the new field's value on
  its next write. If the user wants existing rows to show a value
  immediately, `PATCH` each one with `{"values": {}}` (an empty patch still
  triggers recomputation) or touch any field on it.
- A formula field can read another formula field (formulas chain), and a
  formula that looks up through a `link_record` field automatically
  recomputes on the *referencing* records whenever the record it points at
  changes — e.g. renaming a customer updates `customer.name` lookups on all
  of that customer's orders, with no extra work from you.
- Because computed values live in real stored columns, you can **sort and
  filter queries by formula fields** exactly like user-entered ones
  (`sortFieldId`/`filterFieldId` accept their field IDs).

**Formula gotchas** — the things that most often bite agents:

- A formula field **can go in a table's initial creation payload** —
  dependencies are checked against the whole payload at once, so
  `{"fields": [..., {"kind": "formula", "formula": "price * qty"}]}` works
  even though `price`/`qty` are defined in the same request. Order inside
  the array doesn't matter.
- When adding a formula to an **existing** table via `POST .../fields`,
  every field it references must already exist — you get `400` otherwise.
  Create the dependencies first, the formula last.
- **No retroactive backfill.** Adding a formula to a table that already has
  rows leaves those rows empty (`null`) until each row's next write. To force
  it, `PATCH` each record with `{"values": {}}` — an empty patch still
  triggers recomputation.
- **`number` is integer-only.** A formula stored as `resultKind: "number"`
  that yields a non-integer (e.g. `a / b` where `b` doesn't divide `a` evenly)
  fails the write with `non-integer result`. For anything that can be
  fractional, use `resultKind: "currency"` — it preserves decimals.
- **Division by zero fails the whole write** (`400`). Guard any formula that
  divides by a user-editable number with `if(... == 0, null, ...)` even if
  today's data looks safe — tomorrow's row might not be. The guard genuinely
  short-circuits: the untaken branch is never evaluated, so
  `if(qty == 0, null, price / qty)` is safe.
- **`today()`/`now()` are write-time snapshots**, not a live clock. A
  `due_date < today()` flag only flips when the row is next written — if
  time-driven freshness matters, touch the rows (empty `PATCH`) on a schedule
  and tell the user it's snapshot-based.
- **`resultKind` is locked after creation** and the expression's inferred
  type is checked against it at create time — an error like
  `formula result for <key> is <kind>, incompatible with field type ...`
  means fix `resultKind`, not the math. To change the result type later,
  create a new formula field.
- **`if` treats a `null` condition as false** (takes the else branch), and
  **`concat` skips `null` arguments** instead of printing the word "null" —
  both are usually what you want, but don't be surprised by them.
- **Fan-out is capped**: when a referenced record changes, at most 1000
  referencing records are recomputed per write. For normal workloads this
  never matters; for a table with thousands of inbound links it does.
- A formula can't hold a reference or JSON value: valid `resultKind` values
  are `text`, `long_text`, `email`, `phone`, `number`, `currency`, `checkbox`,
  `date`, `datetime`, and `single_select`. To *follow* a link, use a lookup
  path in the expression; you can't *produce* one.

### Designing a formula: the sequence to follow

Use this sequence rather than guessing from labels or attempting one huge
schema mutation:

1. `GET /v0/agent/tables/{tableID}` and inspect active field **keys** and
   types. Formulas bind to keys (`unit_price`), never labels (`Unit price`).
2. Identify the inputs and choose the formula's `resultKind` *before* writing
   the expression. Use `currency` for money, ratios, percentages, or any value
   that can be fractional; use `number` only for guaranteed integers; use
   `checkbox` for comparisons/boolean logic; use `text` for `concat`/status
   labels.
3. If an input or reference is missing, add that stored field first. For a
   lookup, create the referenced table first, then add the `link_record`, then
   add the formula. For chained formulas, create all inputs and intermediate
   fields in one table-create payload, or add them in dependency order.
4. Add the formula, using a readable field key that describes the *result*
   (`subtotal`, `risk_label`, `margin_pct`) rather than the implementation.
   The API validates the entire expression, result type, missing fields, bad
   lookups, and cycles immediately — treat a `400` as design feedback, not a
   transient failure to retry.
5. Create or PATCH a representative record and inspect the returned `Values`.
   Test the edge cases: null optional inputs, zero denominators, both sides of
   each `if`, and a linked record with a missing/null reference.
6. If this is a new formula on a populated table, backfill each old row with
   `PATCH {"values": {}}`. Then query/sort/filter on the computed field to
   confirm it materialized as intended.

**Formula language boundaries** — formulas are expressions, not a general
scripting language. Do **not** send assignments (`x = 1`), `let`/variables,
blocks, semicolons, SQL, JSON/array literals, arbitrary JavaScript, external
calls, `switch`/`case`, `&&`, `||`, or `!`. Use field keys, the operators
listed above, nested `if`, and the built-in functions only. Use `==` rather
than `=`, and quote literal text (`'priority'`).

### Complete chained formula schema example

This single `POST /v0/agent/tables` payload demonstrates formulas that depend
on stored fields *and* formulas that depend on other formulas. Dependencies
are validated across the complete payload; `subtotal` is computed before
`total_after_discount`, then `order_band`.

```json
{
  "name": "Orders",
  "key": "orders",
  "fields": [
    { "name": "Unit Price", "key": "unit_price", "kind": "currency" },
    { "name": "Quantity", "key": "quantity", "kind": "number" },
    { "name": "Discount %", "key": "discount_pct", "kind": "number" },

    { "name": "Subtotal", "key": "subtotal", "kind": "formula",
      "formula": "unit_price * quantity", "resultKind": "currency" },
    { "name": "Total After Discount", "key": "total_after_discount", "kind": "formula",
      "formula": "round(subtotal * (1 - coalesce(discount_pct, 0) / 100), 2)",
      "resultKind": "currency" },
    { "name": "Order Band", "key": "order_band", "kind": "formula",
      "formula": "if(total_after_discount >= 1000, 'enterprise', if(total_after_discount >= 100, 'priority', 'standard'))",
      "resultKind": "single_select", "options": ["standard", "priority", "enterprise"] }
  ]
}
```

Creating an order sends **only** the three stored inputs:

```json
{ "values": { "unit_price": "125.50", "quantity": 10, "discount_pct": 15 } }
```

The response includes the derived values automatically: a subtotal numerically
equal to `1255`, `total_after_discount` equal to `1066.75`, and
`order_band: "enterprise"`. Currency is a decimal value; callers should compare
it numerically rather than depending on trailing-zero formatting in an immediate
write response. Never send those three formula keys back in a create/update
request.

**Composing complex formulas** — since the language is small but composable,
complex logic is built by nesting the same handful of primitives rather than
reaching for anything exotic. Some patterns:

- **Multi-branch logic** (no native `switch`/`case` — nest `if`):
  ```
  if(subtotal > 1000, 'enterprise',
    if(subtotal > 100, 'priority',
      if(subtotal > 0, 'standard', 'empty')))
  ```
- **Safe division everywhere a denominator could be zero or null**:
  ```
  if(coalesce(quantity, 0) == 0, null, value_impact / quantity)
  ```
- **Multi-field, multi-lookup composition in one string**:
  ```
  concat(upper(customer.tier), ' — ', customer.name, ' (', status, ')')
  ```
- **Weighted/blended scores from several numeric fields**:
  ```
  round(score_a * 0.5 + score_b * 0.3 + score_c * 0.2, 1)
  ```
- **Conditional rounding/normalization combined with a lookup and a guard**:
  ```
  if(product.unit_price == null, null,
    round(quantity_change * product.unit_price, 2))
  ```
- **Boolean flags used by other formulas** — build a same-row boolean field
  first (`is_overdue = due_date < today()`), then reference it from a text
  formula (`status = if(is_overdue, 'late', 'on_time')`) instead of repeating
  the condition — this also keeps each formula's `formula` string short and
  readable, and matches the order dependent fields are evaluated in.

If an expression gets hard to read as one line, that's a sign to split it
into two chained formula fields (one computing an intermediate value, the
next consuming it) rather than writing one giant nested expression — both
work identically since formulas can depend on other formulas, but two named
fields are far easier for the user to inspect and debug in the grid than one
opaque one-liner.

### Records (data)

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/v0/agent/tables/{tableID}/records` | Query/list records. |
| `POST` | `/v0/agent/tables/{tableID}/records` | Create a record. |
| `PATCH` | `/v0/agent/tables/{tableID}/records/{recordID}` | Update a record. |
| `DELETE` | `/v0/agent/tables/{tableID}/records/{recordID}` | Soft-delete a record. |
| `GET` | `/v0/agent/tables/{tableID}/fields/{fieldID}/reference-options` | Search linked-record choices for a `link_record` field. |

**Create a record** — `POST /v0/agent/tables/{tableID}/records`:

```json
{ "values": { "full_name": "Jamie Rivera", "email": "jamie@example.com", "stage": "new" } }
```

`values` keys are field **keys** (not field IDs, not display names). Never
include `formula` field keys in `values` — they're computed automatically and
a direct write is rejected (`FIELD_READ_ONLY`); they *do* appear in response
`Values`, already computed.

**Stored-value quick reference:** send strings for `text`, `long_text`,
`email`, and `phone`; an integer JSON number for `number`; a decimal string
such as `"125.50"` for `currency`; a JSON boolean for `checkbox`; an ISO date
(`"2026-08-29"`) for `date`; RFC3339 (`"2026-08-29T14:30:00Z"`) for
`datetime`; a JSON object/array/value for `json`; and the target record's `ID`
string for `link_record`. For `single_select`, send the option's stored value:
options are normalized to lowercase snake_case, so an option submitted as
`"In Progress"` is stored and written as `"in_progress"`. Fetch the table
schema after creating/updating a field if you need the exact normalized option.

Response is the created record: `{ "ID": "...", "Values": {...}, "RecordVersion": 1, "CreatedBy": "...", "UpdatedBy": "...", "CreatedAt": "...", "UpdatedAt": "..." }`.
`CreatedBy`/`UpdatedBy` are set automatically to whichever actor made the
request — your own API key shows up as `apikey:<prefix>`. You never set these
yourself; they're read-only and ignored if present in a request body.

**Query/list records** — `GET /v0/agent/tables/{tableID}/records`, query
params (all optional):

- `limit` — page size, default 20, max 500.
- `cursor` — pass back the `nextCursor` from a previous page to continue.
- `search` — broad case-insensitive contains search across every active text
  and single-select field. It combines with (does not replace) the one filter
  below.
- `filterFieldId`, `filterOperator`, `filterValue` — one filter. Every field
  supports `eq`, `neq`, `is_null`, and `is_not_null` (omit `filterValue` for
  the two null operators). Text/select fields also support `contains`;
  integer/currency/date/datetime fields also support `gt`, `gte`, `lt`, `lte`.
  JSON fields are only useful with the null operators through this HTTP API.
- `sortFieldId`, `sortDirection` (`asc`/`desc`).

Response: `{ "table": {...}, "records": [...], "nextCursor": "...", "hasMore": true|false, "limit": 20, "totalRecords": 123 }`. `totalRecords` is
the count matching the current search/filter, before pagination. In query
responses, a non-null `link_record` value is automatically shallow-expanded
for display, e.g. `"customer": { "id": "rec_...", "name": "Ada", "email":
"..." }` (the exact display/secondary fields are inferred from the target
schema). This is read-only response convenience: when creating/updating, still
send only the target record ID string (`"customer": "rec_..."`).
If `hasMore` is true and you need more rows, repeat the call with `cursor` set
to `nextCursor`.

**Update a record** — `PATCH /v0/agent/tables/{tableID}/records/{recordID}`:

```json
{ "values": { "stage": "contacted" }, "expectedVersion": 1 }
```

Only send the field keys you're changing — this is a partial update, not a
full replace. `expectedVersion` is optional but recommended: pass the
record's current `RecordVersion` (from a prior create/get/query) to catch the
case where something else changed the row first. If the version doesn't
match, you get `409 Conflict` — re-fetch the record and decide whether to
retry.

**Delete a record** — `DELETE /v0/agent/tables/{tableID}/records/{recordID}`.
This is a soft delete (recoverable by the app owner), not permanent. You may
send `{ "expectedVersion": 3 }` as a JSON body to protect against deleting a
row another caller changed first; success returns `{ "deleted": true }`.

**Resolve a linked-record field's choices** — when a table has a `link_record`
field and you need to know what you can point it at, call
`GET /v0/agent/tables/{tableID}/fields/{fieldID}/reference-options?search=<text>&limit=20&cursor=<cursor>`.
`search`, `limit`, and `cursor` are optional; pass the returned `nextCursor`
for the next page when `hasMore` is true. Response:
`{ "options": [{ "recordId": "rec_...", "label": "...", "secondary": ["..."] }], "nextCursor": "...", "hasMore": true }`.
Use each returned `recordId` as the `values` entry for that field.

## What this key cannot do

By design, an agent key can only touch the schema and record routes listed
above. The following actions are **not exposed under `/v0/agent/*`**; do not
attempt to call a human `/builder/...` route or work around that boundary. An
agent key is not a human user session, so a human-only route is normally
rejected by its own auth/router layer (commonly `401` or `404`, depending on
deployment), not necessarily `403`:

- Inviting or removing members, or changing anyone's role.
- Reading the audit log.
- Creating or renaming the app itself, or changing app-level settings.
- Anything related to views, forms, or dashboards — those aren't exposed
  through this API at all; manage them in the product UI.

If a task needs one of these, tell the user to do it themselves in the
product UI rather than attempting a workaround.

## Errors

| Status | Meaning | What to do |
| --- | --- | --- |
| `400` | Bad request (validation failed, bad filter, invalid/cyclic formula, division by zero at write time, etc.) | Read the `error` field in the JSON body and fix the request. |
| `401` | Missing/invalid/expired/revoked key | Tell the user; don't retry blindly. |
| `403` | Authenticated actor lacks app/role permission | Not expected for normal scoped agent-table operations. Do not try to bypass it; check the intended route and app scope. |
| `404` | Table, field, or record not found | Double-check the ID; it may have been deleted. |
| `409` | Version conflict (schema or record changed since you last read it) | Re-fetch and retry with the current version. |

## Worked example: build a simple CRM

1. `POST /v0/agent/tables` — create `leads` with fields `full_name` (text,
   required), `email` (email), `stage` (single_select: new/contacted/won/lost).
2. Still in the same create payload (or via `POST .../fields` afterwards), add
   derived columns:
   ```json
   { "name": "Summary", "key": "summary", "kind": "formula",
     "formula": "concat(full_name, ' — ', upper(stage))", "resultKind": "text" },
   { "name": "Won", "key": "is_won", "kind": "formula",
     "formula": "stage == 'won'", "resultKind": "checkbox" }
   ```
   Neither is ever written by hand; both populate on every create/update.
3. For each new lead the user describes, `POST .../leads-table-id/records`
   with `values` (only the hand-entered fields).
4. To follow up on stale leads: `GET .../records?filterFieldId=<stage field id>&filterOperator=eq&filterValue=new`,
   then `PATCH` each one's `stage` as you act on it.
5. To report progress: `GET .../records?limit=500` and summarize by `stage`
   (you can also sort by the `is_won` formula field directly).
6. If the formula fields were added *after* rows already existed, backfill by
   `PATCH`-ing each record with `{"values": {}}`.

The same shape works for an invoice log, a monitoring feed, an inventory
tracker, or anything else table-shaped — only the field names and workflow
change, not the API calls.

## Worked example: linked tables with lookup formulas (inventory)

This is the canonical multi-table shape: two tables joined by a `link_record`
field, plus formulas that reach one hop across the link.

1. Create the *referenced* table first — `POST /v0/agent/tables` → `products`
   with `name` (text), `unit_price` (currency), `stock_qty` (number). Keep its
   `table.ID`.
2. Create the *referencing* table — `POST /v0/agent/tables` → `inventory_logs`
   with fields:
   - `product` — `link_record`, `targetTableId` = the products table ID
   - `log_type` — `single_select`: `inbound` / `outbound` / `adjustment`
   - `quantity_change` — `number` (write outbound rows as negative numbers)
   - `value_impact` — `formula`: `round(quantity_change * product.unit_price, 2)`,
     `resultKind: "currency"` — reads the linked product's price, one hop
   - `is_outbound` — `formula`: `log_type == 'outbound'`, `resultKind: "checkbox"`
3. Create log rows with only `product` / `log_type` / `quantity_change` in
   `values` — the response comes back with `value_impact` and `is_outbound`
   already computed.
4. If a product's price later changes, `PATCH` just the product — every log
   row linked to it recomputes `value_impact` automatically (fan-out). Don't
   loop over the log rows yourself; that's handled for you.

The pattern generalizes: any time a row in table A should show data owned by a
row in table B, put a `link_record` on A and a lookup formula (`b.some_field`)
instead of copying values by hand — copies go stale, lookups fan out.
