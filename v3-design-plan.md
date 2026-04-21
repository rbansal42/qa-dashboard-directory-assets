# Find Your Dashboard — v3 design plan

This plan covers:
- **Key metrics band** (area-aware) on the landing page
- **Most-popular dashboards** section (cross-area when unfiltered; in-area when filtered)
- **Metadata** shown in every list row (refresh frequency, department, owner, last updated)
- **Feature 1 — Tooltip pages** on hover
- **Feature 3 — Sort-order field parameter** above the dashboard list
- **Feature 5 — Drillthrough "Dashboard details" page**

Features marked **FUTURE** at the bottom.

---

## 1. Data model changes

### 1a. Extend `DashboardDirectory`

| Column | Type | Example | Why |
|---|---|---|---|
| `DashboardName` | Text | OTP & Delay Analysis | existing |
| `Description` | Text | Flights running behind schedule… | existing |
| `Category` | Text | Flight Ops | existing |
| `URL` | Text (Web URL) | https://… | existing |
| `Keywords` | Text (comma-sep) | otp, delay, punctuality | existing |
| **`Owner`** | Text | Aisha Rahman | **new — for metadata + drillthrough** |
| **`Department`** | Text | Integrated Operations Control | **new — shown in list** |
| **`RefreshFrequency`** | Text | Hourly \| Daily \| Real-time \| Weekly \| On-demand | **new — shown in list + drillthrough** |
| **`DataSource`** | Text | AIMS / Sabre / AMOS | **new — drillthrough only** |
| **`LastRefresh`** | Datetime | 2026-04-19 08:14 | **new — shown in list as "x hrs ago"** |
| **`CreatedOn`** | Date | 2026-02-03 | **new — for "new this month" KPI** |
| **`LongDescription`** | Text | Paragraph… | **new — drillthrough** |

### 1b. Add a `ViewLog` fact table

```
ViewLog(
  ViewID          int,
  DashboardName   text  -- FK to DashboardDirectory
  UserEmail       text,
  OpenedAt        datetime
)
```

Populate from any lightweight tracker: a PowerAutomate flow on each dashboard's "on page load" event, or a Log Analytics query if the PBI workspace already has activity logging enabled.

Create a many-to-one relationship `ViewLog[DashboardName] → DashboardDirectory[DashboardName]`.

### 1c. Core measures

```DAX
-- 30-day views per dashboard (used for popularity ranking + KPI band)
Views30d =
CALCULATE(
    COUNTROWS(ViewLog),
    ViewLog[OpenedAt] >= TODAY() - 30
)

-- Popularity rank within current filter context
PopularityRank =
RANKX(
    ALLSELECTED(DashboardDirectory[DashboardName]),
    [Views30d],
    ,
    DESC,
    Dense
)

-- "Fresh %" = share of dashboards refreshed within their SLA
-- SLA: Hourly < 2hr, Daily < 26hr, Weekly < 8d, etc.
FreshPct =
DIVIDE(
    COUNTROWS(
        FILTER(
            DashboardDirectory,
            SWITCH(
                DashboardDirectory[RefreshFrequency],
                "Real-time", (NOW() - DashboardDirectory[LastRefresh]) * 24 * 60 <= 5,
                "Hourly",    (NOW() - DashboardDirectory[LastRefresh]) * 24 <= 2,
                "Daily",     (NOW() - DashboardDirectory[LastRefresh]) <= 1.1,
                "Weekly",    (NOW() - DashboardDirectory[LastRefresh]) <= 8,
                TRUE
            )
        )
    ),
    COUNTROWS(DashboardDirectory)
)

-- Dashboards added in the last 30 days
NewThisMonth =
CALCULATE(
    COUNTROWS(DashboardDirectory),
    DashboardDirectory[CreatedOn] >= TODAY() - 30
)

-- Total dashboards in current filter context
TotalDashboards = COUNTROWS(DashboardDirectory)

-- Time since last refresh, as a dynamic label
LastRefreshAgo =
VAR _hrs = (NOW() - SELECTEDVALUE(DashboardDirectory[LastRefresh])) * 24
RETURN SWITCH(
    TRUE(),
    _hrs < 1, FORMAT(_hrs * 60, "0") & " min ago",
    _hrs < 24, FORMAT(_hrs, "0") & " hr ago",
    FORMAT(_hrs / 24, "0") & " days ago"
)
```

---

## 2. Landing page layout (revised)

