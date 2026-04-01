---
name: digits-mcp
description: |
  Query and analyze live financial data from Digits using the Digits MCP.
  Use this skill whenever the user asks about their business finances, transactions,
  spending, revenue, cash flow, or anything involving Digits data — even if they
  don't say "Digits" explicitly. Triggers on: "show me my expenses", "how much did
  we spend on X", "what's our revenue this quarter", "find transactions for vendor Y",
  "payroll last month", "top vendors", "cash position", any question about financial
  data or transaction history. Always use this skill if Digits connector tools are
  available in the session.
---

# Digits MCP Skill

Use this skill to answer financial questions by querying live data from Digits.
The tools give you access to businesses, users, and transaction-level data with
rich filtering.

---

## Tool Discovery

Digits connector tools are prefixed with a UUID. Look for tools matching these names
(the `*` prefix is the MCP UUID — it varies per session):

| Tool | Purpose |
|------|---------|
| `*__list_businesses` | List all businesses the user has access to |
| `*__select_business` | Select a business to work with (returns a `business_id`) |
| `*__list_business_users` | List all users with access to a business |
| `*__query_transactions` | Query and filter transactions |

---

## Standard Workflow

Always follow this order:

1. **`list_businesses`** — No parameters needed. Returns a list of businesses with IDs.
2. **`select_business`** — Pass the `business_id` you want. Returns context for that business.
3. **`query_transactions` / `list_business_users`** — Pass the same `business_id`.

If the user has only one business, select it automatically. If they have multiple,
ask which one they mean (or infer from context).

---

## Building a `query_transactions` Call

Every call requires:
- `business_id` — from `select_business`
- `origin` — the time window

### Origin: Specifying the Time Period

The `origin` object defines a time window. All four fields are required:

```json
{
  "interval": "Month",
  "year": 2025,
  "index": 3,
  "interval_count": 1
}
```

| Field | Meaning |
|-------|---------|
| `interval` | `"Day"`, `"Week"`, `"Month"`, `"Quarter"`, or `"Year"` |
| `year` | Calendar year (e.g., `2025`) |
| `index` | Position within year — see table below |
| `interval_count` | How many periods to span (use `1` for a single period) |

**Index values by interval:**

| Interval | Index | Example |
|----------|-------|---------|
| Month | 1–12 | `index: 3` = March |
| Quarter | 1–4 | `index: 2` = Q2 (Apr–Jun) |
| Week | 1–52 | `index: 14` = 14th week of year |
| Year | 1 | Always `1` |
| Day | 1–366 | Day of year |

**Multi-period example** — last 3 months ending March 2025:
```json
{ "interval": "Month", "year": 2025, "index": 3, "interval_count": 3 }
```

**Full year example:**
```json
{ "interval": "Year", "year": 2025, "index": 1, "interval_count": 1 }
```

---

## Common Filter Patterns

The `filter` field is optional. Combine any of these as needed:

### By Category Type (highest level)
```json
"category_types": { "types": ["Expenses"] }
```
Valid types: `"Expenses"`, `"Income"`, `"Assets"`, `"Liabilities"`, `"Equity"`,
`"CostOfGoodsSold"`, `"OtherIncome"`, `"OtherExpenses"`

### By Category Subtype (more specific)
```json
"category_subtypes": ["Payroll", "PayrollExpenses"]
```
Common subtypes: `"Payroll"`, `"PayrollExpenses"`, `"SalesRevenue"`,
`"SubscriptionRevenue"`, `"ProfessionalFees"`, `"TravelAndEntertainment"`,
`"Facilities"`, `"BusinessApplicationsAndSoftware"`, `"CostOfGoods"`

### By Amount Range
```json
"minimum": 1000,
"maximum": 50000
```
Values are in USD dollars (e.g., `1000` = $1,000.00).

### By Transaction Type
```json
"digits_transaction_types": ["PayOut"]
```
Types: `"PayIn"` (money in), `"PayOut"` (money out), `"JournalEntry"`, `"BankTransfer"`

### By Vendor / Party Name
```json
"party_name": "Stripe"
```
Partial match on vendor, customer, or counterparty name.

### By Party Role
```json
"party_roles": ["EntityVendorRole"]
```
Common roles: `"EntityVendorRole"`, `"EntityCustomerRole"`, `"EntitySupplierRole"`

### Exclude Transfers
```json
"omit_transfers": true
```
Use this when you want only operating transactions, not internal bank moves.

---

## Sorting Results

Pass an `order` object to control sort order:

```json
"order": {
  "transaction_order": [
    { "column_name": "Amount", "direction": "Descending" }
  ]
}
```

Sortable columns: `"Amount"`, `"OccurredAt"`, `"Description"`, `"Party"`,
`"Category"`, `"AbsoluteAmount"`

---

## Example Queries

### "Show me expenses last month" (assume current month is April 2025)
```json
{
  "business_id": "<id>",
  "origin": { "interval": "Month", "year": 2025, "index": 3, "interval_count": 1 },
  "filter": {
    "category_types": { "types": ["Expenses"] },
    "omit_transfers": true
  },
  "order": { "transaction_order": [{ "column_name": "Amount", "direction": "Descending" }] }
}
```

### "How much did we spend on payroll in Q1 2025?"
```json
{
  "business_id": "<id>",
  "origin": { "interval": "Quarter", "year": 2025, "index": 1, "interval_count": 1 },
  "filter": {
    "category_subtypes": ["Payroll", "PayrollExpenses"]
  }
}
```

### "Top vendors by spend this year"
```json
{
  "business_id": "<id>",
  "origin": { "interval": "Year", "year": 2025, "index": 1, "interval_count": 1 },
  "filter": {
    "category_types": { "types": ["Expenses"] },
    "party_roles": ["EntityVendorRole"],
    "omit_transfers": true
  },
  "order": { "transaction_order": [{ "column_name": "AbsoluteAmount", "direction": "Descending" }] }
}
```

### "Large transactions over $10k this quarter"
```json
{
  "business_id": "<id>",
  "origin": { "interval": "Quarter", "year": 2025, "index": 2, "interval_count": 1 },
  "filter": {
    "minimum": 10000,
    "omit_transfers": true
  },
  "order": { "transaction_order": [{ "column_name": "Amount", "direction": "Descending" }] }
}
```

---

## Summarizing Results

After calling `query_transactions`, the response includes transaction records.
When presenting to the user:

- **Group and sum** by category or party when the user asks for totals (e.g., "how much did we spend on X")
- **List individually** when the user asks to see or review specific transactions
- **Format amounts** as currency with commas (e.g., `$12,345.67`)
- **Use plain language** for dates — "March 2025" not "2025-03-01"
- **Surface anomalies** if you notice anything unusual (large one-off items, duplicate-looking transactions)
- If results are empty, suggest broadening the time window or relaxing the filter

---

## Tips

- The `direction` parameter (`"Past"` or `"Future"`) shifts the window relative to `origin`. Default/omit it for standard historical queries.
- `interval_count > 1` spans multiple periods ending at `index`. E.g., `index: 6, interval_count: 6` = January through June.
- When a user asks about "this month" or "last quarter", infer the current date and compute the correct `year` + `index` values yourself — don't ask the user.
- If the user asks a question that might require multiple queries (e.g., comparing two time periods), run them in sequence and present a unified answer.
