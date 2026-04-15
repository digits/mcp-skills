---
name: digits-mcp
description: |
  Query and analyze live financial data from Digits using the Digits MCP.
  Use this skill whenever the user asks about their business finances, transactions,
  spending, revenue, cash flow, or anything involving Digits data — even if they
  don't say "Digits" explicitly. Triggers on: "show me my expenses", "how much did
  we spend on X", "what's our revenue this quarter", "find transactions for vendor Y",
  "payroll last month", "top vendors", "cash position", "P&L", "balance sheet",
  "profit and loss", "financial statement", "accounts payable", "accounts receivable",
  any question about financial data or transaction history, summaries by department
  or location, or questions requiring category lookups. Always use this skill if
  Digits connector tools are available in the session.
---

# Digits MCP Skill

Use this skill to answer financial questions by querying live data from Digits.
The tools give you access to businesses, users, transaction-level data, aggregated
summaries, financial statements, and dimension lookups (categories, departments,
locations, parties).

---

## Tool Reference

Digits connector tools are prefixed with a UUID. Look for tools matching these names
(the `*` prefix is the MCP UUID — it varies per session):

| Tool | Purpose |
|------|---------|
| `*__list_businesses` | List all businesses the user has access to |
| `*__select_business` | Select a business to work with (required before most other tools) |
| `*__list_business_users` | List all users with access to a business |
| `*__search_term` | Resolve a name (vendor, category, department, location) to its canonical ID |
| `*__query_transactions` | Query and filter individual transactions |
| `*__dimensional_summarize_transactions` | Aggregate transactions into multi-dimensional summaries (by Party, Category, Time, etc.) |
| `*__financial_statement` | Generate P&L, Balance Sheet, Cash Flow, AR/AP Aging statements |
| `*__list_categories` | List all categories with their IDs and types |
| `*__list_departments` | List all departments with their IDs |
| `*__list_locations` | List all locations with their IDs |

---

## Standard Workflow

Always start here:

1. **`list_businesses`** — No parameters needed. Returns a list of businesses with IDs.
2. **`select_business`** — Pass the `business_id` you want. Activates that business context.

If the user has only one business, select it automatically. If they have multiple, ask which one (or infer from context).

After selecting a business, use the appropriate tool(s) for the question. See "Choosing the Right Tool" below.

---

## Choosing the Right Tool

| User question type | Best tool |
|---|---|
| "Show me all transactions for Stripe last month" | `query_transactions` |
| "Top vendors by spend this quarter" | `dimensional_summarize_transactions` (along=Party) |
| "Monthly revenue trend this year" | `dimensional_summarize_transactions` (along=Time) |
| "How much did we spend on payroll vs software?" | `dimensional_summarize_transactions` (along=Category) |
| "What's our P&L for Q3?" | `financial_statement` (kind=ProfitAndLoss) |
| "Show me the balance sheet" | `financial_statement` (kind=BalanceSheet) |
| "What's our total A/R outstanding?" | `financial_statement` (kind=ARAging) |
| "What categories are under Expenses?" | `list_categories` |
| "Which departments do we have?" | `list_departments` |
| "Find the ID for vendor 'Acme Inc'" | `search_term` |
| "Who has access to this business?" | `list_business_users` |

---

## Resolving Names to IDs with `search_term`

Before filtering by party, category, department, or location, use `search_term` to resolve the user's phrase to a canonical ID. Do not guess or hard-code IDs.

**Required parameter:**
- `text`: The search string (e.g., `"uber eats"`, `"payroll"`, `"marketing dept"`)

**Optional parameter:**
- `kinds`: Limit what object types to search. Valid values: `"Party"`, `"Category"`, `"Department"`, `"Location"`, `"Transaction"`.
  - When searching for transaction-related terms, always pass all five: `["Party", "Category", "Department", "Location", "Transaction"]`
  - When specifically resolving a vendor/customer name, pass `["Party"]`

**Important verification step:** After getting results, confirm the top match truly corresponds to the user's intent. For example, "Uber" is not necessarily "Uber Eats" — if multiple close matches exist, ask the user to confirm before proceeding.

```json
{
  "text": "uber eats",
  "kinds": ["Party", "Category", "Department", "Location", "Transaction"]
}
```

---

## Specifying Time Periods (`origin`)

All data-fetching tools require an `origin` object. All four fields are required:

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
| `interval_count` | Number of periods to span (use `1` for a single period) |

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

When a user says "this month", "last quarter", etc., infer the current date and compute `year` + `index` yourself — don't ask the user.

---

## `query_transactions` — Individual Transactions

Use this tool when the user wants to see or review specific transactions, not just totals.

**Required:** `origin` (time period)

