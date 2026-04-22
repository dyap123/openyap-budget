# FORMWORK Devlog

## 2026-04-21 — Sankey Plan, scenarios, Cmd-K, health gauge, quieter skin

A deeper overhaul aimed at "immersive, cleaner, more minimal, nicer animations and buttons" — and killing the Excel-grid feel of the Plan view. Same single-file deploy, same Firebase, no build step.

### Plan view is now a Sankey flow
- HTML table replaced with a D3 + d3-sankey diagram at `renderPlanSankey()`. Nodes are `Source → Month → Bay`; link width scales with planned $ from the active scenario. Income categories become left-column sources; if none exist, a synthetic `Reserve` node holds the flow.
- Hover a band → floating glass tooltip (`.sankey-tooltip`) with `from → to` and value. Click a band → opens the existing side panel (`openCategoryPanel(catId, monthKey)`) pre-filtered to that month. Click a month node → `openMonthPanel()` opens the first bay at that month.
- Links reveal with a staggered stroke-dashoffset tween (~900ms, 8ms stagger); respects reduced-motion.
- Gradient strokes per link (source color → target color) via `<linearGradient>` defs. Node labels sit outside the bars with a sub-line showing the $ amount.
- D3 + d3-sankey pulled from jsDelivr; pure additive to existing CDN set.

