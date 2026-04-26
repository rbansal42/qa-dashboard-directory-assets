# Dashboard Detail Page — Build Guideline

Standards for creating a Dashboard Detail page in the Qatar Airways Dashboard Directory.
Brand-strict (QR Burgundy + Cool Grey + white). Based on the four reference mockups in this folder
(`qa-dashboard-detail-mockup.html`, `option1-image`, `option2-description`, `option3-compact`).

---

## 1. Purpose

A Dashboard Detail page is the landing surface a user reaches after clicking a tile in the directory.
It must let the user, in under 10 seconds, decide:

1. **Is this the right dashboard for my question?** — title, eyebrow, category, short description / "answers questions like" chips.
2. **Is the data trustworthy and current?** — owner, source, refresh cadence, SLA status, last refresh.
3. **What do I do next?** — Open dashboard (primary), Subscribe / Request access (secondary), or browse Related dashboards.

Everything else (usage chart, refresh history) supports steward conversations and operational trust.

---

## 2. Page Anatomy (top → bottom)

| # | Region | Required | Purpose |
|---|---|---|---|
| 1 | **Ribbon** | yes | Brand bar — "Qatar Airways · Dashboard Directory" + "Internal Use Only". `--ink` background, ~6 px tall. |
| 2 | **Action bar** | yes | `← Back to directory` left; Subscribe / Request access / **Open dashboard ↗** right. White background. |
| 3 | **Title block** | yes | Burgundy gradient banner. Eyebrow ("Dashboard Detail"), `<h1>` with one italic `<em>` highlight, and a category pill on the right. |
| 4 | **Meta bar** | yes | Six-column key/value strip on white: Owner · Department · Data source · Refresh · Last refresh · Status (SLA pill). |
| 5 | **Main grid** (1.1fr / 1fr) | yes | Left column: hero / description / compact card (variant choice — see §6). Right column: Usage card (90-day chart + 3 stats). |
| 6 | **Related dashboards** | yes | 3-up grid of `rel-item` cards, name + category + 30 d views. |
| 7 | **Refresh history** | yes | Last 10 runs table — Started · Duration · Rows loaded · Status. |
| 8 | **Footer bar** | yes | Page indicator (e.g. `Page 2 of 3 · Drillthrough`) and Power BI mark with last-refresh timestamp. |

The page sits inside `.report { max-width: 1280 px; margin: 0 auto }` — designed for a Power BI 1280 × 820 page but flows past 820 vertically.

---

## 3. Brand Tokens (do not override)

Copy this `:root` block verbatim into every detail page. The legacy variable names are kept so existing component CSS continues to work — only the values changed when we moved to the strict brand kit.

```css
:root{
  --ink:#3C0026; --oxblood-0:#3C0026; --oxblood-1:#64003A; --oxblood-2:#780044;
  --amber:#6B6C6E; --amber-dark:#5E6A71; --amber-light:#818A8F; --honey:#E1E1E2;
  --cream:#FFFFFF; --cream-warm:#F0F0F0; --cream-title:#FFFFFF; --white:#FFFFFF;
  --border:#E1E1E2; --border-mid:#E9E9E9;
  --text:#1A1A1A; --text-mid:#5E6A71; --text-light:#6B6C6E;
  --green:#2E7D4F; --red:#780044;
}
```

Mapping to brand names:
- `--oxblood-0/1/2` → QR Deep Burgundy `#3C0026` / QR Burgundy `#64003A` / QR Bright Burgundy `#780044`
- `--amber*` → Cloud Grey `#6B6C6E` / QR Grey `#5E6A71` / Oryx Grey `#818A8F` (cool grey accents — NOT amber)
- `--cream*` → white + Cloud Grey 10/15/20
- Semantic green stays `#2E7D4F`. Semantic "red" is `#780044` (Bright Burgundy) — we do not use a true red.

**Type stack** — body and UI:

```css
body{ font-family: 'Noto Sans', 'Graphik', 'Segoe UI', sans-serif; }
h1, h2, .stat-val, .hero-tagline { font-family: 'Jotia', 'Noto Sans', sans-serif; }
```

`Jotia` is the official QR display face; users without it fall back to Noto Sans. Italic `<em>` inside headlines is the brand voice — use it once per heading, not more.

Load fonts via:
```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Noto+Sans:wght@300;400;500;600;700&display=swap" rel="stylesheet">
```

---

## 4. Layout & Spacing

