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
| `postedSalesInvoices` | `no`, `amount`, `orderDate`, `documentDate` | Use to get dollar amounts for invoices found via the `invoices` entity's `quoteNo` field (the `invoices` entity itself doesn't expose a simple total `amount`). **⚠️ This entity renders a PDF document blob per returned row (~1.5s/row).** A 40-number chunk *times out* rather than erroring cleanly. **Chunk at ~10 invoice numbers per call**, and never try to price a large invoice population through it — bound the invoice set by `documentDate` first. (Confirmed the hard way on the 2026-10-02 refresh: an attempt to price 13,153 legacy invoices this way ran 20 minutes and returned nothing.) |
| `salesInvoiceLine` | `documentNo`, `lineNo`, `no`, `description`, `quantity`, `unitPrice`, `lineAmount` | Only needed if you want line-level detail on Won/invoiced business. Not used for the legacy Won bulk pull (used header-level `postedSalesInvoices.amount` instead, for speed). |

**Critical connector limitation:** every `getGenericEntity` query caps at **2,000 rows**, no `@odata.nextLink` pagination available. A broad, unfiltered pull will silently miss data older or "further down" than whatever the default sort returns. **Always filter by `responsibilityCenter eq 'LCOM'` at minimum, and prefer filtering directly by customer number (`sellToCustomerNo eq 'X' or sellToCustomerNo eq 'Y' ...`, chunked ~25–40 per call) or by `amount ge 5000` over a broad unfiltered pull.** This was the root cause of an entire rep's active pipeline being invisible in an earlier build pass — see Section 6.

`startswith()` and `eq` filters work; **`not` filters are rejected outright** by this BC OData implementation (`BadRequest_MethodNotImplemented`). Structure filters to avoid needing `not`.

### 3b. SalesModel (PowerBI MCP, semantic model)
GUID: `bce6ee25-e19f-4b95-acdd-01ed7d1e666a`. Used for:
- `d_SellToCustomer`: `CustomerNumber`, `CustomerName`, `LastOrderDate`, `DaysSinceLastOrder`, `FirstOrderDate` — customer-level classification (Existing/Winback/New Business fallback).
- `f_Transaction`: `Sell-ToCustomerKey` (join to `d_SellToCustomer[CustomerKey]`), `ItemKey` (join to `d_Item[ItemKey]`), `OrderDate` — used for item-level classification (which specific part, not just which customer).

**RESOLVED as of the 2026-10-02 refresh — item-level classification now scales.** The old failure was real but was a symptom of the wrong query shape, not a broken join: grouping by `f_Transaction[ItemKey]` while resolving `CALCULATE(SELECTEDVALUE(d_Item[ItemNumber]))` returns `null` for every `ItemNumber`. **Group on the dimension columns instead and aggregate the fact's own date column.** This shape works and is the one to use:

```dax
EVALUATE SUMMARIZECOLUMNS(
  d_SellToCustomer[CustomerNumber],
  d_Item[ItemNumber],
  FILTER(VALUES(d_SellToCustomer[CustomerNumber]), d_SellToCustomer[CustomerNumber] = "L009635"),
  FILTER(VALUES(d_Item[ItemNumber]), d_Item[ItemNumber] IN {"CSMN15MF-10","DPCAMM-2"}),
  "LastOrder", CALCULATE(MAX(f_Transaction[OrderDate]))
)
```

Two traps to avoid:
- **Do NOT** use `MAX(d_OrderDate[CalendarDate])` as the measure. It ignores the fact table, so every row comes back with the date dimension's own maximum (`12/31/2028`) and you get a full customer × item cross join. The measure must read `f_Transaction[OrderDate]`.
- Items with no purchase history for that customer are simply absent from the result — that's correct, record nothing for them.