```
┌─ Ribbon ─────────────────────────────────────────────────────────────────┐
├─ Banner  [Title+Subtitle]   [Pinned ×3]   [Search]  ─────────────────────┤
├─ Nav strip  [All Areas]  [Crew Control] … [Training]            [count] ─┤
│                                                                          │
│  ┌─ Panel ──────────────────────────────────────────────────────────┐    │
│  │ Examples: [OTP] [Delay] [Fuel uplift] [GLP] [Crew roster]        │    │
│  │                                                                  │    │
│  │ KPI BAND (5 cards) — figures change with selected area:          │    │
│  │ ┌───────┬───────┬───────┬───────────┬──────────────┐             │    │
│  │ │Total  │Views  │New    │Fresh      │Avg refresh   │             │    │
│  │ │ 132   │12,450 │  8    │  94%      │ 2.3 hrs      │             │    │
│  │ └───────┴───────┴───────┴───────────┴──────────────┘             │    │
│  │                                                                  │    │
│  │ ┌─ Most Popular ─┬─ Keyword cloud ─┬─ All dashboards ───────┐    │    │
│  │ │ 1 OTP Tracker  │                 │ Sort: [Popularity ∨]   │    │    │
│  │ │ ██████████ 1.2k│   word cloud    │ ┌────────────────────┐ │    │    │
│  │ │ 2 Fuel Perf.   │                 │ │Name | Dept | Freq  │ │    │    │
│  │ │ ██████   820   │                 │ │ … rows with meta … │ │    │    │
│  │ │ 3 IOC Live     │                 │ └────────────────────┘ │    │    │
│  │ │ ██████   780   │                 │                        │    │    │
│  │ │ 4 ADSB DOH     │                 │ 37 dashboards          │    │    │
│  │ │ ████     610   │                 │                        │    │    │
│  │ │ 5 Flight Disp  │                 │                        │    │    │
│  │ │ ████     540   │                 │                        │    │    │
│  │ └────────────────┴─────────────────┴────────────────────────┘    │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Footer                                                                  │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2a. KPI band (5 cards) — area-aware

When **All Areas** is selected, the cards show directory-wide figures. When a specific area is selected via the Tile slicer, every card filters automatically (same data model, different SELECTEDVALUE context).

| Card | Value source | When filtered by area |
|---|---|---|
| Total dashboards | `TotalDashboards` | Shows count in that area |
| Views (30d) | `Views30d` | Shows area-specific views |
| New this month | `NewThisMonth` | Shows new in that area |
| Fresh % | `FreshPct` | Shows area freshness |
| Most-opened | `TOPN(1, …, [Views30d])` | Shows the #1 in that area |

Style: small cards in a single horizontal strip, each ~220×90 px. Big number in Georgia/Cormorant serif; tiny label above in uppercase 9 pt; light amber bar underneath showing period.

### 2b. Most Popular column — top 5 with mini bars

Left column of the main grid. Visual: the new **Card visual** (2024+) with multiple rows, OR a **Clustered bar chart** on `Views30d` with rank labels.

Ranking respects the current filter:
- No area selected → top 5 across all 132 dashboards
- "Flight Ops" selected → top 5 within Flight Ops only
- "Flight Ops" + search "otp" → top 5 of Flight Ops dashboards matching "otp"

Each row shows:
- Rank (`1`, `2`, `3` …)
- Dashboard name (clickable — opens the dashboard)
- Mini horizontal bar showing `Views30d` relative to the max in current scope
- View count as a small label

### 2c. All-dashboards list — now with metadata

Columns in the matrix:

| # | Column | Example | Notes |
|---|---|---|---|
| 1 | Name | OTP & Delay Analysis | Conditional format → Web URL, maroon link |
| 2 | Dept | IOC | Small amber tag (conditional format → font colour `#B8893A`) |
| 3 | Freq | Hourly | Pill-styled via conditional background colour |
| 4 | Updated | 2 hr ago | `LastRefreshAgo` measure |

Hide `Category` since it's redundant with the nav strip.

Heat indicator: conditional background on the Name cell using `Views30d` — subtle cream → amber gradient, so busy dashboards visually pop.

---

## 3. Feature 3 — Sort-order field parameter

### 3a. Create the parameter

Power BI Desktop → Modeling → **New parameter → Fields**.

Name: `SortBy`. Add 4 fields:

