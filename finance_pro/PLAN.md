# finance_pro — Stabilization & Customer-Readiness Plan

Goal: kill the "random values" problem (numbers that drift, disagree, or phantom-in) and make the app practical for real customers.

## Diagnosis (verified by reading the code)

1. **Paid bills never stick.** `seedData()` records paid periods as `${y}-${m}` (e.g. `2026-9`) but `dayKey()` produces padded keys (`2026-09`). Every seeded "paid" bill renders as unpaid/overdue. KPIs ("Paid This Month"), the Today checklist and Needs-Attention disagree.
2. **Monet recharge does not reconcile.** Seeded account `balance` values are arbitrary fixed numbers, yet `applyTxToBalance()` mutates balances on every add/edit/delete. Net worth therefore drifts by whatever you do, not by your actual ledger.
3. **Phantom payday.** The Today "7-day cashflow" injects `+$5200` on every 1st of the month in addition to the real day-1 payroll transaction → double-counted, imaginary income.
4. **Time-dependent seed.** `seedData()` truncates the current month to today, and if `localStorage` is unavailable/blocked the whole thing re-runs (or data can't persist) with no notice — "the app shows different numbers every visit".
5. Migration hazard: legacy localStorage data has no migration path for the key-format fix; legacy accounts have no opening-balance field.

## Part A — Ledger-consistent data model (ME / agent 1)

- Single source of truth for paid-period keys: normalize all `paidPeriods` via one helper; add a one-time migration in `DB.load()`/`normalize()` to rewrite legacy `2026-9`-style keys to `dayKey()` format.
- Accounts gain `openingBalance`. `syncBalances()` recomputes every `a.balance = openingBalance + Σ movements(txns)` and is called after seed, import, and every transaction mutation. `applyTxToBalance()` is removed.
- Seed derives implicit `openingBalance` per account as `seededBalance − Σ seededMovements(account)`, so first paint keeps today's demo numbers but they now reconcile exactly with the ledger.
- Legacy accounts: `openingBalance = balance − Σ movements` (preserves displayed balance, makes future edits consistent).
- Today timeline: remove the hard-coded payday; projected inflow = real recorded income only; outflow = unpaid bills + subscription renewals.
- Node syntax check (`node --check` on the extracted `<script>` block) after edits.

## Part B — First-run experience & honest empty states (agent 2)

- First-run modal (flag `PFM_PRO_ONBOARDED`): "Start fresh" (empty data) / "Load sample data" (current seed) / "Import a backup". No demo data silently injected anymore.
- Factory reset dialog gains the same three choices (default: empty).
- Dashboard "Get started" checklist when data is minimal (add account, first income entry, first budget, first bill).
- Audit empty states in every view (transactions, income, expenses, budgets, spending, bills, subscriptions, calendar, accounts) — every list must render a proper emptyState, not a blank section.

## Part C — Persistence honesty & metric window consistency (agent 3)

- Detect storage failures: banner + persistent non-blocking warning chip when `localStorage` writes fail (explains why numbers change on reload); still allow in-memory use + export.
- `DB.reset()` honors "start empty" vs "sample data".
- Audit metrics that mix "current period" vs "now" (attentionItems budget check uses `UI.month` while bills use now — use a single cursor consistently); fix the small inconsistencies found, document the rule.
- `daysElapsedInPeriod`/`periodDays` edge cases for the budget cycle start day >1.

## Part D — Supervision (agent 4)

- After A+B+C land: extract the script block, `node --check`, grep-audit that no `applyTxToBalance`, unpadded `paidPeriods` keys, or hard-coded `5200` remain, verify seed reconciliation (`Σtxns` vs balances) with a small node harness, then write `finance_pro/REVIEW.md` listing any remaining defects. Do not restructure; only fix trivial syntax slips.

## House rules for all agents

- Edit only `finance_pro/index.html` for code work (plus your assigned doc files). Do not touch `life planner/`.
- Preserve the existing design system, Tailwind classes, and delegation model (`data-action`/`data-form`). No new dependencies, no frameworks.
- Keep every change atomic and report: what you changed, where (line anchors), and anything you deliberately left alone.
- After each part: extract `<script>` and run `node --check`.