- Outer page padding: **40 px** left/right on every region (`action-bar`, `title-block`, `meta-bar`, `main`, `related`, `history`).
- Region vertical rhythm: title block `28 px / 22 px`, meta `18 px`, main `24 px`, related `0 / 24 px`, history `0 / 40 px`.
- Cards: `1 px solid var(--border)`, `border-radius: 6 px`, padding `20 px 22 px`, white background.
- **Never apply line/border to a card via `box-shadow` only** — keep an explicit border. The brand reads cleaner when surface boundaries are visible.
- Hero / compact image cards may add a soft burgundy shadow: `box-shadow: 0 4 px 14 px rgba(60,0,38,0.06–0.08)`. Don't go further.
- Main grid is `grid-template-columns: 1.1fr 1fr; gap: 24 px`. Left column gets the descriptive surface; right column is always the Usage card.

---

## 5. Component Patterns

### 5.1 Title block
- Burgundy gradient: `linear-gradient(135deg, #3C0026 0%, #64003A 55%, #780044 100%)`.
- A faint hex/circle SVG pattern overlays the gradient at 10 % opacity (already inlined in the mockups — copy the data-URI as-is).
- Eyebrow: 10 px, 0.3 em letter-spacing, uppercase, Oryx grey, with a 22 px leading rule (`::before`).
- `<h1>`: Jotia / Noto Sans, 36 px, weight 500. Use one italic span: `<h1>OTP &amp; <em>Delay Analysis</em></h1>`. The `<em>` colour is `--honey` (Cloud Grey 20%).
- Category pill: top-right, uppercase, translucent grey fill `rgba(225,225,226,0.2)` with `0.35` border.

### 5.2 Meta bar
- Six equal columns. If a field is unknown, render `—`, never collapse the column.
- `meta-k` = 9.5 px uppercase, letter-spacing 0.12 em, weight 700.
- `meta-v` = 13 px, weight 500.
- Status uses a pill (`.sla.ok` green / `.sla.stale` burgundy-red). Choose state from these rules:
  - `ok` — last refresh is within SLA window.
  - `stale` — last refresh exceeded SLA OR latest run failed.

### 5.3 Left column — choose **one** variant per page

| Variant | When to use | File reference |
|---|---|---|
| **Description** (`.card` with `.desc`) | Default. Steward has written 1–2 paragraphs + a short bullet list of "what's inside". | `option2-description.html` |
| **Hero image** (`.hero-card`) | Flagship / executive-facing dashboards where presentation matters more than longform copy. Image at least 1280 px wide, focal point centered. | `option1-image.html` and `qa-dashboard-detail-mockup.html` |
| **Compact** (`.compact-card`) | When you want both: a small image header + 2–3 sentences + chip row of "answers questions like…". Best for IOC / ops dashboards with tight scan needs. | `option3-compact.html` |

Pick by **content density**, not aesthetics. If steward gave you a paragraph and bullets, use Description. If they only gave you a 2-line summary, use Compact. If they gave you nothing, use Hero with a stock cabin/aircraft photo.

### 5.4 Right column — Usage card
- Title: `Usage · last 90 days <em>· 90 d</em>`.
- 460 × 180 viewBox area chart. Stroke `#64003A` (QR Burgundy). Fill: vertical gradient from `rgba(107,108,110,0.45)` to transparent. Keep `preserveAspectRatio="none"` so the chart fills the card.
- Three horizontal grid lines at y = 45 / 90 / 135, stroke `#E1E1E2`.
- Peak marker: 4 px circle, fill `#6B6C6E`, 2 px white stroke.
- Below chart: month-tick row (4 spans), then a 3-stat row (Views · 30 d, Unique users, Peak slot). `stat-val` is Jotia 22 px / 600, italic `<em>` for the small qualifier.

### 5.5 Related dashboards
- Always render exactly 3 cards. If fewer, leave the slot blank rather than padding with unrelated entries.
- Selection logic: same `category` first; then shared keywords from title + description; tie-break on highest 30 d views.
- Card hover: border → Cloud Grey, background → Cloud Grey 10%.
- Name link uses `--oxblood-1` underline. Category prefix is uppercased Oryx grey, 9.5 px.

### 5.6 Refresh history
- Last 10 runs only — ever. If fewer runs exist, show what's there and don't fake rows.
- Columns fixed: Started · Duration · Rows loaded · Status.
- Status colours: `.status-ok` green, `.status-fail` Bright Burgundy with the failure reason inline ("Failed · source timeout").
- Header row uses Cloud Grey 10% (`--cream-warm`) background, never burgundy.

