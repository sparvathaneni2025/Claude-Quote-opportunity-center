# Infinite Electronics — Quote Opportunity Center
## Build Specification / Prompt (v2 — refreshed 2026-10-03; reflects the actual built-and-verified app)

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
| `salesQuotes` | `number`, `sellToCustomerNo`, `sellToCustomerName`, `status`, `amount`, `followUpDate`, `followUpCode`, `followUpSalespersonCode`, `dueDate`, `expirationDate`, `salespersonCode`, `responsibilityCenter`, `reasonCode`, `lostCode` | `amount` is the reliable quote-total field. `status` values seen (2026-10-03): Quote Issued, Quote WIP, Pending Approval, Converted to Order. **No "Lost" or "Cancelled" status value exists in practice** — see Section 5. **This entity holds open quotes only.** Once a quote converts it is removed from `salesQuotes` — of 1,041 SQIE quotes that produced an invoice, only 4 were still present. Blank dates come back as `0001-01-01`, not null. `reasonCode` came back empty on all 736 quotes pulled. |
| `salesQuoteLines` | `documentNo`, `lineNo`, `type`, `number`, `description`, `quantity`, `unitPrice`, `lineAmount`, `inventory` | `inventory` = current stock. Exclude lines where `type` is blank and `number` is `NCNR` (boilerplate NCNR/freight-notice lines, $0). `type = 'Resource'`/`number = 'FREIGHT'` lines are real freight charges — the dashboard **excludes these from line display** per explicit request, though they remain part of the quote's `amount` total (shows as an unitemized gap). |
| `salesHeader2` | `probabilityPercent`, `Applications`, `Timeframe`, `Industry`, `salesCompetition`, `manufacturingCompetition`, `projectName`, `closingDate`, `programAccount` | The Opportunity block. **"Describe the Opportunity" does NOT exist on this entity** — confirmed absent, don't invent it. |
| `salesOrders` | `no`, `quoteNo`, `sellToCustomerNo`, `status`, `amount`, `orderDate`, `shipped` | `quoteNo` is a direct link back to the originating quote — **more reliable than the quote's own status for detecting a real conversion.** `shipped` (boolean) tells you if it's fulfilled yet. |
| `invoices` | `no`, `quoteNo`, `orderNo`, `sellToCustomerNo`, `sellToCustomerName`, `documentDate`, `postingDate`, `responsibilityCenter`, `salespersonCode`, **`prodSalesAmount`** | Also has a direct `quoteNo` field, independent of quote status. This is the **strongest signal** — in fact the only reliable one — that a quote actually became real revenue. **`prodSalesAmount` carries the invoice total**, so the separate `postedSalesInvoices` round-trip below is unnecessary; pull the amount here in the same call. Supports `documentDate ge YYYY-MM-DD` filters. |
| `postedSalesInvoices` | `no`, `amount`, `orderDate`, `documentDate` | **Superseded — use `invoices.prodSalesAmount` instead.** This entity *requires* a `no` filter (errors with "Sales Invoice Header No. filter is required" otherwise), has no `sellToCustomerNo` or `responsibilityCenter` property, 404s on filters much past ~4KB, and times out at 60s on ~110 OR terms. Not worth the round-trip. |
| `salesInvoiceLine` | `documentNo`, `lineNo`, `no`, `description`, `quantity`, `unitPrice`, `lineAmount` | Only needed if you want line-level detail on Won/invoiced business. Not used for the legacy Won bulk pull (used header-level `postedSalesInvoices.amount` instead, for speed). |

**Critical connector limitation:** every `getGenericEntity` query caps at **2,500 rows** (measured 2026-10-03 — the earlier "2,000" figure was an underestimate), no `@odata.nextLink` pagination available. Demonstrated directly: a broad `invoices` pull run `$orderby=documentDate asc` and `desc` returned exactly 2,500 rows each with **zero overlap**, together covering 2023-01-03 to 2023-02-17 and 2026-08-13 to 2026-10-02 and silently omitting everything between. `$top` is **ignored** — do not rely on it to limit a result. A broad, unfiltered pull will silently miss data older or "further down" than whatever the default sort returns. **Always filter by `responsibilityCenter eq 'LCOM'` at minimum, and prefer filtering directly by customer number (`sellToCustomerNo eq 'X' or sellToCustomerNo eq 'Y' ...`, chunked ~25–40 per call) or by `amount ge 5000` over a broad unfiltered pull.** This was the root cause of an entire rep's active pipeline being invisible in an earlier build pass — see Section 6.

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
2. **Pull quotes per rep, not broadly.** For each rep's account list, query:
   `salesQuotes?$filter=responsibilityCenter eq 'LCOM' and (sellToCustomerNo eq 'X' or ...)&$select=number,sellToCustomerNo,sellToCustomerName,status,amount,followUpDate,followUpCode,followUpSalespersonCode,expirationDate,dueDate,salespersonCode,reasonCode,lostCode`
   Do this **once per rep** (4 calls), not one broad company-wide pull — the row cap will silently drop a large chunk of one or more reps' books otherwise. **Drop the `amount ge 5000` filter from the query** and apply the $5K gate client-side: the per-rep row counts come back far under the cap (2026-10-03: Adam 219, Jerik 194, Kelli 179, Spencer 144 — 736 total), and the unfiltered pull gives you the Lost Quotes view and the Won set from the same four calls instead of three separate passes. **Always print the per-rep row count and assert it is under 2,500** — that assertion is the only thing standing between you and silent truncation.
