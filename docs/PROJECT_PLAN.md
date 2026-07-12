# Swedish Housing Market Dashboard — v1 (MVP) Project Plan

**Tool: custom HTML/JS dashboard** (final decision — see §0 for the path that led here). This file supersedes the Power BI and Looker Studio versions that came before it.

## 0. How the tool choice evolved

Three tools were considered in order, each ruled out or replaced for a concrete reason:

1. **Power BI Desktop** — the original plan. Dropped when the user didn't have Excel available for a side-step in the reshape process and, separately, wanted something faster to iterate on than a locally-installed desktop tool.
2. **Google Looker Studio** — free, web-based, no install. Data was reshaped via a one-time Google Apps Script, but the script had a row-alignment bug (Year column ended up holding another county's sales-count values). Rather than keep debugging blind through spreadsheet screenshots, the reshape was redone **locally** with a verified PowerShell script — see §4.
3. **Custom HTML/JS dashboard (final choice)** — once manually building each chart in Looker Studio's UI started to feel slow, the user asked whether an AI-controllable alternative existed. None of Looker Studio/Power BI/Tableau/Sheets expose an API this assistant can drive — but a self-contained HTML/JS page is something the assistant can build entirely in code, with no manual UI steps. **Known tradeoff, accepted explicitly by the user:** this is not "Looker Studio" or "Power BI" experience for a resume — it demonstrates data visualization / front-end skill instead of a named BI tool. If a recognized BI tool matters later, the `housing_flat.csv` produced in §4 is already in the right shape to import into either one.

## 1. Scope

A single-page dashboard giving an overview of the Swedish housing market, answering:

1. How have housing prices changed over time?
2. Which counties have the highest and lowest housing prices? *(county-level, not municipality — see §2a)*
3. How do apartments compare with villas? *(deferred to v2)*
4. What is the average price per square meter? *(dropped for v1 — no m² field in the source table, see §2a)*
5. How many properties have been sold?

**v1 scope decision:** villas/småhus (one- or two-dwelling buildings) only, at **county × year** granularity.

## 2. Data source

**Primary: Statistics Sweden (SCB), Statistikdatabasen**
Path used: *Housing, construction and building → Real estate prices and registrations of title → Prices of real estate → **"Sold one- and two-dwelling buildings by region, observations and year"*** (table code `BO0501O3`).

**File downloaded:** `data/raw/BO0501O3_20260708-185849.csv`

Selections made:
- **Observations (measures):** all 4 — Number, Purchase price average (1 000 SEK), Assessed value average (1 000 SEK), Purchase-price-coefficient
- **Region:** classification = **County** (21 counties)
- **Year:** all 26 years, 2000–2025

**File format notes (confirmed from the actual export):**
- Encoding is **Windows-1252**, not UTF-8 — a corrected copy was made: `data/raw/BO0501O3_utf8.csv`.
- Layout is **wide**: row 1 = title, row 2 = blank, row 3 = headers, rows 4–24 = one row per county, 104 data columns (4 measures × 26 years). Reshaped in §4.

### 2a. Two scope adjustments made after seeing the real data

1. **County instead of municipality** — this table's region classification only offered National areas / County / Sweden, no municipality level (SCB commonly suppresses municipality-level price averages for small-sample years). County (21 regions) is the finest level available.
2. **No price-per-m² KPI in v1** — no living-area (m²) field in this table. v1 uses **Average purchase price** and **Average assessed value** instead.

**Not used in v1 (flagged for v2):** Apartment/bostadsrätt data — SCB has a **"Tenant-owned flats"** category in the same section (seen while browsing), worth checking first before reaching for Booli/Mäklarstatistik.

## 3. Data model (flat table, star-schema thinking)

Designed as if it were a star schema, then physically flattened into one table (see §4) — this makes it a straight re-import if this ever moves to a real BI tool:

| Conceptual role | Field(s) | Notes |
|---|---|---|
| Fact grain | County × Year | One row per county per year |
| Measures | NumberSold, AvgPurchasePriceSEK1000, AvgAssessedValueSEK1000, PurchasePriceCoefficient | Straight from source |
| "DimCounty" fields | CountyCode, CountyName | Split from `region` column (e.g. "01 Stockholm county" → "01" / "Stockholm county") |
| "DimYear" fields | Year | Already a plain year in the source — no calendar table needed, data is annual only |
| "DimPropertyType" field | PropertyType | Constant `"Villa"` for every row in v1 — costs nothing now, and means v2 (apartments) is just more rows with `PropertyType = "Apartment"`, not a redesign |

## 4. Data prep (done, verified)

An Apps Script approach was tried first inside Google Sheets but produced misaligned rows — not worth debugging blind through a spreadsheet UI. The reshape was instead done **locally** with a PowerShell script, and the output was verified cell-by-cell against the source before use.

1. Source: `data/raw/BO0501O3_utf8.csv` (UTF-8 re-encoded copy of the SCB export).
2. Reshape script: `scratchpad/reshape.ps1` — parses the wide CSV (skips title+blank rows, reads the real header row, splits each of the 104 measure+year column headers), emits one flat row per county × year.
3. **Output: `data/processed/housing_flat.csv`** — 546 data rows (21 counties × 26 years) + header: `CountyCode, CountyName, Year, PropertyType, NumberSold, AvgPurchasePriceSEK1000, AvgAssessedValueSEK1000, PurchasePriceCoefficient`.
4. Verified against source: e.g. Stockholm county, 2000 → NumberSold 6772, AvgPurchasePriceSEK1000 1968, AvgAssessedValueSEK1000 778, PurchasePriceCoefficient 2.49 — matches the raw export exactly.

## 5. Dashboard implementation

**Location:** `dashboard/index.html` — a single self-contained HTML/CSS/JS file (data embedded inline, no external requests, no build step). Published as a Claude artifact for live preview during development.

**Design system:** followed the project's `dataviz` skill — hand-rolled SVG charts (no external chart library, so nothing to violate the sandbox's CSP), the skill's validated blue categorical color (`#2a78d6` light / `#3987e5` dark) as the single series color, light/dark mode both styled via `prefers-color-scheme` and a `data-theme` override, hover tooltips + crosshair on the line/bar charts, and a table-first approach to labeling (direct labels only on chart extremes, full values always in the tooltip).