| Display | Field |
|---|---|
| Most popular | `[Views30d]` |
| Name (A → Z) | `DashboardDirectory[DashboardName]` |
| Recently updated | `DashboardDirectory[LastRefresh]` |
| Refresh cadence | `DashboardDirectory[RefreshFrequency]` |

Check **Add slicer to this page**. This creates a slicer visual automatically.

### 3b. Wire it to the matrix

1. Drag the `SortBy` field parameter into the matrix's **Values** well, placed first.
2. Set its visibility: right-click the column → **Hide** (keeps it as the sort driver without showing it).
3. On the matrix, click the column header of what you want sorted by the parameter → **Sort descending**. PBI uses the parameter as the sort key.

### 3c. Style the slicer

- Visual → Slicer settings → Style → **Tile** (same as the nav strip, smaller)
- Font: Segoe UI Semibold 10 pt, uppercase
- Selected colour: `#43061F` fill, `#F7EBD4` text
- Position: directly above the matrix, right-aligned with label "Sort:" to its left (text box).

Clicking a different sort tile immediately re-sorts the list. Zero bookmarks.

---

## 4. Feature 1 — Tooltip pages

### 4a. Build a hidden tooltip page

1. New page → name it `TT_DashboardTooltip`.
2. Page settings → Canvas → **Tooltip** = On; Type = Custom.
3. Page size: 320 × 180 px.
4. Background: white, 1 px border `#E8DFD0`, rounded 6 px, drop shadow.
5. Place visuals on this canvas:

```
┌─────────────────────────────────────┐
│ [Dashboard name]          [Cat pill]│
│ Department · Owner                  │
│ ▁▂▃▅▆▇▇▅▄▃ (30-day sparkline)      │
│ Refresh: Hourly · Last: 2 hr ago    │
│ Views 30d: 1,240                    │
└─────────────────────────────────────┘
```

- Title: text box bound to `SELECTEDVALUE(DashboardDirectory[DashboardName])`
- Sparkline: Area chart on `ViewLog` grouped by day, 30 days; no axes, gradient fill amber → maroon
- Refresh label: `LastRefreshAgo` measure
- Views: Card with `Views30d`

