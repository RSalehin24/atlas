# AGENTS.md

## Project

**Personal Financial Atlas** — a Bangladesh-focused **total personal financial management app**
replacing a manual Excel system with a Flutter mobile app that serves as a single pane of glass
for the user's entire financial life. It tracks bank accounts, cards, transactions (mostly
captured via SMS parsing), DPS/FDR/savings certificates, investments, informal lending, loan
amortisation, and net worth. It auto-computes tax using data-driven NBR e-Return categories,
generates the wealth statement, and produces a ready-to-file return by July 1 each year.

The app is a **full financial dashboard**, not just a tax calculator:
- **Balance checking** — one-tap view of all account balances (savings, current, DPS, FDR, loan
  outstanding) across all banks and MFS providers
- **Total deposits & investments** — aggregate view of DPS, FDR, Sanchayapatra, shares, CFD, and
  other investments with accrued interest/profit
- **Net worth tracking** — assets minus liabilities, updated with every transaction
- **Cash flow analysis** — monthly income vs expense breakdown by category, spending trends
- **Loan & EMI tracking** — outstanding principal, amortisation schedule, remaining tenure
- **Informal lending** — money lent to/borrowed from people, tracked separately from bank loans
- **Tax year planner** — start-of-year estimate, monthly investment suggestions, refined by
  actual SMS/statement data throughout the year, ready by July 1
- **Admin panel** — year-scoped tax parameter config, no code changes when NBR updates rates

## Design principle

**Zero third-party dependency for your data.** This is a personal financial vault, not a SaaS
product. You own the file, you choose where to back it up, and the app works fully offline.
No Firebase, no cloud sync, no telemetry. Google Play is only a distribution channel — the app
does not rely on Google Play Services for any core function. The architecture assumes you may
eventually side-load or distribute outside the Play Store entirely.

## Stack

- Flutter (mobile, Android-first) — *or React Native, undecided*
- Drift (SQLite ORM) for local persistence — with sqlcipher for full DB encryption
- Biometric unlock (fingerprint/face) + passphrase fallback on app launch
- Admin panel required — tax parameters (slabs, rates, rebate %, instrument caps, minimum tax,
  filing incentives) are data-driven, not hardcoded. Admin activates one `TaxRateSet` per FY
  without app updates.
- Backup = encrypted export file, user-chosen destination (USB, NAS, cloud of their choice)
- [Other deps / state management / package name — fill in]
- [Architecture pattern / module division — to be decided next]

## Platform requirements

- Distribution through Google Play Store is the target for convenience, but the app must function
  identically when side-loaded or distributed from elsewhere. No hard dependency on Play Services.
- All sensitive data (transactions, account numbers, card data) encrypted at rest via sqlcipher.
- Features or library choices that could jeopardize Play Store acceptance must be flagged.

## Status

Schema is **frozen at v2**. Implementation has not started.
The **one authoritative schema** is `docs/database/Schema/Personal_Finance_App_Schema.md` (v2).
Read it before any coding. The `.txt` and `.mermaid` files are derived — the `.md` wins.
Do not change the schema without updating all three files and this file simultaneously.

## Architecture & module design (next step)

Before writing any code, the application must be divided into well-defined modules with
strict separation of concerns. The following must be decided and documented here:

- **Architecture pattern**: e.g. clean architecture (data/domain/presentation layers),
  feature-based modularisation, or a hybrid
- **State management**: Riverpod, Bloc, or other
- **Module breakdown**: how features map to modules, shared core module, shared data module
- **Dependency inversion**: data sources (Drift DAOs) behind repository interfaces, domain
  models separated from DTOs
- **Directory structure**: convention for lib/ layout (e.g.
  `lib/core/`, `lib/features/<name>/`, `lib/data/`, `lib/domain/`)
- **Navigation / routing**: GoRouter or similar, with module-scoped routes

This module design will be recorded in a new `ARCHITECTURE.md` file and the agreed structure
will be reflected back into this section.

## Domain context

- Account types behave differently: `Savings`, `Current`, `DPS`, `FDR`, `LOAN`.
- DPS/FDR/Savings Certificates accrue interest subject to government VAT (Bangladesh Bank
  regulated) — net interest ≠ gross interest, always compute both.
- Common transfer methods: NPSB, BEFTN, RTGS, EFTN, cash, mobile banking (bKash/Nagad/Rocket).
- `Lent` (informal lending to/from people) is tracked separately from formal bank `LOAN`s — don't
  merge them.
- Most transactions will originate from parsed bank SMS alerts, not manual entry. SMS format
  varies per bank and changes without notice — the parsing layer needs to degrade gracefully
  (log + flag for manual review) rather than silently drop unmatched messages.