**Filters (one row, above everything, per the skill's composition rule):** Year-from / Year-to, and a County dropdown (default "All counties"). Every KPI and chart re-renders from the same filtered slice.

**KPIs (the DAX-measure equivalent, computed in JS on each render):**
| KPI | Formula | Why weighted, not a plain average |
|---|---|---|
| Total Sales | `Σ NumberSold` | Sum across the filtered scope |
| Average Price | `Σ(AvgPrice × NumberSold) / Σ NumberSold` | Volume-weighted — a plain average of the 21 counties' averages would let a low-volume county move the number as much as Stockholm. Same principle as a proper DAX measure (`SUMX(...)/SUM(...)`), just written in JS |
| Average Assessed Value | same weighting as above | |
| Avg Purchase-Price-Coefficient | same weighting as above | Sale price ÷ assessed value; above 1.0 means selling above assessed value |

**Charts** (mapped from the original wireframe onto what the data actually supports):
1. **Housing price trend** (line) — average price by year, respects the County filter
2. **Sales volume by year** (thin vertical bars) — replaces "how many sold" as a trend rather than a single number
3. **Average price by county** (horizontal bar, sorted descending, all 21 counties always shown, selected county highlighted) — answers "highest/lowest"
4. **Avg purchase-price-coefficient by county** (horizontal bar) — stands in for the "price per m²" comparison that isn't available in this data (§2a), and is a genuinely useful market-heat signal on its own

## 6. Folder structure

```
HousingProject/
  data/
    raw/          <- original SCB export + UTF-8 re-encoded copy
    processed/    <- housing_flat.csv (final flat table, verified)
  dashboard/
    index.html    <- the dashboard (self-contained, data embedded)
  docs/
    PROJECT_PLAN.md   <- this file
  README.md       <- portfolio-facing description (next step, §8)
```

## 7. Tools required

- A modern browser to view `dashboard/index.html` (or the published artifact link)
- Nothing else — no paid tools, no APIs, no Excel, no Power BI, no Google account

## 8. Status / next concrete step

- [x] Data downloaded, re-encoded, reshaped, and verified (§2, §4)
- [x] Data model defined (§3)
- [x] Dashboard built and published as an artifact (§5)
- [ ] **Next:** review the live dashboard for correctness and readability (label collisions, tooltip accuracy, dark mode), then write the portfolio `README.md` (project overview, objectives, data source, tools, features, skills demonstrated) and decide whether to host `dashboard/index.html` anywhere permanent (e.g. GitHub Pages) for the portfolio link.

## 9. Roadmap (later versions)

- **v2:** Add apartments/bostadsrätter (check SCB's own "Tenant-owned flats" table first — fallback to Booli/Mäklarstatistik), municipality-level detail if a suitable table is found, quarterly granularity, price-per-m² if a source is found, a real YoY comparison metric.
- **v3:** Affordability index (price vs. income), a Sweden map view by county/municipality, and — if a named BI tool credential becomes worth adding — reimport `housing_flat.csv` into Power BI or Looker Studio as a second, tool-specific version of the same analysis.
