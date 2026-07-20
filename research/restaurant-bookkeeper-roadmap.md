# Restaurant Bookkeeper — Product Roadmap & Merge Plan

Working document for the Restaurant Bookkeeper platform (Signal F portfolio).
Consolidates the gap analysis between the two existing codebases and sets the
build order for upcoming development runs. Last updated: 2026-07-19.

## Scope Decision Log

| Date | Decision |
|---|---|
| 2026-07-19 | **Payroll execution is OUT of scope.** No money movement, no direct deposit origination, no tax deposits, no filings. Restaurants keep their third-party payroll service (Gusto, ADP, Paychex). We **import** their payroll journal reports and post the entries — the same relationship most independent bookkeepers have with payroll today. This avoids Money Transmitter Licensing, NACHA obligations, and filing liability entirely. |
| 2026-07-19 | Production platform base is the `restaurant-bookkeeper-in-a-box` repo (double-entry engine, tenancy, billing, compliance calendar). The Python `restaurant_accounting_poc_v5` prototype is the **feature specification** to port from — not a second codebase to maintain. |
| 2026-07-19 | Target backend: evaluating Powabase (Postgres + agent runtime + workflows + OCR/RAG) as the landing platform, replacing the Hatchable runtime. Decision pending real access to Powabase docs. |

## Positioning

Anti-QuickBooks-bloat: a bookkeeping platform that does **only** what a food &
beverage operation needs, end to end, with an AI agent orchestrating the
recurring workflows. No accounts receivable, no perpetual FIFO inventory, no
multi-currency, no generic invoicing. Value proposition: replaces the outside
bookkeeper and the month-end cleanup, produces a tax-ready package at year end.
Payroll stays with the customer's existing payroll service; we record it.

## Current State — Two Complementary Codebases

### Production repo (`chrisfbaileycb-arch/restaurant-bookkeeper-in-a-box`)

The platform. JS/Hatchable runtime, central Postgres.

- Multi-tenant model: organization → location → org_user → workspace, single
  isolation chokepoint (`lib/tenant.js getContext()`), plan-tier enforcement
  (Single $149 / Group $249 / Premium Group $499)
- Double-entry ledger with all-or-nothing balance validation; balances always
  computed from journal lines (audit-safe)
- 42-account flat restaurant COA copied per organization
- Strict 16-column POS CSV contract: sanitization, per-location dedup,
  cadence watchdog
- Reports: P&L (food/bev cost KPIs), balance sheet (integrity check), COGS,
  CSV export
- Check reconciliation: register vs. cleared matching, mismatch statuses,
  discrepancy report, reconciliation summary tied to ledger cash
- Compliance calendar: CO (DR 0100, DR 1094, FAMLI, UITR-1) + federal
  (941/940) with due dates, statuses, estimated amounts from liability
  balances, daily cron
- QuickBooks export bridges (QBO journal CSV, IIF), month-scoped
- Stripe billing paywall, 428-disclaimer acknowledgment pattern for
  high-stakes actions

### Prototype (`restaurant_accounting_poc_v5.py`, Google Drive)

The feature spec. Python/SQLite, verified working end-to-end pipeline.

Modules the production repo does NOT have (port order below):

1. **AP subledger** — invoice ingestion with rule-based line categorization
   to COGS child accounts, aging bins (0–15/16–30/31+), payment settlement
   with optional check linkage
2. **Third-party delivery reconciliation** — DoorDash/UberEats/Grubhub payout
   statements: gross sales, commissions, promotions, refunds, net payout,
   with correct journal treatment (refunds as contra-sales)
3. **Payroll journal import** — Gusto/ADP report ingestion: BOH/FOH wage
   split, employer taxes, single ACH sweep entry (this stays IN scope;
   execution does not)
4. **Bank-deposit clearing matching** — CC settlements and safe drops matched
   to clearing accounts, merchant fee isolation (extends the existing
   checks-only matcher)
5. **Physical inventory adjustments** — periodic counts posted as COGS
   corrections against inventory asset accounts
6. **Prime cost metrics** — COGS% + labor% with 65% threshold warnings
   (depends on payroll import)
7. **Hierarchical COA with tax-line mapping** — parent/child rollups
   (5000/5010 COGS families) and per-account 1120-S / 1125-A / Sch L line
   mapping — the seed of the year-end tax package
8. **Per-POS parsers** — Toast/Clover/Square daily summary formats, cash
   drawer over/short, till payouts, gift-card deferred revenue
9. **Comparative period statements**

Known prototype simplifications to fix during porting:

