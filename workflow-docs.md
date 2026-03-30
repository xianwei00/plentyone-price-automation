# PlentyOne Price Management — Workflow Documentation

## Overview

This n8n workflow automates dynamic price management for a German e-commerce retailer running on PlentyOne/PlentyMarkets. It runs daily at 02:00 Europe/Berlin, fetches all active variants tagged with Tag 16 or Tag 17, calculates floor prices, checks recent sales, and applies pricing rules automatically.

**Workflow ID:** `[WORKFLOW_ID]`
**n8n instance:** `[N8N_INSTANCE_URL]`
**Timezone:** Europe/Berlin
**Google Sheet:** `[GOOGLE_SHEET_ID]`
**Total nodes:** 48
**Status:** Complete — all pricing branches live

---

## Part A — Complete Data Flow

### 1. Entry Point

```
When clicking 'Execute workflow'  [Manual Trigger]
  → Fetch Access Token
```

**Fetch Access Token** sends `POST {PLENTY_API_URL}/rest/login` with `username` + `password`. The response contains `accessToken` (camelCase), which all downstream nodes reference as `$('Fetch Access Token').item.json.accessToken`.

---

### 2. Three Parallel Branches (all start from Fetch Access Token)

From `Fetch Access Token`, three branches run **simultaneously**:

#### Branch A — Tag 16 Variants
```
Fetch Item Variations Tag ID 16
  → Split Out
      → Loop Over Items (SplitInBatches)
          [items output] → Edit Fields → floorPrice16 Calculation → back to Loop
          [done output]  → Merge (input 0)
```

**Fetch Item Variations Tag ID 16:** `GET /rest/items/variations?variationTagId=16&with=variationSalesPrices,Stock`
Returns paginated response: `{ page, totalsCount, isLastPage, entries: [...] }`

**Split Out:** Unwraps the `entries` array — each item becomes a separate n8n item.

**Loop Over Items (SplitInBatches):** Processes items in batches. Two outputs:
- Output 0 (done): when all batches processed → goes to Merge
- Output 1 (items): current batch → goes to Edit Fields

**Edit Fields (Set node):** Extracts these fields from each variation:
```json
{
  "tagId": "16",
  "variantId": 12345,
  "itemId": 6789,
  "number": "EXAMPLE-SKU-37",
  "netStock": 2,
  "EK": 12.80,
  "variationSalesPrices3": { "salesPriceId": 3, "price3": 38.90 },
  "variationSalesPrices17": { "salesPriceId": 17, "price17": 0 }
}
```

**floorPrice16 Calculation (Set node):** Adds:
```json
{ "floorPrice16": "32.04..." }   // string — formula: ((EK+3.84)/0.81/0.89)+8.96
```

#### Branch B — Tag 17 Variants
Identical structure to Branch A but with `variationTagId=17` and floor price formula `((EK+0.84)/0.81/0.89)+8.96` → stored as `floorPrice17` (string).

#### Branch C — Recent Orders
```
Fetch Recent Orders (last 24h)
  → Extract Sold IDs  [Code node]
      → Merge Variant with soldId  (input 0)
```

**Fetch Recent Orders:** `GET /rest/orders?createdAtFrom={now-24h}&typeId=1&itemsPerPage=250`

**Extract Sold IDs (Code node):** Parses the orders response, deduplicates, and outputs one item per sold variant:
```json
{ "soldVariantId": 12345 }
{ "soldVariantId": 67890 }
```

---

### 3. Post-Loop Merge + First Filter

```
Merge  (receives done signals from Loop Over Items and Loop Over Items1)
  ├── Filter  (netStock > 0)  → feeds state lookup path
  └── Merge Variant with soldId  (input 1)  → feeds sold-detection path
```

**Merge:** Waits for both loop "done" outputs. Combines all Tag 16 and Tag 17 variants into a single stream.