### Scenario / what-if mode
- New Firebase path: `budget/scenarios/{id}` → `{ name, createdAt, plan, planItems }`. Base scenario remains at top-level `budget/plan` + `budget/planItems` (no migration).
- Active scenario persisted at `budget/settings/activeScenario` (default `'base'`). All plan reads and writes now go through `planPath()` / `planItemsPath()` helpers.
- Pill group in the top bar: `Base · [user scenarios] · +`. Active pill is highlighted; `+` prompts for a name and clones the current active plan into a new scenario. User scenarios get an `×` to delete (can't delete Base).
- Listeners are rebound (`bindScenarioListeners()`) whenever the active scenario changes, so every view (Plan/Outlook/Compare/Dashboard/Nodes/Panel) reflects the switch with no extra plumbing.

### Command palette (⌘K / Ctrl-K)
- Glass modal with fuzzy scorer (prefix bonus, word-boundary bonus, subsequence fallback). `↑/↓` to navigate, `Enter` to invoke, `Esc` to close.
- Action groups: Go to (7 views), Create (transaction / line item), Scenario (clone, switch to any), Export (.xlsx / quick), Toggle (reduced motion, accent).
- Recent-actions memory: top-5 IDs persisted to `localStorage` under `fw.cmd.recent`; shown as a "Recent" group when the palette opens empty.
- ⌘K hint badge wired into the top bar next to notifications.

### Health gauge
- Animated SVG ring gauge on the Dashboard header. Score 0–100 = weighted blend of `runwayMonths/12` (40%), `1 − overPlanCategories/totalCategories` (30%), and `committedOrDoneItems/upcomingItems` over the next 30 days (30%). Color gates: red <40, amber 40–70, accent ≥70. Stroke-dashoffset tweens in on first paint.
- Re-renders whenever transactions, plan, or line items change (monkey-patched `renderTable` hook).

### Polish layer
- **Unified buttons** — single `.btn` + `.btn-primary` / `.btn-ghost` / `.btn-soft` / `.btn-icon` set; every ad-hoc button markup across Plan / Expenses / Nodes / Settings / Modal swept over to the new system.
- **Toasts** — `toast(msg, kind)` helper (info / success / warn / error). Wired into: plan cell save, line-item add/update/delete, transaction save/delete, scenario switch/create/delete, settings save, accent change. Stack bottom-right, 2.8s auto-dismiss.
- **Skeleton** — `.skel` class with a slow 1.4s shimmer (disabled under reduced motion). Ready to drop over any pre-Firebase surface.
- **Empty states** — `.empty` pattern used in: Plan (no data / all-zero), Transactions (no log / no matches), Command palette (no fuzzy hits).
- **Favicon + OG** — inline SVG favicon (lattice + dot on a rounded dark square). Open Graph + Twitter card meta tags; `og.png` (1200×630) committed alongside `index.html`.
- **Boot animation** — 950ms overlay on first visit per session: lattice mark draws in (stroke-dashoffset), wordmark fades up, then fade out. Session-gated via `sessionStorage`, skipped under reduced motion.

### Quieter / more minimal skin
- Accent glow alpha `0.35 → 0.18`; `--accent-glow-soft` `0.05 → 0.03`. New `--surface-quiet` for tonal range.
- Ambient WebGL: particle count `120 → 55`, opacity `0.75 → 0.40`, rotation/drift speeds halved. Shader mix-amount dialed down (`0.20 → 0.13`).
- Three.js Nodes + Dashboard bay rotation speeds halved (imperceptibly slower, much calmer feel).
- View headers: `text-5xl → text-6xl`, breathing paragraph beneath, kicker line in accent. Section paddings `p-12` / `gap-12`.
- Glass panels: softer shadow (`glow-box` toned down), `1px` accent border instead of heavy drop shadow.
- Top bar breadcrumb: accent-colored "FORMWORK" chip, smaller type, greater letter-spacing for structure.

### Files
- `index.html` — single file, everything above
- `DEVLOG.md` — this entry
- `og.png` — new static asset (1200×630)

## 2026-04-20 — Formwork rebrand, line items, Compare view, 3D lattice

Full overhaul of the OpenYap Budget app. Name retired from `Lumino` → `Formwork` (construction-native: the mold concrete is poured into — the structure that shapes every dollar). Planning goes deeper, comparison goes wider, the whole thing sits on a live WebGL stage.

### Rebrand
- All `LUMINO` strings swapped for `FORMWORK` (title, sidebar wordmark, breadcrumb, export headers, filenames)
- Sidebar icon: `lightbulb` → `grid_view`; added faint 3×3 lattice motif behind the wordmark
- Export filename: `lumino-budget-*.xlsx` → `formwork-budget-*.xlsx`

### Line-item planning (per-category drill-in)
- New Firebase path: `budget/planItems/{catId}/{itemId}` → `{ name, amount, month, status, notes, order, ts }`
- Legacy `budget/plan/{catId}/{YYYY-MM}` preserved. `planCell()` now sums line items when present and falls back to the legacy number otherwise
- Click any category row in Plan (or a category card in Dashboard/Nodes, or a row in Outlook/Compare) → right-side slide-in panel (520px glass) with: summary strip, month filter, inline-editable items table, add/delete, cumulative planned-vs-actual sparkline
- Plan matrix cells with items are non-editable and show a dot indicator; click to drill in. Direct edit on an item-backed cell prompts before overwriting
- Statuses: `planned` / `committed` / `done` with matching color pills

### Compare view (new top-level tab)
- Four tiles: bays, line items, total planned, over-plan count
- Stacked-area "Plan Flow" — one layer per category, full 12-month horizon, with legend
- Sortable Category-vs-Category table: planned, actual, var $, var %, % used bar, item count, biggest line item
- Flat cross-category line-items list with filters (name/notes text, status, category) and sortable columns

### Three.js immersion
- Fixed full-viewport ambient canvas behind everything: gradient-noise shader mesh + 120 drifting accent particles + slow-rotating wireframe octahedron. 45 fps cap, pauses on `visibilitychange`, disabled when reduced-motion toggle is on
- Nodes view: flat HTML orbit replaced with a 3D formwork lattice. Each category is a wireframe rectangular prism (a formwork bay) arranged on a spiral hemisphere, sized ∝ planned total. Actual spend pours in as a translucent fill inside the bay. Line items = glowing studs. Hover → floating tooltip with planned/actual/item-count. Click → drill-in panel
- Dashboard topology: small embedded Three.js scene — same lattice, auto-rotating

### Dashboard additions
- Plan Health card: over-plan count + no-items count
- Next 30 Days card: sum of open line items (status ≠ done) due in the next 30 days

### Settings additions
- Accent picker: cyan / violet / amber — writes `settings.accent` and updates every CSS `var(--accent)` plus WebGL colors live
- Reduced-motion toggle: kills the ambient canvas, disables rotations, honors `prefers-reduced-motion`

### Data model (final)

```
budget/
  categories/{catId}        → { name, type, icon, color, order }
  transactions/{txId}       → { description, amount, date, categoryId, status, notes, ts }
  plan/{catId}/{YYYY-MM}    → amount (legacy, still read)
  planItems/{catId}/{itemId}→ { name, amount, month, status, notes, order, ts }
  settings                  → { companyName, currency, fiscalYearStart, seedTarget, accent, reducedMotion }
```

### Tech notes
- Still single-file. `~/openyap-budget/index.html` now ~1600 lines
- Three.js r160 via CDN (one tag, no build)
- Two independent WebGL contexts: global ambient + Nodes lattice + Dashboard mini. All dispose-safe, all pixel-ratio clamped to 2
- Keeps Firebase RTDB as sole source of truth; no Sheets integration (unlike CUP)

---

## 2026-04-19 — Plan, Outlook, Excel export

Shifted LUMINO from a pure transaction ledger toward a real budgeting tool: you now build a conceptual monthly budget, then your filed expenses compare against it.

### What got added

**Plan view** (`/plan` nav item)
- Matrix: categories × 12 months, one editable dollar cell per intersection
- Values persist to Firebase RTDB at `budget/plan/{catId}/{YYYY-MM}` — zero means "no entry" (cell cleared)
- `Earlier` / `Later` buttons shift the 12-month window ±6 months at a time; starting month derives from `settings.fiscalYearStart`
- Top strip: Total Planned (12-mo), Avg Monthly Burn, % of Seed Target
- Row + column totals update live on blur

**Outlook view** (`/outlook` nav item)
- Four summary tiles: Total Planned, Total Actual, Variance, % of Plan Used
- Cumulative burndown SVG chart — planned line (blue, filled gradient) vs actual line (pink with dots), dashed "Today" marker, auto-scaled Y-axis
- Category variance table: Planned · Actual · Variance · % Used bar (amber >85%, red >100%)
- Month range spans from fiscal start through whichever is later: end of 12-mo plan horizon, or latest-dated actual transaction

**Excel export** (button on Plan view)
- SheetJS (`xlsx.full.min.js`) via CDN — no build step needed
- Downloads `lumino-budget-YYYY-MM-DD.xlsx` with 4 sheets:
  - `Summary` — company/seed/generated-at + overall planned/actual/variance
  - `Plan` — the 12-month matrix as shown in the app
  - `Outlook` — per-category planned/actual/variance/% used
  - `Transactions` — raw actuals sorted by date
- Currency columns use `$#,##0.00` format; % columns use `0.0%` — Excel keeps them as numbers, not strings

### Data model

```
budget/
  categories/{catId}   → { name, type:'expense'|'income', icon, color, order }
  transactions/{txId}  → { description, amount, date, categoryId, status, notes }
  plan/{catId}/{YYYY-MM} → amount (number)
  settings             → { companyName, currency, fiscalYearStart, seedTarget }
```

`plan` is new. Categories + transactions unchanged — outlook derives actuals from transactions by filtering on `categoryId` and bucketing by `date.slice(0,7)`.

### How the comparison works

- **Planned per category per month** = direct lookup in `plan[catId][monthKey]`
- **Actual per category per month** = sum of `transactions[].amount` where `categoryId` matches and `date.slice(0,7) === monthKey`
- **Cumulative curves** = running sum over the month range; actual curve stops at "today" so the gap between it and the planned line shows where you're tracking ahead/behind
- **Variance** = planned − actual (positive = under budget, negative = overspend)

### Tech notes

- Single-file app, no build step. All new code lives in `index.html` (now ~1200 lines)
- Firebase RTDB path listener pattern: `fbListen('budget/plan', ...)` triggers re-render on any write from any client, so two browsers stay in sync
- Chart is hand-rolled SVG — no chart lib dependency
- Optimistic local update in `savePlanCell()` so totals repaint immediately without waiting for Firebase round-trip
