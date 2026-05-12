# First Bus Tools — Bus Reliability & Failure Prediction (`BusReliability`)

Paste this file into a new chat to restore full project context.

---

## Project Overview

**Repo:** new repo — `BusReliability` (start fresh)  
**Status:** Ready to build  
**Stack:** Single self-contained HTML file, vanilla JS, PapaParse (CDN), First Bus branding  
**What it does:** Per-bus health dashboard — for every bus in the fleet, shows each part
ever fitted, when it was last fitted, the calculated Mean Time Between Replacements (MTBR),
and a predicted next failure date. Parts are flagged as Overdue, Due Soon, OK, or Unknown.

---

## The Problem It Solves

Currently there is no way to know whether a part on a specific bus is approaching the end
of its expected life. Maintenance is reactive. This tool makes it predictive:  
> *"Bus 35201 — brake caliper last fitted 14 months ago, MTBR is 18 months, due in 4 months."*

---

## Data Requirements

| File | Format | Required columns | Notes |
|---|---|---|---|
| Fleet reference | CSV | Fleet No, Total Chassis Variant | Commission date optional but useful |
| Transaction history | CSV (3 years) | Fleet No, Part, PartDesc, Posting Date, Qty | More repeat events = more reliable MTBR |

**3 years is important.** Many parts only fail once or twice per bus per year; stacking
3 years provides enough repeat events to calculate a meaningful MTBR. Fewer than 2 events
for a (bus, part) pair means MTBR cannot be calculated — flagged as UNKNOWN confidence.

**Transaction filtering rule (same as all projects):**  
Only include rows where Qty > 0. Negative movements are returns/corrections, not fitments.

---

## Column Detection (carry forward from Bus-Fitment-Location)

Use the same `normaliseHeader()` pattern: lowercase → trim → collapse whitespace → exact
match → substring fallback.

**Known HxGN actual column names (put these first in candidate arrays):**

```js
// Fleet CSV
const COL_FLEET_MODEL   = ['total chassis variant','chassis variant','type','model','bus type','vehicle type','bus model','bodytype'];
const COL_FLEET_FLEETNO = ['fleet no','fleet number','fleet'];
const COL_FLEET_DATE    = ['commission date','commissioned','reg date','date commissioned'];

// Transaction CSV
const COL_TX_FLEETNO  = ['fleet no','fleet number','fleet'];
const COL_TX_MATERIAL = ['part','material','material no','material number','part no','part number','article'];
const COL_TX_DESC     = ['partdesc','part desc','description','material desc','material description','short text','part description'];
const COL_TX_QTY      = ['qty','quantity','issued qty','total qty','issue qty','movement qty'];
const COL_TX_DATE     = ['posting date','issue date','date','movement date'];
```

---

## Processing Methodology

### Step 1 — Build fleet index
`Map<fleetNo → { model, commissionDate }>` from the fleet CSV.  
Normalise fleet no: `.trim().toLowerCase()` on both sides of every join.

### Step 2 — Collect issue events per (bus, part)
For each transaction row where qty > 0:
- Resolve model from fleet index (skip if fleet no not found)
- Parse posting date to a JS Date object
- Accumulate into `Map<fleetNo, Map<partNo, IssueEvent[]>>`
  where `IssueEvent = { date, desc, model }`

### Step 3 — Calculate MTBR
For each (fleetNo, partNo) pair, sort events ascending by date.

**Per-bus MTBR** (preferred — used when ≥ 2 events on that bus):
```
intervals = [event[1].date - event[0].date, event[2].date - event[1].date, ...]
MTBR = average(intervals)  // in days
```

**Fleet-wide fallback** (used when < 2 events on that bus):  
Pool all events for the same (model, partNo) across all buses.  
Calculate average interval from that pooled set.  
Flag result as lower confidence.

**Confidence levels:**
- UNKNOWN — fewer than 2 data points in pool (cannot calculate)
- LOW — 2–4 data points
- MEDIUM — 5–9 data points
- HIGH — 10+ data points

### Step 4 — Calculate predicted next failure
```
lastFitment = most recent event date for (fleetNo, partNo)
predictedNext = lastFitment + MTBR (in days)
daysRemaining = predictedNext - today
```

### Step 5 — Status flags
```
OVERDUE   — daysRemaining < 0
DUE SOON  — daysRemaining < MTBR × 0.2   (configurable threshold, default 20%)
OK        — daysRemaining ≥ MTBR × 0.2
UNKNOWN   — MTBR not calculable
```

The 20% threshold is a config constant at the top of the JS — easy for an engineer to adjust.

---

## Output / UI

### Upload section
Two file inputs (Fleet CSV, 3-year Transaction CSV) + Load button.  
Status bar shows row counts and any column detection warnings.