**Filter:** Removes variants with `netStock = 0`. Output feeds the Google Sheets state lookup — only in-stock variants get their state enriched.

---

### 4. Two Parallel Sub-Paths

After the split at `Merge`, two things happen simultaneously:

#### Sub-Path 1 — State Lookup

```
Filter
  ├── Aggregate → Get row(s) in sheet → Merge Variants with State (input 0)
  └── Merge Variants with State (input 1)
        → Merge1 (input 1)
```

**Aggregate:** Collapses all variants into a single item. This ensures `Get row(s) in sheet` executes **once** regardless of how many variants exist.

**Get row(s) in sheet:** Reads all rows from the "State" Google Sheet tab. Sample row:
```json
{
  "row_number": 2,
  "Variant ID": 12345,
  "Variant Nr": "EXAMPLE-SKU-38",
  "Item ID": 6789,
  "Tag ID": 17,
  "Current Stage": "STAGE_2",
  "Price ID 3": 32.95,
  "Current Price 17": 26.88,
  "EK": 8.50,
  "Floor Price": 21.92,
  "Last Price Change": "2026-03-24",
  "Last Sale Date": "2026-03-27",
  "Days at Floor": 0,
  "Days No Sale": 0
}
```

**Merge Variants with State:** Combines on `Variant ID` = `variantId`. Mode: `combine` / advanced. Joins in-stock variants with their state row.

#### Sub-Path 2 — Sold Detection

```
Merge (ALL variants)
  → Merge Variant with soldId (input 1)
      ├── Update Last Sale Date & Days No Sale  [Google Sheets — terminal]
      └── Merge1 (input 0)
```

**Merge Variant with soldId:** Combines `soldVariantId` (input 0, from Extract Sold IDs) with all variants (input 1). Mode: `combine` / advanced / `mergeByFields: soldVariantId = variantId`. Variants that sold get `soldVariantId` attached; others pass through with it undefined.

**Update Last Sale Date & Days No Sale (Google Sheets):** Immediately writes to the State sheet (`appendOrUpdate`, matched on `Variant ID`):
- `Last Sale Date` = today's date
- `Days No Sale` = `0`

This is a **terminal node** — it updates the sheet as a side effect and does not feed anything downstream.

---

### 5. Merge1 — Synchronisation Point

```
Merge1
  Input 0: from Merge Variant with soldId  (all variants + soldId marker)
  Input 1: from Merge Variants with State  (in-stock variants + state data)
    → Enrich Variants
```

**Merge1:** Waits for both sub-paths to complete before sending data to `Enrich Variants`.

---

### 6. Enrichment

```
Enrich Variants  [Code node, runOnceForAllItems]
```

Normalises all fields into a clean, unified shape. Key operations:

| Field | Source | Notes |
|-------|--------|-------|
| `variantId` | `j.variantId \|\| j['Variant ID']` | Both exist after merge |
| `price3` | `j.variationSalesPrices3.price3` | Nested object from API |
| `currentPrice17` | `j.variationSalesPrices17.price17` | `0` → `null` (not set) |
| `floorPrice` | `j.floorPrice16` or `j.floorPrice17` | Based on `tagId`; `parseFloat` converts from string |
| `currentStage` | `j['Current Stage']` | From state sheet |
| `daysAtFloorCounter` | `j['Days at Floor']` | From state sheet |
| `daysSinceLastChange` | `j['Last Price Change']` → date diff | `999` if empty/null |
| `daysSinceLastSale` | `j['Last Sale Date']` → date diff | `999` if null (never sold) |
| `soldToday` | `j.soldVariantId !== undefined && Number(j.soldVariantId) === varId` | `true` if in recent orders |

Output sample:
```json
{
  "variantId": 12345,
  "itemId": 6789,
  "number": "EXAMPLE-SKU-38",
  "tagId": "17",
  "netStock": 3,
  "EK": 8.50,
  "price3": 32.95,
  "currentPrice17": 26.88,
  "floorPrice": 21.92,
  "currentStage": "STAGE_2",
  "daysAtFloorCounter": 0,
  "daysSinceLastChange": 3,
  "daysSinceLastSale": 0,
  "soldToday": false
}
```

