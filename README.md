# Automated Dynamic Price Management
### n8n + PlentyOne REST API + Google Sheets

A production automation workflow that manages pricing for slow-moving e-commerce inventory — running daily without manual intervention. Built for a German retail client using the PlentyOne/PlentyMarkets platform.

---

## The Problem

Managing sale prices across dozens of product variants is time-consuming and inconsistent when done manually. The business needed:

- Gradual, automatic price reductions for products that haven't sold in 3+ days
- An enforced price floor so nothing is ever sold below cost + margin
- Automatic price recovery when items sell (to capture margin while demand exists)
- A full audit trail of every price change
- Persistent state tracking across daily runs
- Zero manual work after initial setup

---

## The Solution

A 48-node n8n workflow that runs every night at 02:00 (Europe/Berlin). It pulls live product and order data from the PlentyOne API, compares it against a Google Sheets state tracker, decides what action each product needs, applies the price change, and logs everything — all in one automated run.

---

## Measured Impact

| Metric | Before | After |
|--------|--------|-------|
| Weekly pricing hours | 4–6 hrs/week (manual) | 0 hrs (fully automated) |
| Price update frequency | 1–2x/week | Daily at 02:00 |
| Avg. days to first markdown | 7–14 days | 3 days |
| Floor price violations | Occasional (human error) | Zero (enforced programmatically) |
| Pricing decisions logged | None | 100% — full audit trail |
| Weekend/holiday coverage | None | Every day, no exceptions |

**Estimated annual value: €5,400 – €8,000**, broken down as:

- **€3,720** — labour savings (248 hrs/year × €15/hr)
- **€1,440 – €3,600** — revenue recovered from 15–25% faster slow-stock turnover
- **€200 – €500** — margin recaptured via the +€2 post-sale price recovery mechanism
- **€72 – €240** — margin loss prevented by eliminating below-cost pricing errors

Payback period from build cost: **under 4 weeks**.

---

## How It Works

### Stage 1 — Login
Authenticates against the PlentyOne REST API and obtains a short-lived access token used by all downstream requests.

### Stage 2 — Fetch Data (3 parallel branches)
Three API calls run simultaneously:
- **Tag 16 variants** — fetches all product variants in pricing group 1 with their sale prices and stock levels
- **Tag 17 variants** — fetches all product variants in pricing group 2
- **Recent orders** — fetches the last 24 hours of orders to detect which products sold today

Each product loop extracts the purchase price (EK), current sale prices (Price ID 3 = RRP, Price ID 17 = dynamic sale price), stock level, and computes the minimum allowed price using a per-group floor formula.

**Floor price formulas:**
```
Group 1 (Tag 16):  floor = ((EK + 3.84) / 0.81 / 0.89) + 8.96
Group 2 (Tag 17):  floor = ((EK + 0.84) / 0.81 / 0.89) + 8.96
```

### Stage 3 — Combine and Look Up History
All variants are merged into one stream. Only in-stock variants proceed to the state lookup. The Google Sheets State tab is read once (via an Aggregate node to avoid redundant API calls) and joined to each variant using an advanced Merge node.

Simultaneously, the order data is matched against variants to flag which products sold today.

### Stage 4 — Enrich
A Code node normalises the merged data into a clean, consistent shape — parsing dates, computing `daysSinceLastSale` and `daysSinceLastChange`, resolving null sentinel values (`999` = never happened), and setting the `soldToday` boolean.

### Stage 5 — Decision Router
Each variant is evaluated against four prioritised rules (first match wins):

| Priority | Condition | Action |
|----------|-----------|--------|
| 1 | Sold today | Raise price by 2 EUR, capped at RRP |
| 2 | At floor price for 7+ days, no recent sale | Reset to RRP, add Tag 15 |
| 3 | No dynamic price set yet, no sale in 3+ days | Set first discounted price (RRP minus 1 EUR) |
| 4 | Has a dynamic price, no change in 3+ days, not sold | Reduce by 1 EUR |
| — | None of the above | Skip — no change |

### Stage 6 — Price Updates and Logging
Each active branch:
1. Calculates the new price
2. Sends the update to PlentyOne via the REST API (`continueOnFail` — one failure does not stop the run)
3. On success: appends a row to the **Price Change Log** sheet and upserts the **State** sheet
4. On failure: appends a row to the **Errors** sheet

---

## Architecture Highlights

**Parallel fetch branches** — Tag 16, Tag 17, and recent orders all fetched simultaneously, cutting total runtime.

**Single Sheets read** — an Aggregate node collapses all variants to one item before calling Google Sheets, so the State tab is read exactly once per run regardless of how many variants exist.

**Dual sync paths with Merge1** — a synchronisation Merge node waits for both the state-lookup path and the sold-detection path to complete before enrichment runs, ensuring data is always complete.

**Immediate sale date write** — when a sold variant is detected, the State sheet is updated right away (before the pricing decision), so the decision always reflects the current sale status.

**Floor price enforcement** — every price calculation uses `Math.max(newPrice, floorPrice)` to guarantee the price never drops below cost + margin, regardless of how many reduction cycles have run.

**Idempotent design** — the workflow is safe to re-run. State is always read fresh, prices are only changed when conditions are met, and duplicate runs produce the same outcome.

