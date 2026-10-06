# QA report — index.html (finance_pro) — 2026-10-06

Metadata: branch `main` (dirty slate), caller `/qa-only` (report-only), mode Full, driver: gstack headless browse (`$B`) — Aside unavailable (`NEEDS_ASIDE`), file:// origin, localStorage profile, run UTC 2026-10-06T11:20Z, budget 900s (351s remaining at wrap — not time-limited).

Charters: see `exploration-001..003.json` and opening section of this run dir. Contracts were tested per the planning session; no test-plan file exists in repo.

## Findings (severity · category)

### F1 — High · Content/Functional: phantom targets render on a zeroed slate
Fresh boot shows stored settings `{savingsRateTarget:0, emergencyMonthsTarget:0, alertDays:0}`, yet:
- Dashboard KPI renders "Savings Rate 0.0% — **20% target**" and "Emergency Buffer 0% — **0.0 of 6 months covered**".
- Settings → goals form inputs render `savingsRateTarget="20"`, `emergencyMonthsTarget="6"`, `alertDays="5"`.
- After demo load, dashboard renders "above **20% target**".
Root cause: `|| 20` / `|| 6` / `|| 5` fallbacks (index.html lines 753–754, 755, 1065, 1197, 1684, 1704–1705, 2906, 2915, 2924, 3712–3714): `0 || 20 === 20`. Client sees numbers it never set.
**Value: protects=zero-default client build; fails_when=any `|| 20/6/5` fallback is restored; why_new=smoke.mjs never reads rendered target text.**

### F2 — Medium · Visual/Console: `chart-line` icon not found
Console warning (5 occurrences across demo run): `<i data-lucide="chart-line">` — icon name not in bundled lucide set. Renders as empty glyph. Replace with a valid icon name (e.g. `line-chart`/`trending-up`) or add the icon.

### F3 — Known/Expected (from audit, not fixed yet): `BASE_CATEGORIES()` budgets, demo seed
Category list renders "no limit" everywhere and Total Allocated $0.00 on a fresh slate, so the hardcoded `budget:600…1850` values in `BASE_CATEGORIES()` (index.html ~506–517) do not currently leak to the UI — but they are still written into source and resurface anywhere a raw category budget is read. Demo seed remains opt-in only (verified: no auto-seed, Load Demo Data → 125 tx / 7 accounts, merge/replace both work, factory reset returns to 0/0 and checkbox unchecked).

## Passed contracts
- A1/A2: fresh boot zero tx/accounts/bills/goals; 17 baseline categories; only tailwind-CDN console warning.
- All 14 views render on empty slate without errors; totals $0.00 across dashboard/today/calendar/accounts/transactions/income/expenses/budgets/spending/bills/subscriptions/billcalendar.
- Explicit demo load and atomic factory reset verified (exploration-003, L-series assertions in `tests/smoke.mjs` consistent).
- Category edit form budget field blank; no raw budget leak in UI.

## Health scores
- Console: 70 (1 deduped warning class)
- Links: 100 (no broken same-origin targets; single-page anchors only)
- Visual: provisional 90 (no layout pass beyond screenshots/initial + after-demo)
- Functional: 60 (F1 is a shipped user-facing defect despite zeroed DB)
- UX: provisional 92
- Performance: not measured
- Content: 70 (F1 phantom numbers)
- Accessibility: not exercised (no keyboard/screen-reader pass)
Weighted composite (tested categories only): ~76.

## Coverage limits / not run
- No automated browser-level regression added (report-only). Proposed: smoke check asserting rendered savings-rate target text matches stored settings when 0.
- Mobile viewport pass not run.
- Demo N8 assertion (125 tx) matches.
- No data exported or settings changed on any real profile beyond owned file:// localStorage of this headless session.