**Optional filter fields:**
- `minimum` / `maximum`: USD dollar amounts as numbers (e.g., `10000` = $10,000)
- `occurred_after` / `occurred_before`: Timestamps for tighter date bounds
- `category_ids`: `{"ids": ["cat-id-1"]}` — resolve with `search_term` or `list_categories` first
- `category_types`: `{"types": ["Expenses"]}` — valid types: `"Expenses"`, `"Income"`, `"Assets"`, `"Liabilities"`, `"Equity"`, `"CostOfGoodsSold"`, `"OtherIncome"`, `"OtherExpenses"`
- `party_ids`: `{"ids": ["party-id-1"]}` — requires either `category_types` or `party_roles`. **Always resolve with `search_term` first — the tool does not accept party names directly.**
- `party_roles`: `["EntityVendorRole", "EntityCustomerRole", "EntitySupplierRole"]`
- `digits_transaction_types`: `["PayIn"]`, `["PayOut"]`, `["JournalEntry"]`, `["BankTransfer"]`

**Optional `order`:** Sort results, e.g.:
```json
{ "transaction_order": [{ "column_name": "Amount", "direction": "Descending" }] }
```
Sortable columns: `"Amount"`, `"OccurredAt"`, `"Description"`, `"Party"`, `"Category"`, `"AbsoluteAmount"`

**Optional `pagination`:** Defaults to `{"offset": 0, "limit": 250}` when omitted.

### Example — expenses last month (assume current month is April 2025)
```json
{
  "origin": { "interval": "Month", "year": 2025, "index": 3, "interval_count": 1 },
  "filter": {
    "category_types": { "types": ["Expenses"] },
    "digits_transaction_types": ["PayOut"]
  },
  "order": { "transaction_order": [{ "column_name": "Amount", "direction": "Descending" }] }
}
```

### Example — transactions for a specific vendor over $10k
First call `search_term` with `"text": "acme inc"` and `"kinds": ["Party"]` to get the `party_id`, then:
```json
{
  "origin": { "interval": "Quarter", "year": 2025, "index": 1, "interval_count": 1 },
  "filter": {
    "party_ids": { "ids": ["<party_id_from_search_term>"] },
    "party_roles": ["EntityVendorRole"],
    "minimum": 10000
  }
}
```

### Example — large transactions this quarter
```json
{
  "origin": { "interval": "Quarter", "year": 2025, "index": 2, "interval_count": 1 },
  "filter": { "minimum": 10000 },
  "order": { "transaction_order": [{ "column_name": "Amount", "direction": "Descending" }] }
}
```

---

## `dimensional_summarize_transactions` — Aggregated Summaries

Use this tool when the user wants totals, trends, or breakdowns — not individual transactions. It returns amounts aggregated along one or more dimensions.

**Required:** `origin`, and at least one dimension in `along.dimensions`

**`along.dimensions`** — what to group by:
- `"Party"` — break down by vendor/customer
- `"Category"` — break down by account category
- `"Time"` — break down by time period (required when `interval_count > 1`)
- `"Department"` — break down by department
- `"Location"` — break down by location

**Important rules:**
- When requesting only a Time dimension, you must also include a filter (category, party, etc.)
- Include `"Time"` in dimensions whenever `interval_count > 1`
- When filtering by balance sheet categories (Assets, Liabilities, Equity), set `"as_permanent_account": true`
- Cannot answer questions specific to bills or invoices
- Always resolve party / category / department / location using `search_term` (or the list tools) before using their IDs in filters

**Filter fields** (same semantics as `query_transactions`): `category_ids`, `category_types`, `party_ids`, `party_roles`, `minimum`, `maximum`

**Optional `pagination`** (only usable with a single dimension):
```json
{
  "sort_direction": "Descending",
  "sort_field": "Amount",
  "page": { "limit": 5 }
}
```

### Example — top 5 vendors by spend, Q3 2024
```json
{
  "origin": { "interval": "Quarter", "year": 2024, "index": 3, "interval_count": 1 },
  "filter": { "category_types": { "types": ["Expenses"] } },
  "along": { "dimensions": ["Party"] },
  "pagination": { "sort_direction": "Descending", "sort_field": "Amount", "page": { "limit": 5 } }
}
```

### Example — monthly revenue trend for a customer in 2024
First resolve the customer with `search_term`, then:
```json
{
  "origin": { "interval": "Month", "year": 2024, "index": 12, "interval_count": 12 },
  "filter": {
    "party_ids": { "ids": ["<party_id>"] },
    "party_roles": ["EntityCustomerRole"],
    "category_types": { "types": ["Income"] }
  },
  "along": { "dimensions": ["Time", "Party"] }
}
```