**Error isolation** — `continueOnFail` on all price-update nodes means a single API error affects only that variant. All others continue processing normally.

---

## Tech Stack

| Component | Purpose |
|-----------|---------|
| **n8n** (self-hosted) | Workflow orchestration, scheduling, node execution |
| **PlentyOne REST API** | Product data, stock, prices, orders, tag management |
| **Google Sheets** | State persistence, audit log, error log |
| **JavaScript (Code nodes)** | Enrichment logic, date calculations, order parsing |

---

## Workflow Nodes (48 total)

```
Manual Trigger → Fetch Access Token
  ├── Fetch Variations Tag 16 → Split Out → Loop Over Items
  │       [items] → Edit Fields → floorPrice16 Calculation → (back to loop)
  │       [done]  → Merge (input 0)
  ├── Fetch Variations Tag 17 → Split Out1 → Loop Over Items1
  │       [items] → Set fields → floorPrice17 Calculation → (back to loop1)
  │       [done]  → Merge (input 1)
  └── Fetch Recent Orders → Extract Sold IDs → Merge Variant with soldId (input 0)

Merge (ALL variants)
  ├── Filter (netStock > 0) → Aggregate → Get row(s) in sheet
  │                                           → Merge Variants with State (input 0)
  │                        → Merge Variants with State (input 1)
  │                                → Merge1 (input 1)
  └── Merge Variant with soldId (input 1)
          ├── Update Last Sale Date & Days No Sale  [terminal]
          └── Merge1 (input 0)

Merge1 → Enrich Variants → Filter Stock > 0 → Decision Router
    ├── [0: sale_increase]    → Calc → PUT → IF → Log / Update State
    ├── [1: reset]            → Calc → PUT → IF → POST Tag15 → Log / Update State
    ├── [2: stage1_init]      → Calc → POST → IF → PUT UUID → Log / Update State
    ├── [3: stage2_reduction] → Calc → PUT → IF → Log / Update State
    └── [skip]               → (no action)

All IF error=true → Log Error
```

---

## Google Sheets Structure

### State tab
One row per product variant, updated on every run.

| Column | Description |
|--------|-------------|
| Variant ID | PlentyOne variation ID |
| Variant Nr | Human-readable SKU |
| Item ID | PlentyOne parent item ID |
| Tag ID | Pricing group (16 or 17) |
| Current Stage | UNINITIALIZED / STAGE_1 / STAGE_2 / AT_FLOOR / RESET |
| Price ID 3 | RRP — read-only reference price |
| Current Price 17 | Active dynamic price |
| EK | Purchase price |
| Floor Price | Calculated minimum allowed price |
| Last Price Change | Date of most recent price update |
| Last Sale Date | Date of most recent detected sale |
| Days at Floor | Days the price has been at minimum |
| Days No Sale | Days since last sale |

### Price Change Log tab
Append-only audit log — one row per price change with timestamp, old price, new price, action type, and reason.

### Errors tab
Append-only error log — one row per API failure with timestamp, variant ID, action attempted, and error message.

---

## State Machine

```
UNINITIALIZED ──► STAGE_1 ──► STAGE_2 ──► AT_FLOOR ──► RESET
                                              ▲
                                         (7 days with no sales)

Any state: if soldToday → price increase (stage stays the same)
```

---

## Setup

> This workflow was built for a specific client environment. To adapt it:

1. **Copy `.env.example` to `.env`** and fill in your credentials:
   ```
   PLENTY_API_URL=https://YOUR_INSTANCE.my.plentysystems.com
   PLENTY_USERNAME=your_username
   PLENTY_PASSWORD=your_password
   GOOGLE_SHEET_ID=your_sheet_id
   ```

2. **Google Sheets** — create a sheet with three tabs: `State`, `Price Change Log`, `Errors`. The State tab column headers must match exactly (see column list above).

3. **n8n credentials** — configure a Google Sheets OAuth2 credential in your n8n instance and update all Google Sheets nodes to use it.

4. **Tag IDs** — update the `variationTagId` filter parameters in the two fetch nodes to match the tag IDs used in your PlentyOne instance.

5. **Floor price formulas** — review and adjust the formulas in `floorPrice16 Calculation` and `floorPrice17 Calculation` to match your cost structure and margin requirements.

6. **Schedule Trigger** — add a Cron Trigger node (`0 2 * * *`) to run automatically. Connect it in parallel with the Manual Trigger for both automated and on-demand execution.

---

## PlentyOne API Notes

Key findings from building against the live API:

- `GET /rest/items?tagId=X` is **broken** — silently ignores the tag filter and returns all items. Always use `variationTagId` on the variations endpoint.
- The auth endpoint uses `username` (not `email`) in the request body.
- `variationSalesPrices` returns an array — filter by `salesPriceId` to find the right price entry. A price entry with `price: 0` means "not set" and should be treated as null.
- Pagination uses `{ page, totalsCount, isLastPage, entries, itemsPerPage }`.
- Order date filters require W3C format with timezone offset: `2026-03-25T00:00:00+01:00`.

---

## Project Files

| File | Description |
|------|-------------|
| `README.md` | This file |
| `workflow-docs.md` | Full technical documentation of every node and data flow |
| `.env.example` | Environment variable template |
