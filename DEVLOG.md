# LUMINO Devlog

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
