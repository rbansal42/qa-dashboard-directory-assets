# Making the page actually interactive (Power BI 2025)

**Goal:** clicking an area pill filters the dashboard list, the keyword cloud, the pinned tiles, the count, and the stats — all at once, with no bookmarks to maintain.

The 12-bookmark approach in the original tutorial still works, but it's brittle (every new category needs a new bookmark, every styling tweak has to be re-captured). Power BI 2025 has two features that replace it entirely:

1. **Tile slicers** — one slicer styled as pill buttons. One click, every visual on the page that shares the `Category` column filters automatically.
2. **Button "Set slicer" actions** — let the Example chips (OTP, Delay, etc.) push a value into the search slicer directly, no bookmarks.

---

## 1. Data model (unchanged)

One table, `DashboardDirectory`, with columns `DashboardName`, `Category`, `Description`, `URL`, `Keywords`, `LastUpdated`. `URL` must have Data Category = **Web URL**.

If you want usage stats (the "4,218 views" card), add a second table `DirectoryViews(DashboardName, ViewDate)` related many-to-one to `DashboardDirectory[DashboardName]`.

---

## 2. The Category nav strip — replace 12 buttons with ONE slicer

**Delete** the 12 category buttons and 12 bookmarks if you built them. In their place:

1. Insert → **Slicer**.
2. Drag `DashboardDirectory[Category]` into the Field well.
3. In Format → Visual → **Slicer settings → Style**, choose **Tile**. (On newer builds this is labelled **Button**.) Each distinct category now renders as a pill.
4. Format → Visual → **Selection controls**:
   - Single-select: **On**
   - Show "Select all" option: **On** (this becomes your "All Areas" pill)
5. Format → Visual → **Values**:
   - Font: Segoe UI Semibold 11pt
   - Default font colour: `#43061F`
   - Default background: transparent
   - Selected font colour: `#F7EBD4`
   - Selected background: `#43061F`
   - Hover background: `rgba(67,6,31,0.08)` — set in the **Hover** sub-section
6. Format → Visual → **Border**: off. **Padding**: 4 px horizontal, 2 px vertical.
7. Position the slicer spanning the width of the amber nav strip.

**What you get automatically:** click any pill → every visual bound to `DashboardDirectory` (matrix, word cloud, count cards) filters to that category. No bookmarks. No button actions. No maintenance when you add a new category.

---

## 3. The search box — text slicer (also auto-filters everything)

1. Insert → **Slicer** on `DashboardDirectory[Keywords]`.
2. Format → Visual → **Slicer settings → Style** → **Search**.
3. Turn OFF the selection checkboxes (Format → Visual → **Selection controls → Show Select all** = Off, and hide the value list by setting its height to 0 or collapsing it).

   — On 2025 builds, there's a cleaner option: Format → Visual → **Slicer settings → Slicer header** off, and under **Selection controls** set *"Show as search box only"* = On (this hides the checkbox list entirely).
4. Position it top-right of the banner. Style per the mockup.

**Typing in it now filters every visual on the page.** The matrix narrows; the word-cloud re-weights; the count card updates live.

---

## 4. The Example chips — push values into the search

These replace typing for the five common queries. Power BI 2025 buttons support a **Set slicer** action:

1. Insert → **Button → Blank** for each chip ("OTP", "Delay", "Fuel uplift", "GLP code", "Crew roster").
2. Format → Button → **Action**:
   - **Type** → `Set slicer`
   - **Destination** → pick the search slicer you built in §3.
   - **Value** → the literal search term (e.g. `otp`).

No bookmarks. Clicking the OTP chip pushes "otp" into the search slicer, which cascades through every visual.

---

## 5. The keyword cloud — Microsoft Word Cloud visual

1. Visualizations pane → **Get more visuals** → AppSource → search "Word Cloud" (publisher: Microsoft). Install.
2. Insert the visual onto the left column of the main grid.
3. Drag `DashboardDirectory[Keywords]` into **Category** and the count of rows into **Values** (or let it auto-compute frequency).
4. Format → Data colors → use your theme palette.
5. Format → **General → Interactions**: ensure the visual is an interactor, not just a receiver.

**Clicking a word in the cloud now acts as a cross-filter on the whole page** — same effect as clicking a pill or typing a search term.

---

## 6. The pinned tiles — three buttons with URL actions

These aren't filters — they're direct links to three specific dashboards.