3. **Exclude `status = 'Converted to Order'`** from the active set (that's Won, goes to Archive) — everything else (Quote Issued, Quote WIP, Open, Pending Approval, Released) counts as active.
4. **Pull lines** for each matched quote number via `salesQuoteLines?$filter=documentNo eq 'X' or ...` chunked ~25 quote numbers per call. Exclude blank-type/NCNR lines and Resource/FREIGHT lines from what's displayed (note the value gap on the quote instead of silently dropping it).
5. **Classify each quote/line**: pull `d_SellToCustomer` for the matched customer numbers (customer-level fallback), and where feasible pull item-level history via SalesModel `f_Transaction` filtered to specific (customer, item) pairs found on the quotes. Classification rule: ≤365 days since last order = Existing Business; 366–730 days = Winback; no history or >730 days = New Business.
6. **Pull Won quotes**: `salesQuotes?$filter=status eq 'Converted to Order'&$top=2000` (no customer filter — this table is small enough company-wide, ~720 rows, well under the cap), then cross-reference against the account list. Cross-check via `salesOrders?$filter=quoteNo eq '...'` and `invoices?$filter=quoteNo eq '...'` — both should agree with the status field for genuine wins. Track `orderExists`, `shipped`, `invoiced` per Won record for the funnel-stage display, not just a flat "Won" label.
7. **Pull legacy pre-upgrade Won business — per rep and date-bounded, not broadly.** The `$orderby asc`/`desc` trick described in v1 does **not** work: it leaves a three-year hole in the middle (see Section 3a). Instead run one call per rep:
   `invoices?$filter=responsibilityCenter eq 'LCOM' and startswith(quoteNo,'SQ') and documentDate ge 2026-01-01 and (sellToCustomerNo eq 'X' or ...)&$select=no,quoteNo,orderNo,sellToCustomerNo,sellToCustomerName,documentDate,postingDate,prodSalesAmount,salespersonCode`
   Then filter client-side for `quoteNo` NOT starting with `SQIE` (the `not` filter doesn't work server-side, do it in code). `prodSalesAmount` gives you the dollar value directly — no `postedSalesInvoices` round-trip. The same four calls also hand you every **SQIE**-quoted invoice, which is what you need for real conversion (see Section 5).
8. **Get total revenue for a fair comparison period** via SalesModel: `EVALUATE ROW("Total", CALCULATE([Net Sales $], FILTER(VALUES(d_SellToCustomer[CustomerNumber]), d_SellToCustomer[CustomerNumber] IN {...}), d_OrderDate[CalendarDate] >= DATE(year,month,day)))` — **always bound the date range to match the period the quote-numbering scheme you're comparing against has actually existed for.** Comparing Won-quote value against all-time revenue when the quote system is only a few months old is a real mistake made and corrected during this build — don't repeat it.

---

## 5. Real findings baked into this build (don't re-litigate, but do keep re-verifying as things change)

- **Lost quotes are never formally closed. STILL TRUE, now measured across the whole roster (2026-10-03).** All 130 quotes on the team's 133 accounts carrying a `lostCode` still show `status = 'Quote Issued'` (103), `'Quote WIP'` (26) or `'Pending Approval'` (1). Cross-checked the other way: **zero** of those 130 produced an invoice, so the lost codes are accurate — the business really is dead, only the closing step is missing. Needs a **process fix**, not a data fix.
- **NEW: `reasonCode` is populated on zero quotes.** All 130 lost-coded quotes have `lostCode` set and `reasonCode` blank; across all 736 quotes pulled, `reasonCode` is never populated. Check with the BC admin whether the field is even wired into the entry screen before asking reps to fill it in.
- **CHANGED: `status = 'Converted to Order'` is not a Won archive at all.** v1 read this as "order created, not yet shipped". The deeper truth: BC **removes a quote from `salesQuotes` once it converts**. 1,041 distinct SQIE quote numbers appear as `invoices.quoteNo` for these accounts since April 2026, worth **$3,203,985** — and only **4** of those 1,041 are still in `salesQuotes`. So the handful carrying "Converted to Order" are stragglers, not the win population. **v1's headline "$30,609.74 Won vs. $9.34M revenue — 0.33% conversion" was a measurement artifact and should not be repeated.** Counting what is visible: 1,041 converted vs. 736 still open ≈ 59% of SQIE quotes have produced invoiced revenue. **Compute Won from `invoices.quoteNo`, never from quote status.**
- **CHANGED: absence from `salesOrders` does not mean "not shipped".** SQIE0015839 and SQIE0025898 returned nothing from `salesOrders` but are fully invoiced (IESIN1076443, IESIN1066590) — completed orders drop out of that entity. Treat an invoice as proof of fulfilment; reading `salesOrders.shipped` alone understates it. Also, quote-to-order is not one-to-one: SQIE0021139 produced two orders (SOIE0029271, SOIE0029272).
- **NEW: quantity-break tiers inflate the pipeline.** Ten quotes price the same part at several quantity breaks on separate lines, and BC's quote `amount` sums all of them. SQIE0029470 (Anduril) quotes one part at 5k/10k/15k/20k/25k units for a $2,519,335 headline, of which $1,694,420 is the same part counted again. Across the ten: **$1,828,025 overstated**, i.e. a $7.42M headline pipeline is realistically ~$5.6M. Don't drop the lines — they're genuine — but net the tiers out before quoting a pipeline number to leadership.
- **NEW: not one active $5K+ quote has a follow-up date in the future.** 83 of 132 overdue (median 46 days), 49 with no date at all, 0 due today; the latest follow-up date anywhere in the book is already past. Separately, 95 of 132 are past their own expiration date and still open.
- **The SQ/SQIE legacy split is real, and the legacy tail is now clearly winding down.** 1,349 legacy invoices for these accounts, worth **$4,878,739**, posted 2026-01-02 to 2026-10-01 against old "SQ" quote numbers. The monthly shape is the story: Jan 256 invoices, Feb 283, Mar 280, Apr 274 — then May 103, Jun 70, Jul 40, Aug 20, Sep 22, Oct 1. Over the same window SQIE-quoted invoices run the opposite way: Apr 97, May 241, Jun 289, Jul 274, Aug 268, Sep 287. **v1's framing — that legacy volume was masking a failing new quote process — does not hold.** New-system business has fully taken over; the apparent failure was the status-field artifact above. Keep the two populations visually distinct (Archive's "System" column) anyway.
- **The row cap is the single most dangerous failure mode in this build.** It silently drops data rather than erroring, and it looks identical to "there's genuinely nothing there." Always filter tightly (by account list, by status, by amount threshold) rather than pulling broadly and filtering client-side.

---

## 6. Data model (JS arrays embedded in the artifact)

```js
QUOTES = [{
  quoteNo, customer, custNo, owner, bcSalesperson, status,
  followUpDate, followUpCode, followUpSalesperson, dueDate, expirationDate,
  quoteTotal, lastOrderDate, daysSinceLastOrder, classification, // customer-level classification, fallback
  lostCode, reasonCode, // added 2026-10-03; drives the Lost Quotes view and the "Lost code set" flag/filter
  moreLinesValue, // $ gap between quoteTotal and sum of displayed lines (freight, excluded lines, truncation)
  lines: [{ lineNo, item, desc, qty, unitPrice, lineAmount, stock,
            itemClassification, itemDaysSince, itemLastOrderDate }] // item-level classification where available
}]

ARCHIVE = [{ ...same shape as QUOTES entries, plus: outcome:"Won", orderNo, orderExists, shipped, invoiced }]

LEGACY_WON = [{ quoteNo, invoiceNo, customer, custNo, owner, amount, documentDate }] // no line detail, header-level only

LOST_QUOTES = [{ quoteNo, customer, custNo, owner, quoteTotal, lostCode, reasonCode, status, dueDate }]
  // added 2026-10-03; pre-sorted flagged-first then quoteTotal desc

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
9. **Archive** — Won records, Current + Legacy unified, date-range filterable, funnel-stage badges, drill-down (Current only — Legacy records don't have line-level detail pulled). Note the "Current" count is not the real win count — see Section 5.
10. **Roadmap** — what's built vs. genuinely still open.
11. **Lost Quotes** (added 2026-10-03) — every quote carrying a `lostCode`, pulled per rep by the same method as the active pull (no `amount` gate; these are often below $5K). Columns: quote number, customer, owner, quoteTotal, lostCode, reasonCode, status. **Status will read "Quote Issued"/"Quote WIP" on every row — that is the known BC limitation, expected, and is the point of the view.** Rows where one of `lostCode`/`reasonCode` is set and the other blank are flagged and sorted first, then by quoteTotal descending. Backed by `LOST_QUOTES`, a fifth embedded array alongside QUOTES / ARCHIVE / LEGACY_WON / ALL_ACCOUNTS.

**Interactive filters (added 2026-10-03), on My Quote Queue, Rep View, Manager View and Lost Quotes:** the KPI/summary count tiles and the inline flag badges are clickable filters — clicking filters the table to matching rows, clicking the same one again toggles it off. **One active filter at a time (v1)**, deliberately. While filtered, a "Showing: Overdue (83)" chip appears with a clear/× button. Filter predicates live in one `FLAG_FILTERS` map keyed by the same `key` that `computeFlags` stamps on each badge, so a badge and its matching tile always agree.

Every quote number and customer name in every table is clickable → opens a modal (quote detail with all lines + QUOTE/VALUE coaching questions generated from facts already on that quote, or customer detail listing all their quotes, with a back-link between the two).

---

## 8. QUOTE+VALUE coaching panel

Generated per-quote from data already present on that specific quote — **never invent customer-specific facts not in the pull.** Structure: Q (Bid/Buy confirmed? application/project?), U (how will they decide? stock status), O (highest-value line, unitemized gap), T (value story beyond price), E (follow-up status, has outcome been documented). Phrase everything as a question to confirm, not a stated fact.

---

## 9. Still open / roadmap for whoever continues this

- **Repeated No-Win** only sees the current active-quote snapshot. Needs a full historical quote log (not just currently-open quotes) to properly detect "quoted repeatedly, never won" patterns.
- **PowerBI/SalesModel was not authorised for the 2026-10-03 refresh — this is the top blocker.** Classification, Winback and Repeated No-Win all ran on last-order dates carried forward from the 2026-08-05 pull (days-since recomputed against today; "Data unavailable" shown where there was no prior reading, rather than guessing). The bias runs one way — a customer can look staler than they are, never fresher — so those two tabs are flagged in-app as unsafe to work from until the connector is reconnected. **Any refresh must check PowerBI auth first and say plainly in-app if it is missing.**
- **Conversion and Archive still read the quote status field.** They should be rebuilt on `invoices.quoteNo`, which is reachable through the same connector and already proven to find 1,041 real conversions the status field loses. Needs a new embedded array and reworked views.
- **Quantity-break tiers need a netting rule** before the pipeline total can be quoted directly — agree with the team which tier represents the expected landing.
- **Item-level classification** (the precise version, not the customer-level fallback) only covers the original ~28 quotes pulled early in this build. The SalesModel `ItemKey`/`ItemNumber` join bug (Section 3b) needs a real fix — likely filtering item lists in smaller batches — before extending it to the rest.
- **Legacy Won pull reaches back to 2026-01-01** as of the 2026-10-03 refresh (1,349 invoices). Going further back is a matter of extending the `documentDate ge` bound per rep; 2023 data is reachable but was out of scope for this cycle.
- **Interactive filters are one-at-a-time by design in v1.** Combining them (overdue AND zero stock) and saving per-rep views is the obvious next step.
- **Real Lost tracking** is a process problem, not a tool problem — see the separate instructional doc and the leadership proposal doc for Craig/Joanna/Jonathan.
- Nothing in this app writes back to BC. If that's ever wanted (e.g., a "close this quote" button), that's a meaningfully bigger scope change — confirm real business appetite for it before building.

---

## 10. Prompt to hand to Claude to continue this

> "Using the attached account list and the Business Central Explorer / PowerBI MCP connectors, refresh the Quote Opportunity Center dashboard following the exact data-pull methodology in Section 4 of this spec. Pull per-rep, not broadly, and assert each rep's row count is under the 2,500-row cap — a broad pull drops rows silently. Preserve the existing app structure, styling, and views (Section 7) exactly — just refresh the embedded QUOTES/ARCHIVE/LEGACY_WON/LOST_QUOTES/ALL_ACCOUNTS arrays with current data, merging onto `window.storage` so rep-edited Quote Type / Target Price / Comments are never overwritten. Re-verify the Section 5 findings still hold and flag anything new the same way — plainly, in Data Quality, with the debugging evidence shown, not just the conclusion. Check PowerBI auth before starting; if it is unavailable, say so in-app rather than presenting carried-forward classification as fresh."
