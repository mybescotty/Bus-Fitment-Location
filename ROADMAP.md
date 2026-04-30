# First Bus Tools — Project Roadmap

This file captures planned and future projects agreed during development sessions.
Paste into a new chat to restore context.

---

## Current Project — Depot Startup MRP (`FbgStoresLaunch`)

**Status:** Active  
**Stack:** Single self-contained HTML file, JavaScript, SheetJS, PapaParse  
**What it does:** Plans Day One stockholding and steady-state MIN/MAX for a new bus depot, scaling group-wide HxGN transaction history to a user-defined fleet mix and age profile.  
**Inputs:** Fleet reference CSV, transaction history CSV, depot builder (bus type / qty / avg age), lead time, opening month  
**Output:** 3-sheet Excel workbook — Day One Pack, Steady-State MIN/MAX, Confidence Flags  
**Features built:** Transaction filtering, fleet resolution (direct + 3-digit prefix fallback), age weighting, confidence scoring, trend analysis, seasonal adjustment, data diagnostics panel, appendix/terminology guide, First Bus branding

---

## Next Project — Part Life Expectancy & Failure Prediction

**Status:** Planned — start new repo and new chat  
**Concept:** A predictive maintenance tool — not "what should I hold" but "what is going to fail and when"  
**Approach:**
- Calculate Mean Time Between Replacements (MTBR) per (part, bus type) from repeat issues in transaction history
- Stack up to 3 years of transaction data for reliable MTBR confidence (more repeat events per bus)
- Overlay fleet commission dates from fleet reference to calculate time since last known replacement
- Flag buses where age since last replacement ≥ MTBR × threshold — overdue or approaching due date
- Output: per-bus health dashboard ("Bus 35201: brake caliper overdue, alternator due in 2 months")

**Data requirements:** Same transaction CSV + fleet reference CSV (commission date column needed for age calculation). 3 years of data recommended.  
**Stack:** Same single HTML file approach  
**Note:** Once built, an engineering escalation flag (bus consuming a part at 2x fleet average rate) becomes a near-trivial add-on

---

## Current Project — Interactive Bus Parts Explorer (`Bus-Fitment-Location`)

**Status:** Active — this repo  
**Concept:** A visual SVG bus diagram where parts are mapped to locations on the bus for easier identification and lookup  
**Two-phase plan:**

### Phase 1 — Zone-based explorer (buildable with current data) ✅ In progress
- Bus model dropdown → SVG side-elevation of bus divided into zones (Engine, Brakes/Wheels/Suspension, Doors/Body, Electrical/HVAC/Interior)
- Material group descriptions used to map parts to zones (config-driven keyword matching)
- Click a zone → see all parts for that bus model in that zone, with usage counts
- No additional data needed beyond existing transaction + fleet reference files

### Phase 2 — Precise exploded diagram (bus breakdown project)
- Admin panel where an engineer can open a part and click/pin its exact location on the bus schematic
- Builds a persistent part → location mapping over time
- Once enough parts are placed, diagram becomes true exploded-view quality
- **Action:** Investigate whether any existing data source (manufacturer parts catalogue, existing bus breakdown documentation) can pre-populate the location mapping before falling back to manual pinning

**Stack:** Single HTML file, SVG drawn inline (no image files), admin mode for Phase 2  
**Note:** The user will investigate existing data sources for part locations — if available, Phase 2 can be automated rather than manual

---

## Other Ideas (Backlog)

- **Supplier performance tracker** — lead time actuals vs promised, fill rates, supply failure frequency
- **Parts commonality map** — cross-model parts matrix, standardise stockholding, reduce unique SKU count
- **Obsolescence detector** — parts with significant drop in issues over time; buses being phased out
- **Depot comparison tool** — per-bus consumption rates across depots, flag outliers for investigation
- **New bus type onboarding** — use manufacturer service schedule as proxy demand when no transaction history exists yet
- **Engineering escalation flag** — part replaced at 2x fleet average rate on a specific bus → flag for engineering review
- **Stores audit assistant** — compare system MIN/MAX against physical count, rank discrepancies

---

## Shared Conventions Across All Projects

- Single self-contained HTML file — no backend, no install, runs anywhere
- JavaScript in-browser processing, SheetJS for Excel output, PapaParse for CSV parsing
- First Bus branding: magenta `#E4007C`, navy `#1D2B5F`
- Column detection uses normalised exact match with substring fallback (handles HxGN column name variations)
- All transaction filtering follows METHODOLOGY.md rules from FbgStoresLaunch