- "OCR" invoice scanning is a keyword parser over pre-structured text (real
  OCR is an integration, likely Powabase's pipeline)
- "Plaid" matching is a CSV simulation on description keywords (real Plaid is
  an integration: OAuth, tokens, webhooks)
- POS food/beverage splits are hardcoded ratios (80/20, 75/25) — must come
  from actual POS category data
- Denormalized `balances` table can drift — keep production's
  compute-from-journal approach
- Parsers lack validation (`float()` on raw input) — port onto production's
  all-or-nothing validation pattern

## Build Order

### Phase 1 — Engine parity (port prototype features into production repo)

STATUS 2026-07-20: **COMPLETE** — all seven items built as stacked draft PRs
on Restaurant-Bookkeeper-In-A-Box (#1 → #6, merge in order). Migrations
0006–0011; each module has functional tests (balanced-entry + rejection
paths) pending the in-repo test suite.

1. ✅ COA migration: hierarchical accounts + `tax_line` column (PR #1)
2. ✅ AP subledger: categorization rules + aging + payment recording (PR #2)
3. ✅ Payroll journal import + prime cost KPIs — feeds compliance-calendar
   liabilities; net-pay reconciliation enforced (PR #3)
4. ✅ Delivery platform reconciler — enforced payout identity, contra-revenue
   refunds, take rates (PR #4)
5. ✅ Bank feed matching — deposit clearing with merchant-fee isolation,
   check routing, unmatched queue never auto-posts (PR #5)
6. ✅ Inventory adjustments + comparative statements (PR #6)
7. ✅ POS daily-summary normalizers — real category splits, drawer
   over/short; v5's hardcoded 80/20 ratio deliberately NOT ported (PR #6)

### Phase 2 — Platform move (pending Powabase evaluation)

- Runtime adapter: `hatchable` SDK gateway → Postgres client + platform auth
- RLS on the (already RLS-ready) schema
- Cron → scheduled workflows
- Billing: align `lib/billing.js` (flat $149) with the three plan tiers
  defined in `lib/tenant.js`

### Phase 3 — The agent layer

- Expose engine functions as agent tools: post entries, run reports,
  reconciliation, compliance status, AP aging
- Recurring orchestration: daily sales posting, weekly delivery + payroll
  imports, bank matching, month-end close checklist
- Human approval gates on all posting actions (reuse the 428-ack pattern)
- Document ingestion: real invoice/receipt OCR via platform pipeline
- Alerting: prime cost threshold, compliance due dates, cadence watchdog
  misses, reconciliation differences

### Phase 4 — Integrations & polish

- Plaid (real): bank feeds replacing bank CSV import
- Year-end tax package export (tax-line rollups from Phase 1 COA work)
- Multi-state tax matrix (production is Colorado-only today)
- Mobile companion (mockups exist; build after web console matures)

## Platform Candidates (Phase 2 decision)

Three candidates under evaluation. All three sites are unreachable from the
dev environment (network policy); knowledge is from search results and
third-party reviews — verify hands-on before committing.

### Powabase — integrated AI backend

Postgres + pgvector, auth, storage, OCR/RAG pipeline, in-platform agent
runtime, visual workflows. Strongest on document ingestion (invoice OCR) and
hosted agent orchestration. Unknowns: pricing, maturity, lock-in.

### AppDeploy (appdeploy.ai) — chat-native web runtime

MCP-native deploy target: Claude Code deploys and iterates directly. Managed
hosting, database, auth, file storage, secrets, scheduled jobs, real-time
sync, custom domains, version rollback, code export (no source lock-in).
Closest analog to the current Hatchable runtime, with the key advantage that
the agent workflow (Claude via MCP) is first-class. Positioned for internal
tools / lightweight SaaS; flagged weak for compliance-heavy production.
Must verify before adopting: raw SQL/migration control (our migrations are
plain Postgres), multi-tenant isolation/RLS, Stripe webhook signature
verification, database (not just code) export, production SLAs.

### Expo / EAS — the mobile companion (Phase 4)

Not a backend candidate; it is the answer to the dual-interface vision's
mobile half. React Native + EAS build/submit/update, official MCP server and
documented Claude Code integration (docs.expo.dev/agents/claude). Industry
standard; low-risk default choice for the mobile app whenever Phase 4 starts.

### Working conclusion

AppDeploy and Expo are complementary (web runtime + mobile app), not a
Powabase replacement pair. If the agent is Claude-over-MCP rather than an
in-platform runtime, Powabase's remaining differentiator is the OCR/RAG
pipeline — which could instead be a point service (document-AI API) behind
our own endpoints. Decision path: trial-deploy a slice of the production app
to AppDeploy via MCP and test the five verification items above; keep the
engine portable (pure lib/ + thin db gateway) so no candidate becomes
load-bearing before it earns it.

### Evaluation criteria (any backend candidate)

1. Agent workflow: MCP/tool access for deploy + iterate, scheduled triggers,
   notification hooks
2. Postgres access model: raw SQL migrations, RLS, connection pooling
3. Auth service: fit with the org_user/workspace model
4. Document pipeline (or clean slot for an external OCR service)
5. Pricing at 10/100/1000 locations; production support tier
6. Data export (code AND database) / lock-in posture

## Open Items

- Master proposal Google Doc not accessible via connector (ID returns
  not-found) — re-share needed
- Five dashboard mockup PNGs not yet reviewed against the build order
- Embedded payroll research retained for reference only (out of scope per
  decision log)