---

### 7. Second Filter

```
Filter Stock > 0  (netStock > 0)
  → Decision Router
```

Final stock check after enrichment. Removes any zero-stock variants that may have passed through `Merge1` from the all-variants path.

---

### 8. Decision Router

```
Decision Router  [Switch node, 4 named outputs]
```

Evaluates each variant in order. **First matching rule wins.**

| Output | Name | Condition |
|--------|------|-----------|
| 0 | `sale_increase` | `soldToday == true` |
| 1 | `reset` | `currentPrice17 == floorPrice` AND `daysAtFloorCounter >= 7` AND `!soldToday` |
| 2 | `stage1_init` | `currentPrice17 === null` AND `daysSinceLastSale >= 3` |
| 3 | `stage2_reduction` | `currentPrice17 > 0` AND `daysSinceLastChange >= 3` AND `!soldToday` |
| fallback | `skip` | Anything else — no action taken |

---

### 9. Action Branches

Each active branch follows: **Calculate** → **HTTP Request** (continueOnFail) → **IF Error?** → (success) **Log + Update State** / (error) **Log Error**

#### Branch 0: sale_increase
- **When:** Variant sold in last 24h
- **Calc:** `newPrice17 = round2(min(currentPrice17 + 2, price3))`
- **HTTP:** `PUT .../variation_sales_prices/17`
- **State:** `Current Stage` unchanged, `Last Sale Date` = now

#### Branch 1: reset
- **When:** Price is at floor for 7+ days, no recent sale
- **Calc:** `newPrice17 = price3`
- **HTTP:** `PUT .../variation_sales_prices/17` then `POST .../variation_tags` with `{"tagId": 15}`
- **State:** `Current Stage = RESET`, `Days at Floor = 0`

#### Branch 2: stage1_init
- **When:** Price 17 has never been set (null) AND no sale in 3+ days
- **Calc:** `newPrice17 = round2(max(price3 - 1, floorPrice))`
- **HTTP:** `POST .../variation_sales_prices/17` (create) then `PUT .../variations/{variantId}` with `{"priceCalculationUuid": "[PRICE_CALCULATION_UUID]"}`
- **State:** `Current Stage = STAGE_1`, `Days at Floor = 0`

#### Branch 3: stage2_reduction
- **When:** Price 17 exists AND no change in 3+ days AND not sold today
- **Calc:** `newPrice17 = round2(max(currentPrice17 - 1, floorPrice))`
- **State:** `Current Stage = STAGE_2` or `AT_FLOOR` (if new price == floorPrice); `Days at Floor` increments when at floor

---

### 10. Logging

**Price Change Log (append):** Written for every successful price change.
- Columns: `Timestamp`, `Variant ID`, `Variant Nr`, `Item ID`, `Old Price`, `New Price`, `Stage`, `Reason`, `Tag`, `Floor Price`

**State sheet (appendOrUpdate):** Updated after every price change. Matched on `Variant ID`.
- Columns: `Variant ID`, `Variant Nr`, `Item ID`, `Tag ID`, `Current Stage`, `Price ID 3`, `Current Price 17`, `EK`, `Floor Price`, `Last Price Change`, `Last Sale Date`, `Days at Floor`, `Days No Sale`

**Errors sheet (append):** Written when any price update HTTP request fails.
- Columns: `Timestamp`, `Variant ID`, `Action Type`, `Error Message`

---

## Part B — Full Data Flow Diagram

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
  │                                             → Merge Variants with State (input 0)
  │                        → Merge Variants with State (input 1)
  │                                → Merge1 (input 1)
  └── Merge Variant with soldId (input 1)
          ├── Update Last Sale Date & Days No Sale  [terminal — writes State sheet]
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

