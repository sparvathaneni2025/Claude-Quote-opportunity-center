# Infinite Electronics — Quote Opportunity Center
## Build Specification / Prompt (v1, reflects the actual built-and-verified app)

Use this document as a prompt to Claude (or a spec for any developer) to reconstruct, refresh, or continue building this dashboard. Everything in here reflects what was actually verified against live Business Central and SalesModel data — not assumptions. Where something is a known limitation rather than a fact, it's labeled as such.

---

## 1. Purpose

An internal tool for T-Jay Taylor's inside sales team (Adam Purcell, Kelli Jones, Jerik Wong, Spencer Doty) that:
- Tracks active $5,000+ quotes across the team's assigned account list, with daily follow-up flags
- Classifies quote lines as Existing Business / Winback / New Business based on real order history
- Tracks real Won conversion (not just BC's quote status field, which is unreliable)
- Surfaces data-quality problems in Business Central rather than hiding them
- Gives reps and the manager filterable, drillable views instead of a static export

**Non-goal:** this is not a replacement for Business Central. It's read-mostly (pulls from BC/SalesModel), with a thin layer of rep-editable fields (Quote Type, Target Price, Comments) that persist locally in the app, not written back to BC.

---

## 2. Architecture

Single self-contained HTML file (`quote_opportunity_center.html`), rendered as a Claude artifact:
- All data is embedded as JavaScript arrays (`QUOTES`, `ARCHIVE`, `LEGACY_WON`, `ALL_ACCOUNTS`) at build time — this is a **snapshot**, not a live-query app. Refreshing means re-running the data pulls in Section 4 and regenerating these arrays.
- Rep-editable fields (Quote Type, Target Price, Comments) persist via the artifact's `window.storage` API, keyed by `quoteNo|lineNo`.
- No external dependencies. Vanilla JS, inline CSS. Modal-based drill-down (quote detail, customer detail, archive detail) rather than page navigation.
- Branding: warm off-white background (`#FAF7F2`), dark charcoal nav (`#2B2B2B`), burnt-orange accent (`#C1541C`), cream cards (`#FFF9F2`), rounded corners.

---

## 3. Data sources

### 3a. Business Central Explorer (MCP connector, OData-based)
Company instance: `azr-bc03.infinite.local`. Relevant entities and the exact fields confirmed to exist and work:

| Entity | Key fields used | Notes |
|---|---|---|
| `salesQuotes` | `number`, `sellToCustomerNo`, `sellToCustomerName`, `status`, `amount`, `followUpDate`, `followUpCode`, `followUpSalespersonCode`, `dueDate`, `expirationDate`, `salespersonCode`, `responsibilityCenter`, `reasonCode`, `lostCode` | `amount` is the reliable quote-total field. `status` values seen: Quote Issued, Quote WIP, Open, Pending Approval, Released, Converted to Order. **No "Lost" or "Cancelled" status value exists in practice** — see Section 5. |
| `salesQuoteLines` | `documentNo`, `lineNo`, `type`, `number`, `description`, `quantity`, `unitPrice`, `lineAmount`, `inventory` | `inventory` = current stock. Exclude lines where `type` is blank and `number` is `NCNR` (boilerplate NCNR/freight-notice lines, $0). `type = 'Resource'`/`number = 'FREIGHT'` lines are real freight charges — the dashboard **excludes these from line display** per explicit request, though they remain part of the quote's `amount` total (shows as an unitemized gap). |
| `salesHeader2` | `probabilityPercent`, `Applications`, `Timeframe`, `Industry`, `salesCompetition`, `manufacturingCompetition`, `projectName`, `closingDate`, `programAccount` | The Opportunity block. **"Describe the Opportunity" does NOT exist on this entity** — confirmed absent, don't invent it. |
| `salesOrders` | `no`, `quoteNo`, `sellToCustomerNo`, `status`, `amount`, `orderDate`, `shipped` | `quoteNo` is a direct link back to the originating quote — **more reliable than the quote's own status for detecting a real conversion.** `shipped` (boolean) tells you if it's fulfilled yet. |
| `invoices` | `no`, `quoteNo`, `orderNo`, `sellToCustomerNo`, `sellToCustomerName`, `documentDate`, `postingDate`, `responsibilityCenter`, `salespersonCode` | Also has a direct `quoteNo` field, independent of quote status. This is the **strongest signal** that a quote actually became real revenue. |
| `postedSalesInvoices` | `no`, `amount`, `orderDate`, `documentDate` | Use to get dollar amounts for invoices found via the `invoices` entity's `quoteNo` field (the `invoices` entity itself doesn't expose a simple total `amount`). |
| `salesInvoiceLine` | `documentNo`, `lineNo`, `no`, `description`, `quantity`, `unitPrice`, `lineAmount` | Only needed if you want line-level detail on Won/invoiced business. Not used for the legacy Won bulk pull (used header-level `postedSalesInvoices.amount` instead, for speed). |

