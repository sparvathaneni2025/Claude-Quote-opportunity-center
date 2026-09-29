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

- **Lost quotes are never formally closed.** Every $5,000+ quote checked with a `lostCode` set (DEMANDCHG, PRICING, NO BID, PROJ-LOST, etc. — over 100 checked) still shows `status = 'Quote Issued'` or `'Quote WIP'`. There is no reliable way to compute a Lost bucket from current BC data. This needs a **process fix** (reps/BC admin actually closing quotes), not a data fix. A separate one-page instructional doc for the team covers this.
- **`status = 'Converted to Order'` means an order was created, not that it shipped or invoiced.** Cross-checking against `salesOrders.shipped` and `invoices.quoteNo` showed 2 of 3 confirmed Won quotes hadn't shipped yet. Track funnel stage (Order Created → Shipped/Invoiced), not a flat Won/Lost binary.
- **BC was recently upgraded**, changing quote numbering from an old "SQ" prefix to the current "SQIE" prefix (same business, same accounts, ~April 2026 cutover). A large, ongoing volume of real revenue (**$1,059,243.50** confirmed since April 2026, still posting as of early August) is legacy pre-upgrade business finishing its lifecycle against old "SQ" quote numbers — this is why the current system's own Won total ($30,609.74) looks tiny in isolation. Keep these two populations visually distinct (Archive's "System" column: Current vs. Legacy) rather than merging them into one number.
- **The 2,000-row query cap is the single most dangerous failure mode in this build.** It silently drops data rather than erroring, and it looks identical to "there's genuinely nothing there." Always filter tightly (by account list, by status, by amount threshold) rather than pulling broadly and filtering client-side.

---

## 6. Data model (JS arrays embedded in the artifact)