1. Insert → **Button → Blank**. Name label = dashboard name.
2. Format → Button → **Action**:
   - **Type** → `Web URL`
   - **Web URL** → paste the Power BI link to that dashboard.
3. Duplicate × 3. Position them in the banner's middle column.

Optionally turn them into a small slicer on a `PinnedBy[UserEmail]` table filtered by `USERPRINCIPALNAME()` if you want per-user pinning.

---

## 7. The dashboard list — matrix with Web URL column

Unchanged from the original tutorial. Because this matrix sits on the same page as the tile slicer and the search slicer, it receives both filters automatically. No wiring needed.

Keep these formatting decisions in 2025:

- **Style preset**: None
- **Row padding**: 8 px
- **Grid → Horizontal**: `#E8DFD0`, 1 px; **Vertical**: off
- **Totals**: off
- **Conditional formatting on `DashboardName` → Web URL** → base on `URL`
- **Conditional formatting on `DashboardName` → Font colour** → static `#43061F` (theme overrides the default blue)

---

## 8. The stats cards — use the new Card visual (2024+)

The new Card visual (the multi-row card, not the legacy one) handles multiple measures cleanly and picks up theme colours without manual styling.

For **"Directory usage — last 30 days"**:

1. Insert → **Card** (the new one — it has a slightly different icon in the Visualizations pane).
2. Drag `SUM(DirectoryViews[Views])` as the big number.
3. Format → **Callout value** → font Segoe UI Semibold 32 pt, colour `#2A0311`.
4. Format → **Reference label** → text "views", font colour `#B8893A`, italic.
5. Below the card, place a clustered column chart on `DirectoryViews[ViewDate]` vs count; format bars with theme accent.

For **"Top areas this week"**:

1. Insert → **Donut chart** on `DashboardDirectory[Category]` (values = view count).
2. Legend OFF. Detail labels OFF. Use a theme-coloured gradient.
3. Place a small Card or text box next to it with the top three percentages.

Both cards filter down when a category is selected — no wiring needed, they share the `DashboardDirectory` model.

---

## 9. Cross-filter behaviour — the only knob to touch

If you find one visual filtering another in a way you don't want (e.g. clicking a word in the cloud also filtering the banner search), use **Format → Edit interactions** (top ribbon). For each source visual, pick None/Filter/Highlight per target:

| Source | Target | Interaction |
|---|---|---|
| Tile slicer (Category) | Matrix, Cloud, Count, Stats | **Filter** |
| Tile slicer (Category) | Banner search slicer | **None** |
| Search slicer (Keywords) | Matrix, Cloud, Count, Stats | **Filter** |
| Search slicer (Keywords) | Tile slicer | **None** |
| Word Cloud | Matrix | **Filter** |
| Word Cloud | Tile slicer | **None** |
| Pinned-tile buttons | (all) | **None** — these open URLs, they don't filter |

Set once, done forever.

---

## 10. Measures you still need

```DAX
ResultsCount =
VAR _cat = SELECTEDVALUE(DashboardDirectory[Category], "All areas")
VAR _n   = COUNTROWS(DashboardDirectory)
RETURN _n & " dashboards in " & _cat

-- new in PBI 2025: dynamic format strings for measures
-- makes the count card auto-switch between singular/plural
ResultsCountFmt =
VAR _n   = COUNTROWS(DashboardDirectory)
RETURN _n
-- format string (paste into the measure's Dynamic format string box):
-- IF ( [ResultsCountFmt] = 1, "0 dashboard", "#,##0 dashboards" )
```

---

## Summary — what clicking an area actually does

With the setup above:

1. User clicks **"Flight Ops"** pill (= Tile slicer value).
2. `DashboardDirectory[Category] = "Flight Ops"` propagates to every visual on the page.
3. Matrix rows shrink to 6. Word cloud re-weights to Flight Ops keywords. Count card says "6 dashboards in Flight Ops". Usage card shows Flight Ops-specific totals. Donut chart highlights the Flight Ops slice.
4. User types "otp" in the search slicer → filter stacks on top of the category filter. Matrix narrows to 1 dashboard.
5. User clicks "All Areas" pill (the Select-all option) → category filter clears; search filter still active.
6. User clicks the ✕ on the search slicer → everything resets.

All of this with **zero bookmarks**, one slicer per filter, and native cross-visual interactions.