6. Page → View → **Hide page** (it shouldn't show in navigation).

### 4b. Attach to the matrix

Select the matrix → Format → **Tooltips** → Type: **Report page** → Page: `TT_DashboardTooltip`.

Hovering any row now summons the card, driven by that row's context.

Same trick for the **Most Popular** visual and the keyword cloud.

---

## 5. Feature 5 — Drillthrough "Dashboard details" page

### 5a. Page layout

```
┌─────────────────────────────────────────────────────────────────────┐
│ ← Back to directory                            [Open dashboard ↗]   │
├─────────────────────────────────────────────────────────────────────┤
│ [Dashboard Title]                                    [Category pill]│
│                                                                     │
│ ┌─ Metadata bar ───────────────────────────────────────────────┐    │
│ │ Owner     Aisha Rahman                                       │    │
│ │ Dept      Integrated Operations Control                      │    │
│ │ Source    AIMS · SQL Server                                  │    │
│ │ Refresh   Hourly (SLA 2 hr)                                  │    │
│ │ Last      2 hr 14 min ago              [●] Within SLA        │    │
│ │ Created   3 Feb 2026                                         │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                     │
│ ┌─ Description ──────────────┬─ 90-day usage ─────────────────┐     │
│ │ [long description]         │  [line chart: views/day]       │     │
│ │                            │                                │     │
│ │ Common questions:          │  Peak: Mon 9 AM                │     │
│ │ • What is the GLP code..   │  30-d total: 1,240             │     │
│ │ • How many flights are..   │  Unique users: 187             │     │
│ └────────────────────────────┴────────────────────────────────┘     │
│                                                                     │
│ ┌─ Related dashboards ───────────────────────────────────────┐      │
│ │ [3-column grid of related cards]                           │      │
│ └────────────────────────────────────────────────────────────┘      │
│                                                                     │
│ Refresh history (last 10 runs) — small table                        │
└─────────────────────────────────────────────────────────────────────┘
```

### 5b. Wire it up

1. New page → name it `P_DashboardDetail`. Canvas 1280 × 820.
2. Page settings → **Drillthrough** → add `DashboardDirectory[DashboardName]` to **Drillthrough filters**.
3. Add a back button: Insert → Button → **Back**. Action: go to previous page.
4. Every visual on this page sits on the drillthrough filter automatically — when you right-click a row on the landing page → *Drill through → P_DashboardDetail*, the whole page is scoped to that one dashboard.

### 5c. Related-dashboards strip

Bottom row: a Clustered bar chart or small table visual that shows up to 3 rows where:

```DAX
RelatedTable =
VAR _me = SELECTEDVALUE(DashboardDirectory[DashboardName])
VAR _myCat = SELECTEDVALUE(DashboardDirectory[Category])
VAR _myKw = SELECTEDVALUE(DashboardDirectory[Keywords])
RETURN
TOPN(
    3,
    FILTER(
        ALL(DashboardDirectory),
        DashboardDirectory[DashboardName] <> _me
            && (
                DashboardDirectory[Category] = _myCat
                || CONTAINSSTRING(DashboardDirectory[Keywords], _myKw)
            )
    ),
    [Views30d],
    DESC
)
```

Calculated table driving a small matrix with Name + mini bar.

### 5d. "Within SLA" traffic-light indicator

Single-cell card with a coloured background driven by a measure:

```DAX
SLAStatus =
VAR _hrs = (NOW() - SELECTEDVALUE(DashboardDirectory[LastRefresh])) * 24
VAR _freq = SELECTEDVALUE(DashboardDirectory[RefreshFrequency])
VAR _slaHrs = SWITCH(_freq,
    "Real-time", 0.083,
    "Hourly", 2,
    "Daily", 26,
    "Weekly", 192,
    999
)
RETURN
IF(_hrs <= _slaHrs, "● Within SLA", "● Stale")
```

Conditional background: green if text = "● Within SLA", red if "● Stale".

---

## 6. Feature 1/3/5 build order (≈ 4 hours total)

1. **Data model** (45 min) — add the new columns, populate with real values for the 8 pilot dashboards, create `ViewLog`, relationship, 6 core measures.
2. **Metadata columns in matrix** (20 min) — drop `Department`, `RefreshFrequency`, `LastRefreshAgo` into the matrix; style with conditional formatting.
3. **KPI band** (30 min) — 5 small cards in a strip above the main grid. Bind to the 5 measures.
4. **Most Popular** visual (30 min) — new Card visual or Clustered bar chart on top-5 by `Views30d`.
5. **Sort field parameter** (20 min) — Modeling → New parameter → Fields → 4 options → wire into matrix.
6. **Tooltip page** (45 min) — hidden `TT_DashboardTooltip` page with name, sparkline, meta; attach to matrix + Most Popular.
7. **Drillthrough page** (60 min) — `P_DashboardDetail` with metadata bar, description, 90-day chart, related table, SLA indicator.

Total: ~4 hours of focused build.

---

## 7. Other visual changes on the landing page

- Remove the second stats card (donut "Top areas this week") — the KPI band now covers this.
- Keep the keyword cloud in the centre column; it complements the metadata-rich list.
- Examples row stays; chips set the search slicer via **Set slicer** action (already in v2 guide).

---

## 8. Marked for FUTURE

These were in the earlier list. Parking them until after v3 ships:

- **2. Smart narrative** — auto-generated insight paragraph. Nice-to-have; needs the usage log healthy first.
- **4. Usage heat column** — partially covered by the Most-Popular strip; full-row gradient is a polish for v4.
- **6. Recently opened by you** — per-user, requires PowerAutomate flow + `USERPRINCIPALNAME()` wiring.
- **7. Favourites / per-user pinning** — replaces hardcoded pins with user-scoped table.
- **8. Q&A visual** — natural-language search. Consider once we have enough metadata to make it useful.
- **9. Row-level security by department** — only if directory spans sensitive areas.
- **10. Request-access + PowerAutomate approval** — bigger investment; queue for v5.
- **11. Mobile-optimised view** — revisit after desktop v3 stabilises.
- **12. Decomposition tree on usage** — BI-team internal tool; not a v3 priority.
- **Copilot for Power BI** — enable if/when the tenant gets F-SKU capacity.

---

## 9. Summary of what changes for the user

Before v3:
- Search + 12 category pills + a flat list

After v3:
- Search + 12 category pills + 5 area-aware KPIs + Top-5 most-popular per context + keyword cloud + sortable metadata-rich list + hover tooltips + right-click drillthrough details

**No new buttons, no new bookmarks.** Every filter still flows through the existing Tile slicer and Search slicer. The new surface is driven entirely by adding data + placing native PBI visuals.