### Selectors
- **Bus model dropdown** — populated after load; shows all models with transaction data
- **Fleet number dropdown** — filters to buses of the selected model; shows individual bus view

Both selectors work independently:
- Model selected → aggregate view across all buses of that type (avg MTBR, most common failures)
- Fleet number selected → per-bus view for that specific bus

### Per-bus health table
Columns: **Part No.** | **Description** | **Last Fitment** | **MTBR** | **Predicted Next** | **Status** | **Confidence**

- Status shown as a coloured badge: red = OVERDUE, amber = DUE SOON, green = OK, grey = UNKNOWN
- Default sort: OVERDUE first, then DUE SOON, then OK, then UNKNOWN
- Secondary sort within status group: soonest due date first
- Table is searchable by part number or description (same search input pattern as Bus-Fitment-Location)

### Model-level aggregate view (when no fleet number selected)
Shows: most-replaced parts across the fleet, average MTBR per part, how many buses have
that part flagged as overdue or due soon. Useful for a parts manager planning stock.

### Export CSV button
Same pattern as Bus-Fitment-Location: downloads the current view (respecting model/fleet
filter and search term) as a .csv file.

---

## AppState Structure

```js
const AppState = {
  fleetRows: [], fleetCols: {},
  txRows: [], txCols: {},
  fleetIndex: new Map(),          // Map<fleetNo → { model, commissionDate }>
  eventIndex: new Map(),          // Map<fleetNo, Map<partNo, IssueEvent[]>>
  mtbrIndex: new Map(),           // Map<key → { mtbr, confidence, source }>  key = fleetNo+'|'+partNo or model+'|'+partNo
  healthRows: [],                 // flattened, sorted for current view
  models: [],
  modelFleets: new Map(),         // Map<model → fleetNo[]>
  selectedModel: null,
  selectedFleet: null,
  sortCol: 'status', sortAsc: true,
};
```

---

## File Layout

```
bus-reliability.html
  <head>
    <style>  ← all CSS, design tokens, status badge colours
  <body>
    <header>           ← First Bus branding (navy + magenta accent)
    <main>
      #upload-section  ← Fleet CSV + Transaction CSV inputs
      #status-bar
      #selector-section (hidden until load)
        #model-select
        #fleet-select
      #health-section (hidden until model selected)
        #summary-bar    ← "X overdue, Y due soon across Z buses"
        #search-wrap
        #health-table-wrapper
          table#health-table
      #export-row
  <script src="https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.4.1/papaparse.min.js">
  <script>
    §1  Config (MTBR_WARNING_THRESHOLD = 0.2, confidence bands)
    §2  Column detection (normaliseHeader, detectColumn, resolveFleetCols, resolveTxCols)
    §3  App state + CSV parsing (parseCSV → Promise, handleLoad)
    §4  Data processing (buildFleetIndex, buildEventIndex, calcMTBR, buildHealthRows)
    §5  Rendering (renderSummaryBar, renderHealthTable, renderRow)
    §6  Export (exportHealthCSV)
    §7  UI utilities (escHtml, fmtDate, fmtDays, statusBadge)
    §8  Init (DOMContentLoaded, wire all events)
```

---

## Branding & Shared Conventions

- **First Bus colours:** magenta `#E4007C`, navy `#1D2B5F`
- Status badge colours: red `#d32f2f`, amber `#e65100`, green `#2e7d32`, grey `#757575`
- Single self-contained HTML file — no backend, no install, runs in any browser
- PapaParse CDN: `https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.4.1/papaparse.min.js`
- Large file write rule: use staged Bash heredoc (`<< 'UNIQUEEOF'`), ~150 lines per stage,
  echo line count after each stage. See `LARGE-FILE-GUIDE.md` in the Bus-Fitment-Location repo.

---

## Verification Checklist

1. Load fleet + 3-year transaction CSVs → status bar shows row counts, no column errors
2. Model dropdown populates; select a model → summary bar shows overdue/due-soon counts
3. Select a fleet number → per-bus table shows correct last fitment dates and predicted dates
4. A part with only 1 event → shows UNKNOWN status, no predicted date
5. A part with 10+ events → shows HIGH confidence, MTBR within expected range
6. Search box filters table live
7. Export CSV downloads correct data for current view
8. Status sort order: OVERDUE → DUE SOON → OK → UNKNOWN

---

## Engineering Escalation Add-on (near-trivial once core is built)

Once MTBR is calculated per (bus, part), flag buses where the observed replacement rate
is ≥ 2× the fleet average for that part. These buses have abnormal consumption and warrant
engineering investigation (damaged mounting, incorrect spec, operator behaviour, etc.).

Add as a separate tab or toggle in the same tool — reuses all the same data structures.