Run it one customer at a time with that customer's own item list; `ExecuteQuery` accepts 4 queries per call. 61 customers / 367 customer-item pairs completed in 16 calls.

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
7. **Pull legacy pre-upgrade Won business** — *method corrected on the 2026-10-02 refresh; the old `$orderby` asc/desc sweep below was silently truncating and is no longer recommended.* Filter by customer number **and** bound by date, one call per rep:
   `invoices?$filter=(sellToCustomerNo eq 'X' or ...) and responsibilityCenter eq 'LCOM' and startswith(quoteNo,'SQ') and documentDate ge 2026-04-01&$select=no,quoteNo,orderNo,sellToCustomerNo,sellToCustomerName,documentDate,postingDate,salespersonCode&$top=2000`
   Then filter client-side for `quoteNo` NOT starting with `SQIE` (`not` doesn't work server-side, do it in code) and de-duplicate by invoice `no`. Four calls returned 285 / 805 / 345 / 562 rows — comfortably under the cap — yielding **530** post-cutover legacy invoices.
   Get dollar amounts via `postedSalesInvoices?$filter=no eq 'X' or ...` **chunked at ~10 invoice numbers per call** (see the ⚠️ note in §3a — 40 per call times out).
   **Why bound the date:** without `documentDate ge 2026-04-01`, this same per-rep query returns **13,153** SQ-numbered invoices reaching back to **2023-01-03**. The old build's "272 invoices / $1,059,243.50" was not the real population — it was where the 2,000-row cap cut off, mistaken for the whole. The post-cutover window is the population that actually matters (legacy backlog still finishing), and it's the only one that can be priced in reasonable time.
8. **Get total revenue for a fair comparison period** via SalesModel: `EVALUATE ROW("Total", CALCULATE([Net Sales $], FILTER(VALUES(d_SellToCustomer[CustomerNumber]), d_SellToCustomer[CustomerNumber] IN {...}), d_OrderDate[CalendarDate] >= DATE(year,month,day)))` — **always bound the date range to match the period the quote-numbering scheme you're comparing against has actually existed for.** Comparing Won-quote value against all-time revenue when the quote system is only a few months old is a real mistake made and corrected during this build — don't repeat it.

---

## 5. Real findings baked into this build (don't re-litigate, but do keep re-verifying as things change)

*Status of each finding as re-verified on the 2026-10-02 refresh is marked inline.*

- **Lost quotes are never formally closed. ✅ STILL TRUE.** All **127** lost-coded quotes across the team's accounts (no dollar gate) still show `status` of `Quote Issued`, `Quote WIP` or `Pending Approval` — none closed. Confirmed cause: **there is no `Lost` or `Cancelled` status value in this BC instance at all.** Needs a **process fix** (reps/BC admin actually closing quotes), not a data fix.
- **🆕 The `reasonCode` field is dead company-wide.** `salesQuotes?$filter=responsibilityCenter eq 'LCOM' and reasonCode ne ''` returns **zero rows** — no account filter, no dollar threshold. So every lost-coded quote has a blank Reason Code, and the reverse case is currently impossible anywhere in LCOM. `lostCode` is the only loss signal that exists.
- **`status = 'Converted to Order'` means an order was created, not that it shipped or invoiced. ✅ STILL TRUE.** Of 6 Converted-to-Order quotes on these accounts, only 2 have `shipped = true`. Track funnel stage, not a flat Won/Lost binary.
- **🆕 The order cross-check fails in BOTH directions — don't join on `salesOrders` alone.** 2 of those 6 Won quotes have **no row at all** in `salesOrders`, yet **do** have posted invoices: their orders completed and dropped out of the live entity. "No order row" is therefore ambiguous between "never ordered" and "already finished". Judge real conversion from `invoices.quoteNo`.
- **🆕 The Won archive is structurally incomplete.** BC removes a quote from the live `salesQuotes` table once its order completes, so Won history leaks away over time — the company-wide `Converted to Order` count in LCOM is now only ~170 rows, versus ~720 at the previous refresh. Any win rate computed from the live table is **biased low**. Reading true Won history requires the archive entities (`salesQuoteArchives` / `salesHeaderArchives`), which this build still does not pull.
- **BC was recently upgraded**, changing quote numbering from an old "SQ" prefix to the current "SQIE" prefix (same business, same accounts, ~April 2026 cutover). **⚠️ CORRECTED:** the previously recorded "$1,059,243.50 / 272 invoices since April 2026, still happening" was a **2,000-row truncation artifact read as a complete population**. The real figures: **13,153** SQ-numbered invoices for these accounts going back to **2023-01-03**, of which **530** are post-cutover backlog. Crucially the trend is a **steep decline** (Apr 274 → May 103 → Jun 70 → Jul 40 → Aug 20 → Sep 22 → Oct 1 invoices), so the earlier conclusion that this "isn't a one-time transition blip" **no longer holds** — the backlog is nearly cleared. Still keep the two populations visually distinct (Archive's "System" column).
- **🆕 Volume-tier quotes inflate the pipeline total — flag, don't silently correct.** **18** of the 134 active quotes repeat the *same part* across several lines at an *identical unit price* with a *different quantity* each — a "price me these quantities" tier list where the customer buys one tier. BC's header `amount` sums them all. Worst case: `SQIE0029470` (Anduril) sums 5 tiers of one part to **$2,519,335** when the largest single tier is **$824,915**. Across all 18 this overstates the book by **$1,872,500** — ~25% of the reported $7.57M. The dashboard flags these with a "Volume tiers ×N" badge rather than adjusting the number; the rep confirms which tier is live.
- **🆕 SalesModel customer-number collisions are widespread, not anecdotal.** **22 of the 133** assigned accounts return more than one `d_SellToCustomer` record for the same customer number, and in **19** of those the wrong pick flips the account's classification. Resolve by matching on customer *name* and preferring the most recent order date.
- **The 2,000-row query cap is the single most dangerous failure mode in this build.** It silently drops data rather than erroring, and it looks identical to "there's genuinely nothing there." Always filter tightly (by account list, by status, by amount threshold) rather than pulling broadly and filtering client-side.

---

## 6. Data model (JS arrays embedded in the artifact)

```js
QUOTES = [{
  quoteNo, customer, custNo, owner, bcSalesperson, status,
  followUpDate, followUpCode, followUpSalesperson, dueDate, expirationDate,
  lostCode, reasonCode, // present on active quotes too -- BC never closes them, so they stay here
  quoteTotal, lastOrderDate, daysSinceLastOrder, classification, // customer-level classification, fallback
  moreLinesValue,   // $ gap between quoteTotal and sum of displayed lines (freight, NCNR/comment lines)
  tierCount, tierOverstated, // volume-tier detection; tierOverstated > 0 drives the "Volume tiers" flag
  lines: [{ lineNo, item, desc, qty, unitPrice, lineAmount, stock,
            itemClassification, itemDaysSince, itemLastOrderDate,
            itemLevel }] // true = real item history; false = customer-level fallback
}]

ARCHIVE = [{ ...same shape as QUOTES entries, plus: outcome:"Won", orderNo, orderExists, shipped, invoiced }]

LEGACY_WON = [{ quoteNo, invoiceNo, customer, custNo, owner, amount, amountKnown, documentDate }] // header-level only

LOST = [{ quoteNo, customer, custNo, owner, bcSalesperson, quoteTotal,
          lostCode, reasonCode, status, codeMismatch,  // codeMismatch = exactly one of the two codes set
          followUpDate, expirationDate, dueDate }]

ALL_ACCOUNTS = [{ custNo, name, owner }] // full 133-account roster, used to compute zero-activity accounts

DQ_EVIDENCE = { ... } // verified facts about the pull, so the Data Quality view shows evidence, not stored conclusions
```

**Rep-edit preservation:** rep-entered Quote Type / Target Price / Comments live in `window.storage` under `qoc-line-edits-v1`, keyed `quoteNo|lineNo`. A refresh replaces the data arrays only; because the keys are stable and storage is never cleared, rep edits merge back onto fresh BC data automatically. Nothing in the refresh path writes to storage.

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
11. **Lost Quotes** — every quote carrying a `lostCode`, pulled per-rep with **no $5,000 gate**. Columns: quote number, customer, owner, quoteTotal, lostCode, reasonCode, status. Status will still read Quote Issued / Quote WIP / Pending Approval — that is the known BC limitation above, shown deliberately rather than hidden. Rows where exactly one of lostCode/reasonCode is populated are flagged and sorted first, then by quoteTotal descending. (Given `reasonCode` is unused company-wide, that is currently *every* row — which is the point.) Loss-reason codes and rep names render as clickable filter chips.

**Interactive filters (My Quote Queue, Rep View, Manager View, Lost Quotes):** KPI tiles and inline flag badges are clickable filters — clicking filters the table to matching rows, clicking again toggles off. **One active filter at a time (v1)**, so selecting a different chip replaces the current one. While filtered, a `Showing: Overdue (12)` indicator appears with a clear (×) button.

Every quote number and customer name in every table is clickable → opens a modal (quote detail with all lines + QUOTE/VALUE coaching questions generated from facts already on that quote, or customer detail listing all their quotes, with a back-link between the two).

---

## 8. QUOTE+VALUE coaching panel

Generated per-quote from data already present on that specific quote — **never invent customer-specific facts not in the pull.** Structure: Q (Bid/Buy confirmed? application/project?), U (how will they decide? stock status), O (highest-value line, unitemized gap), T (value story beyond price), E (follow-up status, has outcome been documented). Phrase everything as a question to confirm, not a stated fact.

---

## 9. Still open / roadmap for whoever continues this

- **Full Won history from the archive tables.** The single biggest remaining gap. The live `salesQuotes` table drops a quote once its order completes, so the Won archive only holds wins still lingering there and any win rate from it is biased low. Pulling `salesQuoteArchives` / `salesHeaderArchives` is the fix, and it also unblocks the item below.
- **Repeated No-Win** only sees the current active-quote snapshot. Needs the full historical quote log (same archive-table pull as above) to properly detect "quoted repeatedly, never won" patterns.
- ~~**Item-level classification** only covers the original ~28 quotes~~ — **DONE (2026-10-02).** The join bug is resolved (see §3b for the working query shape); item-level classification now covers 368 of 488 active quote lines across all four reps. The remaining 120 are parts with no purchase history for that customer, which correctly fall back to customer-level.
- **Legacy Won pull is deliberately bounded to post-cutover invoicing (2026-04-01+).** Going further back is possible — the data reaches to 2023 — but pricing it is the blocker, not finding it: `postedSalesInvoices` renders a PDF per row (§3a) and cannot price 13k invoices in reasonable time. A bulk amount source would be needed.
- **Real Lost tracking** is a process problem, not a tool problem — see the separate instructional doc and the leadership proposal doc for Craig/Joanna/Jonathan.
- Nothing in this app writes back to BC. If that's ever wanted (e.g., a "close this quote" button), that's a meaningfully bigger scope change — confirm real business appetite for it before building.

---

## 10. Prompt to hand to Claude to continue this

> "Using the attached account list and the Business Central Explorer / PowerBI MCP connectors, refresh the Quote Opportunity Center dashboard following the exact data-pull methodology in Section 4 of this spec. Pull per-rep, not broadly, to avoid the 2,000-row cap silently dropping data. Preserve the existing app structure, styling, and views (Section 7) exactly — just refresh the embedded QUOTES/ARCHIVE/LEGACY_WON/LOST/ALL_ACCOUNTS arrays with current data, merge onto `window.storage` so rep-edited Quote Type / Target Price / Comments are never overwritten, re-verify the findings in Section 5 still hold, and flag anything new the same way — plainly, in Data Quality, with the debugging evidence shown, not just the conclusion."

**Four traps that have each cost a full refresh cycle — read these before starting:**
1. The **2,000-row cap is silent**. Filter by explicit customer number, one rep at a time, and *bound invoice pulls by date*. Check every response's row count; anything at or near 2,000 means truncation, not "that's all there is". Two separate figures in this spec were once wrong because a capped result was read as a complete population.
2. **`postedSalesInvoices` renders a PDF per row.** Chunk at ~10, never 40. See §3a.
3. **The SalesModel item-history query shape matters.** Group on dimension columns, aggregate `f_Transaction[OrderDate]`. See §3b for the working query and the two traps.
4. **`not` filters are rejected** by this BC OData endpoint. Do exclusions client-side.

**Verify before presenting.** The refresh is testable end-to-end in headless Chromium (Playwright is available): stub `window.storage`, load the file, assert every view renders without JS errors, assert the click-filters narrow the table and toggle off, and assert a rep-entered value survives a page reload. The 2026-10-02 refresh shipped with 45 such checks passing; re-run them rather than eyeballing the artifact.