## Part C — Business Rules Quick Reference

| Scenario | Route | Price Change |
|----------|-------|-------------|
| Variant sold today | sale_increase | +2 EUR (max = RRP) |
| At floor 7+ days, no sale | reset | Restore to RRP, add Tag 15 |
| Never had Price 17, no sale 3d | stage1_init | RRP minus 1 EUR (min floor), set UUID |
| Has Price 17, no change 3d | stage2_reduction | minus 1 EUR (min floor) |
| Anything else | skip | No change |

**Floor price formulas:**
- Tag 16: `((EK + 3.84) / 0.81 / 0.89) + 8.96`
- Tag 17: `((EK + 0.84) / 0.81 / 0.89) + 8.96`

**Price sentinel values:**
- `price17 = 0` means "not set" — treated as `null` throughout
- `daysSinceLastSale = 999` when `Last Sale Date` is empty — ensures stage1_init triggers on first run
- `daysSinceLastChange = 999` when `Last Price Change` is empty

---

## Part D — Known Data Quirks

1. `variationSalesPrices3` and `variationSalesPrices17` are **nested objects** (not dot-notation strings)
2. `floorPrice16` / `floorPrice17` are **strings** from the Calc nodes — use `parseFloat()`
3. `price17 = 0` means **not set** — treated as `null` throughout the workflow
4. `Last Price Change` can be an **empty string** if never changed — treated as null → `daysSinceLastChange = 999`
5. `variantId` (camelCase) and `Variant ID` (title case) both exist after the merge — same value, different keys
6. Two stock filters exist: `Filter` (before state lookup) and `Filter Stock > 0` (after enrichment, before Decision Router)

---

## Part E — Google Sheets Reference

**Sheet ID:** `[GOOGLE_SHEET_ID]`
**Credential ID:** `[GOOGLE_SHEETS_CREDENTIAL_ID]`
**State sheet gid:** `[STATE_SHEET_GID]`

| Tab | Purpose | Write operation |
|-----|---------|----------------|
| State | One row per variant — tracks current stage, prices, dates | `appendOrUpdate` on `Variant ID` |
| Price Change Log | Append-only audit log of every price change | `append` |
| Errors | Append-only log of API failures | `append` |

**State sheet column names (exact):**
`Variant ID`, `Variant Nr`, `Item ID`, `Tag ID`, `Current Stage`, `Price ID 3`, `Current Price 17`, `EK`, `Floor Price`, `Last Price Change`, `Last Sale Date`, `Days at Floor`, `Days No Sale`

---

## Part F — API Reference

**Base URL:** `{PLENTY_API_URL}`

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/rest/login` | POST | Get access token (`username` field, not `email`) |
| `/rest/items/variations?variationTagId=16&with=variationSalesPrices,Stock` | GET | Fetch Tag 16 variants |
| `/rest/items/variations?variationTagId=17&with=variationSalesPrices,Stock` | GET | Fetch Tag 17 variants |
| `/rest/orders?createdAtFrom=...&typeId=1&itemsPerPage=250` | GET | Fetch recent orders |
| `/rest/items/{itemId}/variations/{variantId}/variation_sales_prices/17` | PUT | Update price 17 |
| `/rest/items/{itemId}/variations/{variantId}/variation_sales_prices/17` | POST | Create price 17 (stage1 only) |
| `/rest/items/{itemId}/variations/{variantId}/variation_tags` | POST | Add Tag 15 on reset |
| `/rest/items/{itemId}/variations/{variantId}` | PUT | Set priceCalculationUuid |

**Auth header:** `Bearer {{ $('Fetch Access Token').item.json.accessToken }}`
**Date format for orders:** W3C with timezone offset e.g. `2026-03-25T00:00:00+01:00`
**priceCalculationUuid:** `[PRICE_CALCULATION_UUID]`