```js
QUOTES = [{
  quoteNo, customer, custNo, owner, bcSalesperson, status,
  followUpDate, followUpCode, followUpSalesperson, dueDate, expirationDate,
  quoteTotal, lastOrderDate, daysSinceLastOrder, classification, // customer-level classification, fallback
  moreLinesValue, // $ gap between quoteTotal and sum of displayed lines (freight, excluded lines, truncation)
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

Every quote number and customer name in every table is clickable → opens a modal (quote detail with all lines + QUOTE/VALUE coaching questions generated from facts already on that quote, or customer detail listing all their quotes, with a back-link between the two).

---

## 8. QUOTE+VALUE coaching panel

Generated per-quote from data already present on that specific quote — **never invent customer-specific facts not in the pull.** Structure: Q (Bid/Buy confirmed? application/project?), U (how will they decide? stock status), O (highest-value line, unitemized gap), T (value story beyond price), E (follow-up status, has outcome been documented). Phrase everything as a question to confirm, not a stated fact.

---

## 9. Still open / roadmap for whoever continues this

- **Repeated No-Win** only sees the current active-quote snapshot. Needs a full historical quote log (not just currently-open quotes) to properly detect "quoted repeatedly, never won" patterns.
- **Item-level classification** (the precise version, not the customer-level fallback) only covers the original ~28 quotes pulled early in this build. The SalesModel `ItemKey`/`ItemNumber` join bug (Section 3b) needs a real fix — likely filtering item lists in smaller batches — before extending it to the rest.
- **Legacy Won pull only reaches back to ~April 2026.** Going further back needs more `$orderby`/pagination passes against the 2,000-row cap.
- **Real Lost tracking** is a process problem, not a tool problem — see the separate instructional doc and the leadership proposal doc for Craig/Joanna/Jonathan.
- Nothing in this app writes back to BC. If that's ever wanted (e.g., a "close this quote" button), that's a meaningfully bigger scope change — confirm real business appetite for it before building.

---

## 10. Prompt to hand to Claude to continue this

> "Using the attached account list and the Business Central Explorer / PowerBI MCP connectors, refresh the Quote Opportunity Center dashboard following the exact data-pull methodology in Section 4 of this spec. Pull per-rep, not broadly, to avoid the 2,000-row cap silently dropping data. Preserve the existing app structure, styling, and views (Section 7) exactly — just refresh the embedded QUOTES/ARCHIVE/LEGACY_WON/ALL_ACCOUNTS arrays with current data, re-verify the findings in Section 5 still hold, and flag anything new the same way — plainly, in Data Quality, with the debugging evidence shown, not just the conclusion."

---

## 11. Corrections and additions from the 2026-09-29 refresh

This section is appended by a later refresh. **Where it contradicts sections 1–10, this section is right** — the statements below were measured directly against the live connectors, and the ones marked *correction* mean section 3 or 4 was actively misleading, not merely incomplete.

### 11a. Connector behaviour

- **Correction — the row cap is 2,500, not 2,000, and `$top` is ignored.** Section 3a says 2,000. A query sent with `$top=2000` returns **2,500** rows, reproduced on two different sort directions. You cannot use `$top` to bound a result or to probe how close you are to the limit. The practical danger: validating a pull by checking "did I get exactly 2,000 rows?" will pass a truncated 2,500-row result as complete.
- **Correction — the two-sort trick in section 4 step 7 recovers about 4% of the legacy data.** Pulling legacy `SQ` invoices once `$orderby=documentDate asc` and once `desc` yields **472** usable records. Pulling **per account** yields **13,149** — 28× more, reaching back to 2023-01-03 instead of ~April 2026. Each sort direction spends its whole 2,500-row budget on one end of the range (roughly six weeks at the start, seven at the end) and leaves 2023-02 through 2026-08 entirely unpulled. Per-account is the only method that works here, exactly as it is for active quotes.
- `postedSalesInvoices` is the **only** entity carrying invoice `amount` (the `invoices` entity has no amount field). It cannot be browsed — it rejects any query without an explicit `no` filter — and exposes no `quoteNo`, `sellToCustomerNo` or `responsibilityCenter`, so amounts must be fetched invoice number by invoice number. Measured limits, tighter than they first appear: 150 `no eq` terms exceeds a URL length limit outright ("resource has been removed"); **45 terms consistently times out** against the 60s MCP limit, retried three times; 40 worked earlier in the same run but is marginal; **15 terms succeeded first time, every time**. Use 15–20 per call. **`$select` is mandatory in practice** (without it one invoice returns ~302 KB, because it expands invoice lines *and* a base64 PDF, and times out at 60s); and **calls must be serial** — firing four in parallel timed out three of them. Budget ~35 calls per 500 invoices and expect it to be the slowest part of the whole refresh.
- Date literals work unquoted in filters (`documentDate ge 2024-01-01`), which is what makes per-account date-windowed splits possible when one account alone exceeds the cap.
- Still true: `not` filters are rejected outright. Do all negation client-side.

### 11b. SalesModel

- **The `ItemKey`/`ItemNumber` join bug in section 3b is solved.** Don't use `SUMMARIZECOLUMNS` grouped by `f_Transaction[ItemKey]`. Instead drive from an in-query `DATATABLE` of the (customer, item) pairs and use `CALCULATE` with `FILTER(ALL(dim), …)` filter arguments — those evaluate in the outer row context, so no `RELATED()` and no context transition is involved. Ran 60 pairs in 3 batches with no errors. Extending it further is batching work, not research.
- **`f_Transaction` is not purely transactional.** Its `TransactionType` column carries `Goal` and `Budget` planning rows alongside Invoice / CreditMemo / OpenOrder / Archived / ICC DailyBilling. **Any query that sums or counts `f_Transaction` without excluding Goal and Budget mixes plan into actuals.** `Archived` is a judgment call (it includes orders later deleted or cancelled); including it moved the date on 2 of 60 pairs and changed no recency band.
- **`d_Item` holds one row per item number per `SourceCompany`**, so `ItemKey` is a part-plus-company identifier, not a part identifier. Filter order history by `ItemNumber` to sweep in every variant; filtering by `ItemKey` silently sees one company's slice.
- **There is no Legacy Customer Number column** on `d_SellToCustomer`, or anywhere in the SalesModel schema. The only "Legacy" object is a measure called `Legacy Customers`, which is a count. **The two-key join that the pro-active-KPIs methodology calls for cannot be implemented as specified**, so a customer renumbered during the upgrade can still read as lapsed. This needs an answer from whoever owns the semantic model before lapsed/winback logic is trusted for renumbered accounts.
- **The customer-number collision in section 3b is systematic, not anecdotal.** 133 assigned accounts return 155 rows: **22 numbers resolve to two rows each**, always a live `SourceCompany = 'IEONE'` record versus a dormant legacy `'PE'` shell. All 22 are C-prefixed. Disambiguate by preferring `IEONE`, then the most recent `LastOrderDate`. **Do not fall back to name matching** — C193073 and C197030 both carry the name "Saronic Technologies".
- **"Never ordered" is encoded as sentinels, not nulls:** `LastOrderDate`/`FirstOrderDate` = `1/1/1900` and `DaysSinceLastOrder` = `99999`. Anything that averages or charts days-since-order without excluding these is skewed by 99999s. Also present: rows with an empty `CustomerName`, all PE-side shells.
- **An account-level sentinel is not proof a customer never bought.** At least one case has the account reading as never-ordered while item-level history shows a real dated order for that part. Where the two disagree, the per-part fact data was the trustworthy one.

### 11c. Section 5 findings, re-verified 2026-09-29

- **Lost quotes still never formally close.** All 123 quotes with a `lostCode` sit in an open status. *Refinement:* the statuses are Quote Issued (97), Quote WIP (25) **and Pending Approval (1)** — a filter hardcoded to the first two drops a record. No "Lost" or "Cancelled" status value is in use at all.
- **New: `reasonCode` is entirely unpopulated** — blank on all 720 quotes scanned, so this is not two fields drifting apart but one field nobody fills in. Any column keyed on it is structurally empty. Worth confirming with BC admin whether `reasonCode` is even the intended field for this team.
- **`Converted to Order` still does not mean shipped or invoiced.**
- **New: a fully-invoiced order leaves the `salesOrders` entity.** 2 of 6 Won quotes have a posted invoice but no `salesOrders` row; their invoices still carry an `orderNo`. Testing conversion against `salesOrders` alone undercounts, and the error grows as more orders close out. Test order **OR** invoice.
- **The SQ/SQIE legacy split still holds**, and the pre-upgrade backlog is visibly tapering: 274 legacy invoices in 2026-04 down to 19 in 2026-09.

### 11d. Notes on section 4's method

Section 4's per-rep instruction is right and the reason is worth restating: this refresh returned **130 active $5,000+ quotes / $7.09M** versus 89 / ~$4.5M previously, and the largest single call returned 213 rows. Per-rep is what keeps every call far enough from the cap that truncation cannot happen.

Legacy invoice coverage for the post-upgrade window is now complete: **526 invoices, $2,129,166.04**, dated 2026-04-01 onward, versus 272 / $1,059,243.50 in the previous build. The monthly taper (274 in April down to 19 in September) is the number to watch — when it reaches zero, pre-upgrade backlog stops explaining the small current-system Won total.

One gap this refresh could not close: **`ALL_ACCOUNTS` was not re-derived.** The account roster embedded in the app was reused as-is, because the source `Pro_active_Sales_Accounts_Effective_July_2026.xlsx` was not available to the refresh. If T-Jay has changed the account list, this refresh does not reflect it, and every pull is scoped to the stale roster. Supply the current spreadsheet to the next refresh.