### 5.7 Footer bar
- Page indicator left (`Page 2 of 3 · Drillthrough` or variant label).
- Power BI mark right: 10 × 10 yellow square `#F2C811` (the only non-brand colour on the page; this is Microsoft's mark and must stay yellow), italic note with timestamp.

---

## 6. Content Rules

- **Title** — sentence case, max 5 words, one italic `<em>` highlight on the noun: "OTP & *Delay Analysis*".
- **Eyebrow** — always literally "Dashboard Detail". Don't reword.
- **Category pill** — must match a directory category exactly (e.g. Flight Ops, Commercial, IOC, Crew Control). No new categories without a directory update.
- **Description copy** — 1–2 sentences max in Compact, 1–2 paragraphs + ≤4 bullets in Description. Plain English, no Power BI jargon ("dataset", "model") in user-facing copy. Refer to systems by their business name (AIMS, GoNow, Sabre) not a SQL hostname.
- **"Answers questions like" chips** — phrase as a question users actually ask. 3 chips, ≤6 words each. Avoid yes/no questions.
- **Owner** — single human name. If the dashboard has multiple owners, pick the steward; surface backups in Subscribe.
- **Refresh cadence** — show frequency + SLA: "Hourly (SLA 2 hr)", "Daily 02:00 GST (SLA 6 hr)".
- **Last refresh** — relative ("1 hr 30 min ago"), never absolute, in the meta bar. Footer carries the absolute timestamp.

---

## 7. Data Binding

The mockups are static HTML. When wiring to real data:

| Field | Source |
|---|---|
| Title, eyebrow, category | Directory metadata table (single row keyed by dashboard ID) |
| Owner, department | Directory metadata; fall back to Power BI workspace owner |
| Data source label | Free-text from steward; do **not** auto-derive from the dataset connection string |
| Refresh cadence + SLA | Steward-declared, stored alongside the dataset record |
| Last refresh, refresh history | Power BI REST API `/datasets/{id}/refreshes?$top=10` |
| Status pill | Computed: `ok` if `last_refresh + sla_minutes >= now()` AND `last_refresh.status == 'Completed'` else `stale` |
| Usage chart + stats | Power BI usage metrics (`Report views by Date`, `Unique viewers`, `Peak hour`) for last 90 d |
| Related | Pre-computed nightly: same category + keyword cosine similarity on title + description |

Render server-side or pre-bake at publish time. Don't fetch usage data on page load — it's slow and the steward conversations don't need second-precision freshness.

---

## 8. Accessibility

- All images on the page must have meaningful `alt` text. The Qsuite hero is decorative-ish but still describe the subject ("Qatar Airways Qsuite Business Class cabin").
- Title gradient + white text contrast ratio is ≥ 4.5 : 1 — verified for `#FFFFFF` on `#64003A`. Don't let designers darken the title text.
- The category pill on burgundy must use `--amber-light` (`#818A8F`) for text — that combination passes AA at 11 px when bold.
- Status pills must not be the only signal — the icon dot + text label is mandatory; never colour-only.
- Buttons keep visible focus rings (browser default is fine; do not `outline:none` without a replacement).

---

## 9. Build Checklist

Before shipping a new Dashboard Detail page:

- [ ] `:root` token block matches the canonical block in §3 exactly (no extra warm tones, no amber).
- [ ] Headline uses Jotia/Noto Sans stack with one italic `<em>` span.
- [ ] Body uses Noto Sans/Graphik/Segoe UI stack.
- [ ] Title gradient direction is `135 deg` and uses the three burgundy stops (not two).
- [ ] Hex pattern overlay on the title block uses `rgba(107,108,110,0.10)` stroke (cool grey, not amber).
- [ ] Meta bar shows all six fields; no field is silently dropped.
- [ ] SLA pill state matches the §5.2 rules.
- [ ] One left-column variant chosen — not two stacked.
- [ ] Usage chart strokes are `#64003A`; gradient stops are `#6B6C6E`.
- [ ] Related grid renders exactly 3 cards.
- [ ] History table is capped at 10 rows.
- [ ] Footer carries the Power BI yellow mark and an absolute last-refresh timestamp.
- [ ] No `Cormorant Garamond`, no amber hex (`#B8893A` etc.), no cream hex (`#FBF6EC` etc.) — verify with grep before commit.
- [ ] Page renders cleanly at 1280 × 820 (the Power BI page size) without horizontal scroll.

---

## 10. References

- Brand palette / type — [Qatar Airways VI Guidelines](https://qhub.qatarairways.com.qa/QatarSite/media/Department-Marketing/VSG-QR-Brand-Assets.pdf) (internal).
- Existing variants in this folder:
  - `qa-dashboard-detail-mockup.html` — hero image, full feature set
  - `qa-dashboard-detail-option1-image.html` — image-led
  - `qa-dashboard-detail-option2-description.html` — description-led
  - `qa-dashboard-detail-option3-compact.html` — compact hybrid
- Theme JSON for matching Power BI report visuals — `qatar-theme.json`.