**Critical connector limitation:** every `getGenericEntity` query caps at **2,000 rows**, no `@odata.nextLink` pagination available. A broad, unfiltered pull will silently miss data older or "further down" than whatever the default sort returns. **Always filter by `responsibilityCenter eq 'LCOM'` at minimum, and prefer filtering directly by customer number (`sellToCustomerNo eq 'X' or sellToCustomerNo eq 'Y' ...`, chunked ~25–40 per call) or by `amount ge 5000` over a broad unfiltered pull.** This was the root cause of an entire rep's active pipeline being invisible in an earlier build pass — see Section 6.

`startswith()` and `eq` filters work; **`not` filters are rejected outright** by this BC OData implementation (`BadRequest_MethodNotImplemented`). Structure filters to avoid needing `not`.

### 3b. SalesModel (PowerBI MCP, semantic model)
GUID: `bce6ee25-e19f-4b95-acdd-01ed7d1e666a`. Used for:
- `d_SellToCustomer`: `CustomerNumber`, `CustomerName`, `LastOrderDate`, `DaysSinceLastOrder`, `FirstOrderDate` — customer-level classification (Existing/Winback/New Business fallback).
- `f_Transaction`: `Sell-ToCustomerKey` (join to `d_SellToCustomer[CustomerKey]`), `ItemKey` (join to `d_Item[ItemKey]`), `OrderDate` — used for item-level classification (which specific part, not just which customer).

**Known bug, not yet resolved:** `SUMMARIZECOLUMNS` grouped by `f_Transaction[ItemKey]` while also trying to `CALCULATE(SELECTEDVALUE(d_Item[ItemNumber]))` in the same query returns `null` for every `ItemNumber` — the join breaks under that specific query shape. Item-level classification that works: `FILTER` + `RELATED()` inside a `SELECTCOLUMNS`, filtering explicitly by a list of item numbers (not grouping across many at once). This only scales to a few dozen items per query, not hundreds.

**Known data quality issue in SalesModel itself:** customer numbers are not always unique — e.g., `C132371` resolves to both "Epirus" (real, active) and an unrelated dormant record "Environmental Dimensions Inc" (last order 2017). **Always verify the customer name matches what's expected, don't trust the number alone.**

### 3c. Account list
Source: uploaded `Pro_active_Sales_Accounts_Effective_July_2026.xlsx`. Columns: Owner, OMID Name, Account Name, Account Number, Brand. 133 real rows (rest of the 284-row sheet is blank padding — filter to rows with a non-null Owner). All 133 are Brand = "LC" (L-com), which maps to `responsibilityCenter = 'LCOM'` in BC.

---

## 4. How the data was actually pulled (repeat this to refresh)

1. **Load the account list**, get `custNo → {name, owner}` for all 133 accounts.
2. **Pull active quotes per rep, not broadly.** For each rep's account list, query:
   `salesQuotes?$filter=(sellToCustomerNo eq 'X' or ...) and responsibilityCenter eq 'LCOM' and amount ge 5000&$select=number,sellToCustomerNo,sellToCustomerName,status,amount,followUpDate,followUpCode,followUpSalespersonCode,expirationDate,dueDate,salespersonCode,reasonCode,lostCode&$top=2000`
   Do this **once per rep** (4 calls), not one broad company-wide pull — the 2,000-row cap will silently drop a large chunk of one or more reps' books otherwise.
