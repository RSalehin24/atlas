# AGENTS.md

## Project

**Personal Financial Atlas** — a Bangladesh-focused personal finance app replacing a manual Excel
system with a Flutter mobile app that tracks bank accounts, cards, transactions (mostly captured
via SMS parsing), DPS/FDR/savings certificates, investments, informal lending, and auto-computes
tax using data-driven NBR e-Return categories.

## Stack

- Flutter (mobile, Android-first) — *or React Native, undecided*
- Drift (SQLite ORM) for local persistence — with sqlcipher for full DB encryption
- Biometric unlock (fingerprint/face) + passphrase fallback on app launch
- Admin panel required — tax parameters (slabs, rates, rebate %, instrument caps, minimum tax,
  filing incentives) are data-driven, not hardcoded. Admin activates one `TaxRateSet` per FY
  without app updates.
- [Other deps / state management / package name — fill in]

## Platform requirements

- The app must be publishable to Google Play Store — all dependencies, permissions, and
  configurations must comply with Google Play policies (privacy policy, data handling,
  permissions rationale, target API level, etc.). Features or library choices that could
  jeopardize Play Store acceptance must be flagged.

## Status

Schema is being refined during inception phase; implementation has not started.
The authoritative field-level schema lives in `docs/database/Schema/Personal_Finance_App_Schema.md`. Read that file before touching anything
data-model related; don't re-derive entities from scratch in this file or from guesswork.

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
- Tax computation is entirely data-driven. Eight config entities under `TaxRateSet` (slabs,
  thresholds, exemption rule, rebate rule, investment caps, minimum tax, filing timeline)
  live in the database. Admin activates one `TaxRateSet` per FY. The app computes tax,
  exemptions, rebates, and investment suggestions by reading the active config — no code
  changes needed when NBR updates rates.
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

## Open decisions — ask, don't assume

- ID strategy: autoincrement `INTEGER` vs UUID `TEXT`. Not yet decided. Default to autoincrement
  for now (single-device, no sync planned) unless told otherwise.
- Multi-profile/multi-person support is out of scope — schema assumes one user.
- Field-level encryption for account/card numbers is undecided — don't add it speculatively.
  Full DB encryption (sqlcipher + biometric unlock) is confirmed — see Stack section.

## Rough phasing

1. Ledger core — `Bank`, `Account`, `Category`, `Transaction`, `Card`, manual entry only
2. Richer tracking — `SavingsCertificate`, `Investment`, `Lent`, `Nominee`, `Attachment`
3. SMS automation — `SmsParsingRule`, `ParsedSmsLog`, auto-creating transactions from SMS
4. Statement reconciliation — `Statement`, `StatementLine`, PDF parsing + matching UI
5. Tax admin + planner — `TaxRateSet` + 7 child config entities, admin panel, planning UI
6. Automation layer — `RecurringTransactionTemplate`, budgeting/insight views

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