- Tax computation is entirely data-driven. Nine config entities under `TaxRateSet` (slabs,
  thresholds, exemption rule, rebate rule, investment caps, surcharge, minimum tax, filing
  timeline, plus the per-FY seeded TaxSpendingHeads and Categories) live in the database.
  Admin activates one `TaxRateSet` per FY. The app computes tax (Section 78 A, slabs, minimum
  tax, surcharge, rebate, clawback, filing incentive/penalty) by reading the active config —
  no code changes needed when NBR updates rates.
- Bank statement PDF uploads reconciled against SMS-parsed transactions. The statement is the
  authoritative source: statement values override SMS values. Unmatched statement lines create
  new transactions; unmatched SMS transactions are flagged for review.
- Rebate-eligible investments must be held until maturity (Finance Act 2026). Early withdrawal
  forfeits the rebate — added as tax in the withdrawal year.

## Data model conventions (don't deviate without flagging it first)

- Money is stored as `INTEGER` in minor units (poisha = taka × 100). Never `REAL`/float for
  monetary values. Format to taka only at the UI layer.
- `Card.cvv` is never persisted, full stop — even if a feature request implies storing it.
- `Transaction` is the central ledger entity. Monthly summaries, net worth, category breakdowns
  are SQL queries over `Transaction` + `Account` — not separate hand-maintained tables.
- An internal transfer between two of the user's own accounts is two `Transaction` rows (a debit
  and a credit) linked via `linked_transfer_id`, not a distinct `Transfer` entity.
- `Attachment` is one polymorphic table (`owner_type` + `owner_id`) shared by `Account`,
  `Investment`, `Lent`, `Transaction`, and `SavingsCertificate`. SQLite can't enforce that as a
  real foreign key — it's an app-layer constraint, so don't "fix" it into five separate tables
  without discussing it first.
- `Nominee` is polymorphic (`owner_type` + `owner_id`) shared by `Account` and
  `SavingsCertificate`. Same app-layer FK constraint as Attachment.
- `EmiAmortizationSchedule` drives the auto-split of each EMI into interest (→ EXPENSE) and
  principal (→ LOAN_REPAYMENT). The user never manually splits.
- `TaxRebateContribution` links to `TaxYear` directly (not through `TaxRebatePlan`), enabling
  cross-year rebate clawback queries when an investment is broken early.

## Open decisions — ask, don't assume

- ID strategy: autoincrement `INTEGER` vs UUID `TEXT`. Not yet decided. Default to autoincrement
  for now (single-device, no sync planned) unless told otherwise.
- Multi-profile/multi-person support is out of scope — schema assumes one user.
- Field-level encryption for account/card numbers is undecided — don't add it speculatively.
  Full DB encryption (sqlcipher + biometric unlock) is confirmed — see Stack section.

## Rough phasing

Phase | Entities | Depends On
1 — Ledger Core | Bank, Branch, Account, AccountTermDetail, LoanDetail, Category, Transaction, Card, Nominee, EmiAmortizationSchedule | —
2 — Richer Tracking | SavingsCertificate, InvestmentCompany, Investment, Lent, Attachment, TaxProfile | Phase 1
3 — SMS Automation | SmsParsingRule, ParsedSmsLog | Phase 1
4 — Statement Recon | Statement, StatementLine, PDF parse engine | Phase 1, 3
5 — Tax Engine | TaxRateSet + 8 child configs, TaxYear, TaxYearIncomeSource, TaxYearComputationDetail, TpsDeduction, RebateClawbackEvent, TaxRebatePlan, TaxRebateContribution, WealthCategory, WealthAsset, SurchargeRule, computation engine | Phase 1, 2
6 — Automation Layer | RecurringTransactionTemplate, Tag, TransactionTag, budget/insight views | Phase 1

## Working agreement

- Keep diffs scoped to what was asked — don't drive-by refactor unrelated files.
- This project runs on BYOK / rate-limited models day to day, so prefer small, focused changes
  over large speculative rewrites; ask before doing a multi-file pass.
- [Build / test / lint / format commands — fill in once the project is scaffolded]
- [Directory structure conventions, e.g. lib/features/*, lib/data/* — fill in]

## Feature capture workflow

Features are **not** specified upfront. The user describes what they want during chat
sessions, and the assistant records the decisions in `FEATURES.md` incrementally.
Do not speculatively write features or schema that the user hasn't explicitly asked for.

When the user asks for a new feature or changes an existing one:

1. Ask clarifying questions if the ask is ambiguous — don't guess.
2. Once clear, update `FEATURES.md` with the agreed scope (user story,
   acceptance criteria, any schema changes).
3. If the feature introduces new entities or modifies existing ones, update the
   schema doc in `docs/database/Schema/` at the same time.
4. Never commit code for a feature that hasn't been recorded in `FEATURES.md` first.