3. **Exclude `status = 'Converted to Order'`** from the active set (that's Won, goes to Archive) — everything else (Quote Issued, Quote WIP, Open, Pending Approval, Released) counts as active.
4. **Pull lines** for each matched quote number via `salesQuoteLines?$filter=documentNo eq 'X' or ...` chunked ~25 quote numbers per call. Exclude blank-type/NCNR lines and Resource/FREIGHT lines from what's displayed (note the value gap on the quote instead of silently dropping it).
5. **Classify each quote/line**: pull `d_SellToCustomer` for the matched customer numbers (customer-level fallback), and where feasible pull item-level history via SalesModel `f_Transaction` filtered to specific (customer, item) pairs found on the quotes. Classification rule: ≤365 days since last order = Existing Business; 366–730 days = Winback; no history or >730 days = New Business.
6. **Pull Won quotes**: `salesQuotes?$filter=status eq 'Converted to Order'&$top=2000` (no customer filter — this table is small enough company-wide, ~720 rows, well under the cap), then cross-reference against the account list. Cross-check via `salesOrders?$filter=quoteNo eq '...'` and `invoices?$filter=quoteNo eq '...'` — both should agree with the status field for genuine wins. Track `orderExists`, `shipped`, `invoiced` per Won record for the funnel-stage display, not just a flat "Won" label.
7. **Pull legacy pre-upgrade Won business**: `invoices?$filter=responsibilityCenter eq 'LCOM' and startswith(quoteNo,'SQ')&$select=no,quoteNo,orderNo,sellToCustomerNo,sellToCustomerName,documentDate,postingDate&$top=2000`, both `$orderby=documentDate asc` and `desc` to cover more of the date range (still capped at 2,000 each), filter client-side for `quoteNo` NOT starting with `SQIE` (the `not` filter doesn't work server-side, do it in code) and customer number in the account list. Get dollar amounts via `postedSalesInvoices?$filter=no eq 'X' or ...` chunked ~40 invoice numbers per call.
8. **Get total revenue for a fair comparison period** via SalesModel: `EVALUATE ROW("Total", CALCULATE([Net Sales $], FILTER(VALUES(d_SellToCustomer[CustomerNumber]), d_SellToCustomer[CustomerNumber] IN {...}), d_OrderDate[CalendarDate] >= DATE(year,month,day)))` — **always bound the date range to match the period the quote-numbering scheme you're comparing against has actually existed for.** Comparing Won-quote value against all-time revenue when the quote system is only a few months old is a real mistake made and corrected during this build — don't repeat it.

---

## 5. Real findings baked into this build (don't re-litigate, but do keep re-verifying as things change)

- **Lost quotes are never formally closed.** Every $5,000+ quote checked with a `lostCode` set still shows `status = 'Quote Issued'`, `'Quote WIP'`, or `'Pending Approval'` — never closed. Re-verified on the September 2026 refresh: 29 of 29 (100%) active lost-coded quotes for the team's accounts also have `reasonCode` blank. There is no reliable way to compute a Lost bucket from current BC data. This needs a **process fix** (reps/BC admin actually closing quotes, and populating reasonCode alongside lostCode), not a data fix. A separate one-page instructional doc for the team covers this. The Lost Quotes view (Section 7, item 11) surfaces this list directly.
- **`status = 'Converted to Order'` means an order was created, not that it shipped or invoiced.** Cross-checking against `salesOrders.shipped` and `invoices.quoteNo` showed 2 of 6 confirmed Won quotes (as of the latest refresh) hadn't shipped/invoiced yet. Track funnel stage (Order Created → Shipped/Invoiced), not a flat Won/Lost binary.
- **BC was recently upgraded**, changing quote numbering from an old "SQ" prefix to the current "SQIE" prefix (same business, same accounts, ~April 2026 cutover). A large, ongoing volume of real revenue is legacy pre-upgrade business finishing its lifecycle against old "SQ" quote numbers — this is why the current system's own Won total looks tiny in isolation (compare Archive's Current vs. Legacy totals at refresh time). Keep these two populations visually distinct (Archive's "System" column: Current vs. Legacy) rather than merging them into one number.
- **The 2,000-row query cap is the single most dangerous failure mode in this build.** It silently drops data rather than erroring, and it looks identical to "there's genuinely nothing there." Always filter tightly (by account list, by status, by amount threshold) rather than pulling broadly and filtering client-side.
- **Customer-number collisions in SalesModel keep surfacing on refresh, not just the original two.** Confirmed again: `C132371` (Epirus vs. a dormant "Environmental Dimensions Inc") and `C129074` (Anduril Industries vs. a dormant "Hope College"). Newly found on a later refresh: `C151561` (Pano AI vs. a dormant "Phantom Intelligence") and `C105468` (Coast Pneumatics Inc vs. an unrelated dormant "Kfmb Tv"). Always filter SalesModel matches by name, never trust the customer number alone.
- **Some real, currently-active accounts show zero SalesModel order history under their current Sell-to Number** — seen on a later refresh for Marshall Electronics Inc (L100201), Coast Pneumatics Inc (C105468), 4T Manufacturing (CIE0003447), and Mojave Advanced Solutions Inc. (CIE0002252). This is exactly the Legacy Customer Number remap risk called out in Section 3b/3c — none of these have been joined on Legacy Customer Number yet, so their New Business classification should be treated as unverified rather than confirmed until that fallback join is implemented.

---

## 6. Data model (JS arrays embedded in the artifact)

```js
QUOTES = [{
  quoteNo, customer, custNo, owner, bcSalesperson, status,
  followUpDate, followUpCode, followUpSalesperson, dueDate, expirationDate,
  quoteTotal, lastOrderDate, daysSinceLastOrder, classification, // customer-level classification, fallback
  moreLinesValue, // $ gap between quoteTotal and sum of displayed lines (freight, excluded lines, truncation)
  lostCode, reasonCode, // straight from salesHeader/salesQuotes — feeds the Lost Quotes view (Section 7, item 11)
  lines: [{ lineNo, item, desc, qty, unitPrice, lineAmount, stock,
            itemClassification, itemDaysSince, itemLastOrderDate }] // item-level classification where available
}]

ARCHIVE = [{ ...same shape as QUOTES entries, plus: outcome:"Won", orderNo, orderExists, shipped, invoiced }]

LEGACY_WON = [{ quoteNo, invoiceNo, customer, custNo, owner, amount, documentDate }] // no line detail, header-level only

ALL_ACCOUNTS = [{ custNo, name, owner }] // full 133-account roster, used to compute zero-activity accounts
```

---

## 7. Views built (tabs)

1. **My Quote Queue** — all active $5K+ lines, team-wide. Filterable by Quote Type (Bid/Buy/Not set), Classification, Customer. Columns include Stock (flagged red at zero). Flags: Overdue, Due Today, Missing Follow-Up (date/code/salesperson), Expiring ≤5 days, Owner≠BC Salesperson, Missing Bid/Buy.
2. **Rep View** — same table, scoped to one rep via a dropdown, own KPIs.
3. **Winback** — lines where `itemClassification === 'Winback'`. Surfaces same-part-quoted-twice patterns (e.g., a specific part quoted on 2 separate active quotes while also being 12–24 months stale).
4. **Repeated No-Win** — customer+part quoted 2+ times (within the *visible active set only* — a real limitation, needs full historical quote log to do properly) and never purchased.
5. **Conversion** — Active + Archive combined, date-range filterable (by Due Date), rep leaderboard (quoted value, Won value, win rate), conversion by Quote Type, conversion by Classification.
6. **Manager View** — team rollup (value, overdue, missing follow-up, zero-activity accounts per rep) + exception-based coaching queue (every flagged line, ranked by severity then value, not just top-N by dollar).
7. **Customers** — high-activity (top 20%) / low-activity (bottom 20%, has ≥1 quote) / **zero-activity (no quotes at all — a distinct, more important category)** / full sortable-by-conversion table.
8. **Data Quality** — plain-language list of every real data problem found, with the debugging evidence, not just a symptom.
9. **Archive** — Won records, Current + Legacy unified, date-range filterable, funnel-stage badges, drill-down (Current only — Legacy records don't have line-level detail pulled).
10. **Roadmap** — what's built vs. genuinely still open.
11. **Lost Quotes** — every active $5K+ quote where `lostCode` is populated (pulled per-rep, same method as Section 4), status included as-is (still shows Quote Issued/WIP — a known BC limitation, not a bug here). Columns: quote number, customer, owner, quote total, lost code, reason code, status. Rows where `lostCode` is set but `reasonCode` is blank (or vice versa) are flagged and sorted first, then by quote total descending.

Every quote number and customer name in every table is clickable → opens a modal (quote detail with all lines + QUOTE/VALUE coaching questions generated from facts already on that quote, or customer detail listing all their quotes, with a back-link between the two).

**Interactive filters (My Quote Queue, Rep View, Manager View, Lost Quotes):** the KPI/summary count cards and the inline flag badges are clickable filters — clicking one filters the table/list to matching rows, clicking the same one again clears it. Only one quick filter is active at a time per view (v1). A "Showing: `<label>` (`<count>`)" indicator with a clear/× button appears whenever a quick filter is active. This is in addition to the existing dropdown filters (Quote Type, Classification, Customer) on Queue/Rep View, which combine with a quick filter via AND.

---

## 8. QUOTE+VALUE coaching panel

Generated per-quote from data already present on that specific quote — **never invent customer-specific facts not in the pull.** Structure: Q (Bid/Buy confirmed? application/project?), U (how will they decide? stock status), O (highest-value line, unitemized gap), T (value story beyond price), E (follow-up status, has outcome been documented). Phrase everything as a question to confirm, not a stated fact.

---

## 9. Still open / roadmap for whoever continues this

- **Repeated No-Win** only sees the current active-quote snapshot. Needs a full historical quote log (not just currently-open quotes) to properly detect "quoted repeatedly, never won" patterns.
- **Item-level classification** (the precise version, not the customer-level fallback) is not currently computed at all — as of the September 2026 refresh every line uses the customer-level classification. An earlier build pass had ~28 quotes with true item-level precision; that coverage was not carried forward or recomputed on this refresh (traded off to keep an automated refresh within a reasonable run time). The SalesModel `ItemKey`/`ItemNumber` join bug (Section 3b) needs a real fix — likely filtering item lists in smaller batches — before item-level precision is worth re-extending to any quotes.
- **Legacy Customer Number fallback join** (Section 3b/3c) still isn't implemented — 4 active accounts currently show no SalesModel history under their Sell-to Number as a result (see Section 5). Needs `d_SellToCustomer`'s Legacy Customer Number field pulled alongside Sell-to Number and matched as a fallback.
- **Real Lost tracking** is a process problem, not a tool problem — see the separate instructional doc and the leadership proposal doc for Craig/Joanna/Jonathan. The Lost Quotes view (Section 7, item 11) makes the current gap visible but can't close it.
- **Quick filters (Section 7's interactive-filter note) are single-select, v1 only.** Combining more than one quick filter at a time, or giving Lost Quotes/Manager View the same dropdown-filter treatment Queue/Rep View have, is open.
- Nothing in this app writes back to BC. If that's ever wanted (e.g., a "close this quote" button), that's a meaningfully bigger scope change — confirm real business appetite for it before building.

---

## 10. Prompt to hand to Claude to continue this

> "Using the attached account list and the Business Central Explorer / PowerBI MCP connectors, refresh the Quote Opportunity Center dashboard following the exact data-pull methodology in Section 4 of this spec. Pull per-rep, not broadly, to avoid the 2,000-row cap silently dropping data. Preserve the existing app structure, styling, and views (Section 7) exactly — just refresh the embedded QUOTES/ARCHIVE/LEGACY_WON/ALL_ACCOUNTS arrays with current data, re-verify the findings in Section 5 still hold, and flag anything new the same way — plainly, in Data Quality, with the debugging evidence shown, not just the conclusion."