### Example — spending by party, filtered to a specific category, June–August 2023
First resolve the category with `search_term`, then:
```json
{
  "origin": { "interval": "Month", "year": 2023, "index": 8, "interval_count": 3 },
  "filter": { "category_ids": { "ids": ["<category_id>"] } },
  "along": { "dimensions": ["Party"] },
  "pagination": { "sort_direction": "Descending", "sort_field": "Amount" }
}
```

**Note:** Pagination can only be used with a single dimension. If you want to compare top parties across two months, run separate paginated queries per month.

---

## `financial_statement` — P&L, Balance Sheet, Cash Flow, Aging Reports

Use this tool for high-level financial statements. Do not use `query_transactions` or `dimensional_summarize_transactions` to approximate a statement — use this directly.

**Required parameters:**
- `kind`: One of `"ProfitAndLoss"`, `"BalanceSheet"`, `"CashFlow"`, `"APAging"`, `"ARAging"`
- `origin`: Time period (same structure as above)

**Statement types:**
- `ProfitAndLoss` — Income Statement: revenue, expenses, net income
- `BalanceSheet` — Assets, liabilities, and equity at a point in time
- `CashFlow` — Cash movements by operating / investing / financing activities
- `APAging` — Total amount currently owed to vendors. Use **only** for outstanding totals, not for general bill details/history.
- `ARAging` — Total amount currently owed by customers. Use **only** for outstanding totals, not for general invoice details/history.

**Optional parameters:**
- `look_back_count`: Override the default lookback (defaults: 12 months, 4 quarters, 3 years)
- `show_account_numbers`: Include account numbers in the output (bool)
- `fiscal_year_start_month`: Fiscal year start month (1–12)
- `department_ids`: Filter to specific departments (array of strings — resolve with `list_departments` first)
- `location_ids`: Filter to specific locations (array of strings — resolve with `list_locations` first)
- `category_id`: Filter to a specific category

### Example — P&L for Q3 2024
```json
{
  "kind": "ProfitAndLoss",
  "origin": { "interval": "Quarter", "year": 2024, "index": 3, "interval_count": 1 }
}
```

### Example — Balance Sheet for December 2024
```json
{
  "kind": "BalanceSheet",
  "origin": { "interval": "Month", "year": 2024, "index": 12, "interval_count": 1 }
}
```

### Example — Cash Flow for last 6 months
```json
{
  "kind": "CashFlow",
  "origin": { "interval": "Month", "year": 2024, "index": 12, "interval_count": 6 }
}
```

### Example — Total outstanding A/P with account numbers
```json
{
  "kind": "APAging",
  "origin": { "interval": "Month", "year": 2024, "index": 12, "interval_count": 1 },
  "preferences": { "show_account_numbers": true }
}
```

### Example — Department-level P&L
Call `list_departments` to get IDs, then:
```json
{
  "kind": "ProfitAndLoss",
  "origin": { "interval": "Quarter", "year": 2024, "index": 4, "interval_count": 1 },
  "department_ids": ["dept-123", "dept-456"]
}
```

---

## `list_categories`, `list_departments`, `list_locations`

Use these to enumerate available entities before using their IDs in filters. All three require no parameters and return a list of entities with `id`, `name`, and relevant metadata.

**When to use each:**
- `list_categories` — when you need category names, types, or IDs (e.g., to understand what's under "Expenses" or find a specific account)
- `list_departments` — when you need department IDs to filter a statement or summary
- `list_locations` — when you need location IDs to filter a statement or summary

After listing, pass the returned IDs to `dimensional_summarize_transactions`, `query_transactions`, or `financial_statement` as appropriate.

---

## `list_business_users`

Returns all users with access to the selected business.

**Required:** `business_id` from `select_business`

Use when the user asks "who has access to this business?" or "show me all users."

---

## Presenting Results

- **Group and sum** by category or party when the user asks for totals
- **List individually** when the user wants to review specific transactions
- **Format amounts** as currency with commas (e.g., `$12,345.67`)
- **Use plain language** for dates ("March 2025" not "2025-03-01")
- **Surface anomalies** if you notice anything unusual (large one-off items, apparent duplicates)
- If results are empty, suggest broadening the time window or relaxing a filter

---

## Tips

- `interval_count > 1` spans multiple periods ending at `index`. E.g., `index: 6, interval_count: 6` = January through June.
- If a question requires multiple queries (e.g., comparing two time periods), run them in sequence and present a unified answer.
- The `direction` parameter (`"Past"` or `"Future"`) shifts the window relative to `origin`. Omit it for standard historical queries.
- For AP/AR questions about *total outstanding balances*, use `financial_statement`. For *individual bill or invoice details*, use `query_transactions` with appropriate category filters.
- The response from `dimensional_summarize_transactions` may include a `summary` field where the current period amount is zero while the prior period is non-zero — do not treat prior-period amounts as current values.
