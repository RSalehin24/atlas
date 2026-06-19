# Personal Finance App — Data Schema (v1)

This extends your `Personal Financial Application.txt` draft and folds in patterns found in
`SAVINGS.xlsx` (your current Excel system). Types are written generically (SQLite/Drift-friendly:
`TEXT`, `INTEGER`, `REAL`, `DATE`, `DATETIME`, `BOOLEAN`).

---

## 0. What changed from your draft, and why

- **Added `Transaction`** — wasn't in the original doc at all, but it's the entity everything else
  hangs off. See §2.8.
- **Added `Lent`** — your `SAVINGS.xlsx` has a whole sheet for money lent to people (`CITY to SONALI`,
  `HAND to HAND`), tracked separately from bank `LOAN`s. Not in your entity draft, but clearly part
  of your actual financial picture. See §2.15.
- **Extracted `Branch` from `Bank` and `Account`** — `branch`, `routing_number`, and
  `branch_address` were duplicated as plain text fields across entities. They now live in a
  dedicated `Branch` table with a FK → `Bank`. `Account` references `Branch` via `branch_id`
  instead of carrying its own inline `branch` and `routing_number`.
- **Split `Account` into base + detail tables** instead of one wide table with ~15 nullable
  FDR/DPS/LOAN-only columns. A `SAVINGS` row never touches those columns under this design.
- **Added `Category`, `Tag`** — your draft has no way to classify a transaction as "Food" vs "Rent"
  vs "Salary," which your `Current Month Breakdown` sheet clearly relies on.
- **Added `SmsParsingRule` / `ParsedSmsLog`** — supporting tables for capturing transactions from
  bank SMS alerts rather than typing every entry by hand.
- **Added `RecurringTransactionTemplate`** — for EMIs, DPS installments, rent: things that repeat
  on a schedule, visible in your LOANS and DPS data.
- **Generalized `Attachment`** — your `PROOF` sheet and "Associated Documents" on `Investment` are
  the same pattern (a file attached to some other record). One table instead of duplicating file
  fields on Investment, Loan, Lent, etc.
- **Completed cut-off/ambiguous fields** — `Savings Certificate.Expected` → `expected_maturity_amount`;
  `Nominee` (was a bare string) → its own table supporting multiple nominees with percentage split,
  which is how Bangladeshi bank accounts actually work.
- **Filled gaps your draft had**: Current accounts had no `Balance` field even though the type enum
  includes `CURRENT`; `LOAN` had no `Interest Rate`; `Card` had no `Credit Limit` or billing-cycle
  fields despite Credit being a listed card type.
- **Full tax solution** — `is_tax_rebate_eligible` moved from the generic `Account` table to
  `AccountTermDetail` (DPS‑only). Added `TaxSpendingHead` (NBR-defined heads: Medical, House
  Rent, Conveyance, Section 44 investments, etc.) linked from `Category` via `tax_head_id`,
  enabling the app to aggregate transaction spending by NBR head. Also added `TaxProfile`
  (employment type, TIN, employer data), `TaxYear` (income aggregation, VAT on interest, TPS,
  net payable), and linked `TaxRebatePlan`/`TaxRebateContribution` under each tax year to track
  rebatable DPS and Sanchayapatra investments.
- **Statement reconciliation** — added `Statement` (uploaded PDF per account/card + month/year,
  stored permanently with dedup via file hash) and `StatementLine` (each parsed row with
  `match_status` and FK → `Transaction`). `Transaction` gained `statement_line_id` and
  `reconciled_at` so that SMS-parsed entries can be overridden by the authoritative bank statement.
- **Tax planner + admin config** — added eight config entities (`TaxRateSet`, `TaxSlab`,
  `TaxFreeThreshold`, `SalaryExemptionRule`, `RebateRule`, `EligibleInvestment`,
  `MinimumTaxRule`, `FilingTimelineRule`) so all tax parameters (slabs, thresholds, exemption
  formulas, rebate percentages, per-instrument investment caps, minimum tax, filing incentives)
  live in data instead of code. An admin panel switches the active `TaxRateSet` each financial
  year without app updates. `TaxYear` was extended with `exempted_income`, `tax_before_rebate`,
  `max_possible_rebate`, and `investment_required_for_max_rebate` to power the what‑if planner.

---

## 1. Conventions used throughout

- **Primary keys**: `id TEXT PK`, treated as opaque strings. Plain autoincrement `INTEGER` works
  fine for a single-device local SQLite app and is simpler in Drift — switch to UUIDs only if you
  plan multi-device sync later. Pick one now; don't mix.
- **Money**: store as `INTEGER` in minor units (poisha, i.e. value × 100), never `REAL`. Floating
  point will eventually produce a balance like `15000.000000004`. Format to Taka only at the UI layer.
- **Every table** gets `created_at DATETIME` and `updated_at DATETIME` even where not listed below.
- **Enums**: implemented as `TEXT` with a `CHECK` constraint in SQLite, or a proper Dart enum column
  in Drift. Listed inline per entity.
- **Computed fields** (interest, profit, maturity amounts) are marked *(computed)* — store them if
  you want fast reads and historical snapshots even after rates change, but compute from source
  fields rather than letting the user hand-edit them.

---

## 2. Entities

### 2.1 Bank (financial institution)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| name | TEXT | |
| institution_type | TEXT | enum: `BANK`, `MFS`, `NBFI` — lets you add bKash/Nagad/Rocket later without a new table |
| swift_code | TEXT | nullable |
| logo_url | TEXT | nullable |

`Bank` has a 1:N relationship with `Branch`.

### 2.2 Branch

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| bank_id | TEXT FK → Bank | |
| name | TEXT | e.g. "Gulshan Branch", "Motijheel Branch" |
| routing_number | TEXT | nullable — Bangladesh-specific branch-level routing number |
| address | TEXT | nullable |
| district | TEXT | nullable |
| contact_number | TEXT | nullable |

MFS providers (bKash, Nagad, Rocket) typically don't have branches — a single "Digital" branch record can serve as a placeholder.

### 2.3 Account (base)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| bank_id | TEXT FK → Bank | |
| branch_id | TEXT FK → Branch | nullable — MFS accounts may not have a branch |
| identifier_name | TEXT | your own label, e.g. "EBL Salary A/C" |
| account_number | TEXT | mask in UI; consider encrypting at rest |
| type | TEXT | enum: `SAVINGS`, `CURRENT`, `DPS`, `FDR`, `LOAN` |
| account_holder_name | TEXT | |
| mobile_number | TEXT | nullable |
| email | TEXT | nullable |
| currency | TEXT | default `BDT` |
| current_balance | INTEGER | meaning depends on type — see below |
| overdraft_limit | INTEGER | nullable, `CURRENT` only |
| status | TEXT | enum: `ACTIVE`, `CLOSED`, `MATURED`, `DEFAULTED` |
| opening_date | DATE | |

`current_balance` meaning by type: `SAVINGS`/`CURRENT` = ledger balance · `DPS`/`FDR` = accumulated
value to date · `LOAN` = outstanding principal remaining. Keeping one column across all types means
"total net worth" is a single `SUM(current_balance)` with a sign flip for `LOAN`, instead of a
type-by-type special case.

### 2.4 AccountTermDetail (1:1 with Account, type = `FDR` or `DPS` only)

| Field | Type | Notes |
|---|---|---|
| account_id | TEXT PK/FK → Account | |
| start_date | DATE | |
| maturity_date | DATE | |
| tenure_months | INTEGER | |
| interest_rate | REAL | |
| vat_rate | REAL | Bangladesh Bank-mandated govt VAT on interest |
| gross_interest | INTEGER | *(computed)* |
| vat_amount | INTEGER | *(computed)* |
| net_interest | INTEGER | *(computed)* |
| maturity_amount | INTEGER | *(computed)* |
| auto_renewal | BOOLEAN | |
| principal_amount | INTEGER | FDR only |
| installment_amount | INTEGER | DPS only |
| installment_date | INTEGER | DPS only, day-of-month 1–31 |
| total_principal_on_maturity | INTEGER | DPS only, *(computed: installment × tenure)* |
| is_tax_rebate_eligible | BOOLEAN | DPS only, default false — marks this DPS for tax rebate planning |

### 2.5 LoanDetail (1:1 with Account, type = `LOAN` only)

| Field | Type | Notes |
|---|---|---|
| account_id | TEXT PK/FK → Account | |
| loan_type | TEXT | enum: `PERSONAL`, `AUTO`, `HOME`, `CREDIT_CARD_EMI`, `INFORMAL`, `OTHER` |
| disbursed_date | DATE | |
| start_date | DATE | |
| end_date | DATE | |
| principal_amount | INTEGER | |
| interest_rate | REAL | nullable — 0 for an informal family loan |
| processing_fee | INTEGER | nullable; your LOANS sheet calls this "Expenses" |
| emi_amount | INTEGER | |
| installment_date | INTEGER | day-of-month |
| duration_months | INTEGER | |
| total_emi_payable | INTEGER | *(computed: emi × duration)* |
| total_paid | INTEGER | running total, ideally derived from `SUM(Transaction.amount)` against this loan rather than hand-tracked |
| repayment_status | TEXT | enum: `ONGOING`, `REPAID`, `DEFAULTED` |

### 2.6 Nominee

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| account_id | TEXT FK → Account | |
| name | TEXT | |
| relationship | TEXT | e.g. Spouse, Son, Father |
| percentage_share | REAL | shares across nominees on one account should sum to 100 |
| contact_number | TEXT | nullable |
| date_of_birth | DATE | nullable |
| guardian_name | TEXT | nullable, if nominee is a minor |

*MVP shortcut: if multiple nominees per account isn't something you need yet, a single
`nominee_name TEXT` column on Account works fine — promote to this table when you need it.*

### 2.7 Card

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| account_id | TEXT FK → Account | nullable, 0..1 — only debit cards link to an account |
| type | TEXT | enum: `DEBIT`, `CREDIT` |
| network | TEXT | enum: `VISA`, `MASTERCARD`, `AMEX`, `TAKAPAY`, `DINERS_CLUB` |
| card_number_masked | TEXT | store only last 4, e.g. "•••• 4521" |
| card_number_encrypted | TEXT | nullable — only if you genuinely need the full PAN; encrypt at app layer |
| card_holder_name | TEXT | |
| expire_month | TEXT(2) | |
| expire_year | TEXT(2) | |
| balance | INTEGER | debit: mirrors account; credit: current outstanding |
| credit_limit | INTEGER | nullable, credit only |
| billing_cycle_day | INTEGER | nullable, credit only |
| payment_due_date | DATE | nullable, credit only |
| minimum_due | INTEGER | nullable, credit only |
| annual_fee | INTEGER | nullable |
| status | TEXT | enum: `ACTIVE`, `BLOCKED`, `EXPIRED`, `CANCELLED` |

**Security note**: your draft includes `CVV` as a stored field. There's no legitimate ongoing reason
to persist it after data entry — it exists for point-of-sale verification, not record-keeping. I'd
drop the column entirely. If you must keep it for some reason, it needs app-layer encryption with a
key outside the SQLite file (e.g. platform secure storage) — never plaintext, since a default SQLite
database is just a file on disk.

### 2.8 Transaction (core entity)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| account_id | TEXT FK → Account | required |
| card_id | TEXT FK → Card | nullable |
| category_id | TEXT FK → Category | nullable |
| type | TEXT | enum: `DEBIT`, `CREDIT` |
| amount | INTEGER | always positive; direction comes from `type` |
| currency | TEXT | |
| balance_after | INTEGER | nullable — running balance post-transaction, useful for reconciling against SMS and catching missed entries |
| transaction_date | DATETIME | |
| value_date | DATE | nullable, when bank processing date differs from transaction date |
| description | TEXT | raw narration, often the SMS text verbatim |
| counterparty_name | TEXT | nullable |
| counterparty_account | TEXT | nullable |
| method | TEXT | enum: `CASH`, `NPSB`, `BEFTN`, `RTGS`, `EFTN`, `MOBILE_BANKING`, `CARD`, `INTERNAL_TRANSFER`, `CHEQUE`, `STANDING_INSTRUCTION`, `OTHER` |
| reference_number | TEXT | nullable, bank's own transaction ID |
| status | TEXT | enum: `PENDING`, `COMPLETED`, `FAILED`, `REVERSED` |
| source | TEXT | enum: `MANUAL`, `SMS_PARSED`, `STATEMENT_IMPORT`, `API` |
| raw_sms_id | TEXT FK → ParsedSmsLog | nullable |
| statement_line_id | TEXT FK → StatementLine | nullable — reverse link to the reconciled statement row; set when statement overrides this transaction |
| reconciled_at | DATETIME | nullable — when this transaction was last reconciled against a bank statement |
| linked_transfer_id | TEXT FK → Transaction (self) | nullable — pairs the two legs of a transfer between your own accounts |
| recurring_template_id | TEXT FK → RecurringTransactionTemplate | nullable |
| attachment_id | TEXT FK → Attachment | nullable |
| notes | TEXT | nullable |

**Transfer pairing**: moving money from Savings to a DPS installment is two rows — a `DEBIT` on
Savings and a `CREDIT` on DPS — each pointing at the other via `linked_transfer_id`. This keeps every
account's ledger self-consistent without a separate "Transfer" entity.

**SMS link**: when `source = SMS_PARSED`, `raw_sms_id` traces the transaction back to the original
message for audit/debugging when a parse looks wrong.

**Statement link**: when `statement_line_id` is set, the transaction's values (amount, date,
counterparty) have been verified against the bank statement. The statement is the authoritative
source — automated reconciliation overwrites SMS-parsed values with statement values.

### 2.9 Category

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| parent_category_id | TEXT FK → Category (self) | nullable, for subcategories |
| name | TEXT | |
| tax_head_id | TEXT FK → TaxSpendingHead | nullable — links this category to an NBR-defined tax head |
| type | TEXT | enum: `INCOME`, `EXPENSE`, `TRANSFER`, `INVESTMENT`, `LOAN`, `LOAN_REPAYMENT` |
| icon | TEXT | nullable |
| color | TEXT | nullable |
| is_system | BOOLEAN | default false — seed categories vs ones you add |

Seeded income categories:
| User Category | Cat. Type | Tax Head |
|---|---|---|
| Monthly Salary | INCOME | Income from Employment (Sch-1) |
| Festival Bonus | INCOME | Festival Bonus (6th Sch A Para 4) |
| House Rent Received | INCOME | Income from Rent (Sch-2) |
| Business / Freelance | INCOME | Income from Business (Sch-4) |
| Bank Interest | INCOME | Income from Financial Assets |
| Dividend | INCOME | Income from Financial Assets |
| Savings Cert. Interest | INCOME | Income from Financial Assets |
| Capital Gain | INCOME | Income from Capital Gain |
| Other Income | INCOME | Income from Other Sources |

Seeded expense categories (all map to IT‑10BB):
| User Category | Cat. Type | Tax Head (IT‑10BB) |
|---|---|---|
| Groceries & Food | EXPENSE | Personal & Family Fooding… (#1) |
| Household Supplies | EXPENSE | Personal & Family Fooding… (#1) |
| House Rent Paid | EXPENSE | Housing Expense (#2) |
| Home Maintenance | EXPENSE | Housing Expense (#2) |
| Fuel / Petrol | EXPENSE | Personal Transport Expense (#3) |
| Car Maintenance | EXPENSE | Personal Transport Expense (#3) |
| Public Transport | EXPENSE | Personal Transport Expense (#3) |
| Electricity Bill | EXPENSE | Utility Expense (#4) |
| Gas Bill | EXPENSE | Utility Expense (#4) |
| Water Bill | EXPENSE | Utility Expense (#4) |
| Mobile / Internet | EXPENSE | Utility Expense (#4) |
| Tuition / School Fees | EXPENSE | Education Expense (#5) |
| Coaching / Course Fees | EXPENSE | Education Expense (#5) |
| Domestic Travel | EXPENSE | Travel & Vacation (#6) |
| Foreign Travel | EXPENSE | Travel & Vacation (#6) |
| Eid / Festival | EXPENSE | Festival & Other Special (#7) |
| Wedding / Event | EXPENSE | Festival & Other Special (#7) |
| Tax Paid | EXPENSE | TDS & Tax/Surcharge Paid (#8) |
| Loan Interest Paid | EXPENSE | Interest on Personal Loan (#9) |

Seeded transfer categories:
| User Category | Cat. Type | Tax Head |
|---|---|---|
| Internal Transfer | TRANSFER | (none) |
| Cash Deposit | TRANSFER | (none) |
| ATM Withdrawal | TRANSFER | (none) |

Seeded investment categories:
| User Category | Cat. Type | Tax Head |
|---|---|---|
| DPS Contribution | INVESTMENT | DPS Contribution (Sec 44) |
| Life Insurance Premium | INVESTMENT | Life Insurance Premium (Sec 44) |
| Sanchayapatra Purchase | INVESTMENT | Sanchayapatra Investment (Sec 44) |
| PF / CPF Contribution | INVESTMENT | Provident Fund employee (Sec 44) |
| Share / MF Purchase | INVESTMENT | Shares / Debentures / MF (Sec 44) |
| CFD Deposit | INVESTMENT | CFD (Sec 44) |
| Cash Donation | INVESTMENT | Donation cash (Sec 44) |
| Zakat Payment | INVESTMENT | Donation Zakat (Sec 44) |
| Universal Pension | INVESTMENT | Universal Pension (Sec 44) |
| FDR Placement | INVESTMENT | (none — not rebate-eligible) |
| Gold / Asset Purchase | INVESTMENT | (none) |

Seeded loan categories:
| User Category | Cat. Type | Tax Head |
|---|---|---|
| Home Loan Received | LOAN | (none) |
| Personal Loan Received | LOAN | (none) |
| Auto Loan Received | LOAN | (none) |
| Overdraft Taken | LOAN | (none) |

Seeded loan repayment categories:
| User Category | Cat. Type | Tax Head |
|---|---|---|
| Home Loan Principal | LOAN_REPAYMENT | (none) |
| Personal Loan Principal | LOAN_REPAYMENT | (none) |
| Auto Loan Principal | LOAN_REPAYMENT | (none) |
| Overdraft Repayment | LOAN_REPAYMENT | (none) |

*EMI handling: the user (or app via amortisation schedule) splits an EMI into
interest (→ EXPENSE / Loan Interest Paid) and principal (→ LOAN_REPAYMENT).*

### 2.10 TaxSpendingHead (NBR-defined)

Seeded by the app and mapped to official National Board of Revenue spending/deduction heads.
The user does not create or edit these — they are linked from `Category`.

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| name | TEXT | NBR head name e.g. "Salary", "House Rent Allowance", "Life Insurance Premium" |
| code | TEXT | section / schedule / rule reference e.g. "Section 44(1)(a)", "Sixth Schedule Part A Para 3" |
| head_type | TEXT | enum: `INCOME`, `DEDUCTION`, `REBATE_ELIGIBLE`, `ALLOWANCE`, `EXEMPTION` |
| description | TEXT | nullable — explains the rule / limit |
| is_system | BOOLEAN | default true — seeded, not user-created |

The app ships seeded with the official NBR heads matching the e-Return form.
The **expenditure** seeds are the exact 9 categories from **IT-10BB (Statement of Expenses Relating to Lifestyle)**.
The **income** seeds match the 10 heads from the return form. Additional seeds cover Sixth Schedule
allowances/exemptions and Section 44 rebate categories.

| Name | Code | Head Type | Notes |
|---|---|---|---|
| **Expenditure — IT‑10BB Serials 1–9 (exact e‑Return labels)** | | | |
| Personal & Family Fooding, Clothing & Other Essentials | IT-10BB:1 | DEDUCTION | Day-to-day household / grocery spend |
| Housing Expense | IT-10BB:2 | DEDUCTION | Rent & residence maintenance |
| Personal Transport Expense | IT-10BB:3 | DEDUCTION | Fuel, maintenance, parking, tax |
| Utility Expense (Electricity, Gas, Water, Telephone, Mobile, Internet) | IT-10BB:4 | DEDUCTION | All utility bills |
| Education Expense | IT-10BB:5 | DEDUCTION | Children's tuition, school fees, coaching |
| Personal Expense for Local & Foreign Travel, Vacation etc. | IT-10BB:6 | DEDUCTION | Domestic and overseas travel |
| Festival & Other Special Expense | IT-10BB:7 | DEDUCTION | Eid/Puja, wedding, celebration |
| Tax Deducted/Collected at Source (incl. TDS on Sanchayapatra) & Tax/Surcharge Paid on Last Year Return | IT-10BB:8 | DEDUCTION | Advance tax, TDS, paid by employer |
| Interest Paid on Personal Loan (Institution & Other Source) | IT-10BB:9 | DEDUCTION | Bank/NBFI loan interest paid |
| **Income heads (e‑Return income page)** | | | |
| Monthly Salary (basic + dearness + all cash allowances) | Sch‑1 | INCOME | Covers basic, dearness, house rent, medical, conveyance paid in cash |
| Festival Bonus | 6th Sch A Para 4 | EXEMPTION | Up to two festival bonuses — fully exempt |
| Income from Rent | Sch‑2 | INCOME | House property rental income |
| Income from Agriculture | Sch‑3 | INCOME | Agricultural income |
| Income from Business / Profession | Sch‑4 | INCOME | Business / professional / freelance |
| Income from Capital Gain | — | INCOME | Capital gains |
| Income from Financial Assets | — | INCOME | Bank interest, dividend, securities profit |
| Income from Other Sources | — | INCOME | Royalty, license fees, honorarium, govt incentive |
| Share of Income from Firm / AoP | — | INCOME | Partnership / association share |
| Income of Minor / Spouse (if non‑taxpayer) | — | INCOME | Clubbed income |
| Taxable Income from Abroad | — | INCOME | Foreign remittance, overseas earnings |
| **Allowances / Exemptions (Sixth Schedule Part A)** | | | |
| House Rent Allowance | 6th Sch A Para 1 | ALLOWANCE | Exempt up to % of basic by city tier |
| Medical Allowance | 6th Sch A Para 2 | ALLOWANCE | Exempt up to actual or 120k/yr |
| Conveyance Allowance | 6th Sch A Para 3 | ALLOWANCE | Exempt up to 30k/yr |
| Festival Bonus | 6th Sch A Para 4 | EXEMPTION | Two festival bonuses exempt |
| Leave Encashment | 6th Sch A Para 5 | EXEMPTION | At retirement |
| Provident Fund (employer share) | 6th Sch A Para 6 | EXEMPTION | Employer contribution to recognised PF |
| Gratuity | 6th Sch A Para 8 | EXEMPTION | Received at retirement |
| Pension / Annuity | 6th Sch A Para 9 | EXEMPTION | Govt / approved pension |
| Education Assistance | 6th Sch A Para 11 | EXEMPTION | Children's education up to limit |
| Lunch / Meal Subsidy | 6th Sch A Para 12 | EXEMPTION | Free lunch or subsidy |
| Telephone / Internet | 6th Sch A Para 14 | EXEMPTION | Official bill paid by employer |
| Laptop / Vehicle for Official Use | 6th Sch A Para 15 | EXEMPTION | Employer-provided assets |
| **Rebate‑eligible investments (Section 44, Schedule 5)** | | | |
| DPS Contribution | Section 44(1)(a) | REBATE_ELIGIBLE | Deposit Pension Scheme |
| Life Insurance Premium | Section 44(1)(b) | REBATE_ELIGIBLE | Own / spouse life cover |
| Sanchayapatra Investment | Section 44(1)(c) | REBATE_ELIGIBLE | Bangladesh Savings Certificates |
| Provident Fund (employee share) | Section 44(1)(d) | REBATE_ELIGIBLE | GPF / RPF / CPF contribution |
| Superannuation Fund | Section 44(1)(e) | REBATE_ELIGIBLE | Approved superannuation fund |
| Shares / Debentures / Mutual Funds | Section 44(1)(f) | REBATE_ELIGIBLE | Listed share / unit fund purchase |
| CFD / Continuous Fund Deposit | Section 44(1)(g) | REBATE_ELIGIBLE | CFD with scheduled bank |
| Donation (cash) | Section 44(3)(a) | REBATE_ELIGIBLE | Approved charitable institution |
| Donation (Zakat) | Section 44(3)(b) | REBATE_ELIGIBLE | Zakat — deducted before rebate |
| Universal Pension Scheme | Section 44 | REBATE_ELIGIBLE | UPS contribution (2023 Act) |
| Group Insurance Premium | Section 44 | REBATE_ELIGIBLE | Approved group insurance |

The `head_type` drives how the `TaxYear` engine treats each aggregated amount:
- `INCOME` → added to `total_taxable_income`.
- `ALLOWANCE` → tax‑free slice; actual spend above the limit becomes taxable.
- `EXEMPTION` → fully excluded from taxable income.
- `REBATE_ELIGIBLE` → summed into `total_rebate_claimed` capped at Section 44 ceiling.
- `DEDUCTION` → expenditure items that appear on IT‑10BB; aggregated for the lifestyle‑expense
  statement and compared against corresponding allowance/exemption limits.

### 2.11 Tag / TransactionTag

| Table | Fields |
|---|---|
| Tag | id PK, name, color |
| TransactionTag | transaction_id FK, tag_id FK *(composite PK)* |

Free-form labels (e.g. "reimbursable", "business") orthogonal to Category.

### 2.12 RecurringTransactionTemplate

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| account_id | TEXT FK → Account | |
| category_id | TEXT FK → Category | nullable |
| amount | INTEGER | |
| type | TEXT | enum: `DEBIT`, `CREDIT` |
| frequency | TEXT | enum: `WEEKLY`, `MONTHLY`, `QUARTERLY`, `YEARLY` |
| day_of_period | INTEGER | |
| description | TEXT | |
| next_due_date | DATE | |
| end_date | DATE | nullable |
| auto_create | BOOLEAN | default false — whether the app should materialize a Transaction automatically or just remind you |
| is_active | BOOLEAN | default true |

Covers EMIs, DPS installments, rent — anything with a schedule.

### 2.13 SmsParsingRule

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| bank_id | TEXT FK → Bank | |
| sender_id_pattern | TEXT | e.g. "SCBBD", "DBBLAlert" |
| regex_pattern | TEXT | |
| field_mapping | TEXT | JSON: which capture group maps to amount/date/balance/merchant |
| account_type | TEXT | nullable, scopes the rule to one account type |
| sample_sms | TEXT | nullable, kept for reference when the bank changes its format |
| is_active | BOOLEAN | default true |

### 2.14 ParsedSmsLog

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| raw_sms_text | TEXT | |
| sender | TEXT | |
| received_at | DATETIME | |
| matched_rule_id | TEXT FK → SmsParsingRule | nullable |
| parse_status | TEXT | enum: `PARSED`, `UNPARSED`, `FAILED`, `IGNORED` |
| created_transaction_id | TEXT FK → Transaction | nullable |
| notes | TEXT | nullable |

Keeps every inbound SMS, parsed or not — essential for catching the bank-changed-their-format case
and reconciling without re-reading your inbox.

### 2.15 SavingsCertificate

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| bank_id | TEXT FK → Bank | issuing institution |
| certificate_number | TEXT | |
| name | TEXT | holder-facing label |
| type | TEXT | e.g. 5-Year BSP, Family Savings, Pensioner, 3-Monthly Profit |
| holder_name | TEXT | |
| face_value | INTEGER | = your draft's "Balance" |
| issue_date | DATE | |
| schedule_maturity_date | DATE | |
| tenure_months | INTEGER | nullable |
| interest_rate | REAL | nullable |
| vat_rate | REAL | nullable |
| expected_maturity_amount | INTEGER | completes your draft's cut-off "Expected" field |
| encashment_date | DATE | nullable, when redeemed |
| is_tax_rebate_eligible | BOOLEAN | default true |
| status | TEXT | enum: `ACTIVE`, `ENCASHED`, `MATURED` |

### 2.16 Lent (new — present in your real data, missing from your entity draft)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| direction | TEXT | enum: `LENT_OUT`, `BORROWED` |
| counterparty_name | TEXT | person or organization |
| account_id | TEXT FK → Account | the account money left from / will return to |
| method | TEXT | reuses Transaction.method enum |
| amount | INTEGER | |
| lent_date | DATE | |
| expected_return_date | DATE | nullable — your sheet literally has "NOT SURE" as a value here, so this has to be optional |
| actual_return_date | DATE | nullable |
| status | TEXT | enum: `OUTSTANDING`, `PARTIALLY_RETURNED`, `RETURNED`, `WRITTEN_OFF` |
| proof_reference | TEXT | nullable |
| notes | TEXT | nullable |

### 2.17 InvestmentCompany

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| name | TEXT | |
| address | TEXT | nullable |
| contact_info | TEXT | nullable |
| website | TEXT | nullable |

### 2.18 Investment

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| investment_company_id | TEXT FK → InvestmentCompany | |
| funding_account_id | TEXT FK → Account | nullable — which account the money came from |
| investment_datetime | DATETIME | |
| method | TEXT | reuses Transaction.method enum |
| order_reference | TEXT | nullable |
| principal_amount | INTEGER | |
| expected_return_date | DATE | nullable |
| actual_return_date | DATE | nullable |
| return_amount | INTEGER | nullable, until realized |
| profit | INTEGER | nullable, *(computed: return_amount − principal_amount)* |
| status | TEXT | enum: `ACTIVE`, `MATURED`, `DEFAULTED` |

### 2.19 Attachment (generic document store)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| owner_type | TEXT | enum: `ACCOUNT`, `INVESTMENT`, `LOAN`, `LENT`, `TRANSACTION`, `SAVINGS_CERTIFICATE` |
| owner_id | TEXT | the target record's id — SQLite has no native polymorphic FK, so this is an app-layer constraint, not a DB one |
| name | TEXT | |
| file_path | TEXT | |
| mime_type | TEXT | pdf, png, jpg, jpeg |
| uploaded_at | DATETIME | |

Replaces the duplicated "Associated Documents" pattern on Investment and the standalone `PROOF`
sheet — one table, reusable everywhere a file needs attaching.

### 2.20 TaxRebatePlan (per financial‑year planning)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| tax_year_id | TEXT FK → TaxYear | links this plan to the year's tax computation |
| financial_year | TEXT | e.g. "2025-2026" |
| max_eligible_amount | INTEGER | government‑defined ceiling for that FY (e.g. 60 000 BDT for general slab) |
| target_investment | INTEGER | user's planned total |
| notes | TEXT | nullable |
| is_active | BOOLEAN | default true |

One `TaxRebatePlan` per financial year. The app can use this to show progress toward the
rebate limit and suggest how much more to invest.

### 2.21 TaxRebateContribution (line items within a plan)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| plan_id | TEXT FK → TaxRebatePlan | |
| dps_account_id | TEXT FK → Account | nullable — must point to a DPS‑type account |
| savings_certificate_id | TEXT FK → SavingsCertificate | nullable |
| monthly_installment | INTEGER | nullable — for DPS contributions |
| start_date | DATE | |
| end_date | DATE | |
| total_investment | INTEGER | *(computed: monthly_installment × months, or certificate face value)* |
| notes | TEXT | nullable |

Each row represents one planned or actual contribution toward the tax rebate — either a DPS
installment schedule or a Sanchayapatra purchase. At least one of `dps_account_id` /
`savings_certificate_id` must be set.

### 2.22 TaxProfile

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| employment_type | TEXT | enum: `PRIVATE_JOB`, `GOVT_JOB`, `BUSINESS`, `FREELANCER`, `OTHER` |
| tin_number | TEXT | nullable — Taxpayer Identification Number |
| tax_circle | TEXT | nullable |
| tax_zone | TEXT | nullable |
| employer_name | TEXT | nullable |
| employer_address | TEXT | nullable |
| employer_tin | TEXT | nullable |
| contact_number | TEXT | nullable |
| email | TEXT | nullable |

A single profile (multi‑profile out of scope). Employment type drives which tax rules apply.

### 2.23 TaxYear (per‑financial‑year computation)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| financial_year | TEXT | e.g. "2025-2026" |
| tax_profile_id | TEXT FK → TaxProfile | |
| gross_salary_income | INTEGER | from employment |
| interest_income | INTEGER | aggregated from DPS, FDR, SavingsCertificates |
| other_income | INTEGER | rental, freelance, etc. |
| total_taxable_income | INTEGER | *(computed: gross_salary + interest + other)* |
| exempted_income | INTEGER | *(computed: total income − taxable income; driven by SalaryExemptionRule)* |
| vat_on_interest | INTEGER | *(computed: sum of govt VAT from interest‑bearing accounts)* |
| tax_paid_by_employer | INTEGER | TPS deducted at source |
| advance_tax_paid | INTEGER | |
| tax_before_rebate | INTEGER | *(computed: tax on total_taxable_income applied against active TaxSlabs)* |
| total_rebate_claimed | INTEGER | pulled from `TaxRebatePlan.target_investment` for this year |
| max_possible_rebate | INTEGER | *(computed: lowest of 3% of A, rebate_pct_B% of B, or max_rebate_cap per active RebateRule)* |
| investment_required_for_max_rebate | INTEGER | *(computed: max_possible_rebate ÷ (rebate_pct_B / 100) — the minimum investment to hit the cap)* |
| net_tax_payable | INTEGER | *(computed: tax_before_rebate − total_rebate_claimed − tax_paid)* |
| status | TEXT | enum: `DRAFT`, `COMPUTED`, `FILED` |
| notes | TEXT | nullable |

The app aggregates `Transaction` amounts by `Category`, pulls VAT from `AccountTermDetail` /
`SavingsCertificate`, and rebate totals from `TaxRebateContribution` to auto‑fill this entity.
The user reviews and files.

### 2.24 TaxRateSet (master config per financial year)

One active config per FY. The app reads from whichever `TaxRateSet` has `is_active = true` for its
computations. All child entities (slabs, thresholds, rules) cascade-delete when a rate set is
retired.

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| financial_year | TEXT | e.g. "2026-2027" |
| label | TEXT | e.g. "Finance Act 2026 — 5‑year roadmap Year 1" |
| is_active | BOOLEAN | only one may be active at a time |
| notes | TEXT | nullable |

### 2.25 TaxSlab (progressive bracket within a rate set)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| rate_set_id | TEXT FK → TaxRateSet | required |
| slab_order | INTEGER | evaluated in ascending order |
| min_income | INTEGER | inclusive lower bound |
| max_income | INTEGER | nullable — null means "rest of income" (highest slab) |
| rate_percent | REAL | tax rate as a decimal percent, e.g. 10.0 for 10% |

Slabs are evaluated top-down. The first slab has `min_income = 0`. The last slab has `max_income IS NULL`.

### 2.26 TaxFreeThreshold (tax‑free limit by taxpayer category)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| rate_set_id | TEXT FK → TaxRateSet | required |
| category_code | TEXT | enum: `GENERAL`, `WOMAN_SENIOR`, `THIRD_GENDER_DISABLED`, `FREEDOM_FIGHTER_JULI`, `PARENT_OF_DISABLED` |
| category_label | TEXT | human‑readable, e.g. "Women & senior citizens (65+)" |
| exempt_amount | INTEGER | amount of income exempt from tax |

The `PARENT_OF_DISABLED` threshold is an *additional* exemption on top of the primary category.
The app picks the correct row based on `TaxProfile.employment_type` / user‑selected category.

### 2.27 SalaryExemptionRule (standard deduction on employment income)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| rate_set_id | TEXT FK → TaxRateSet | required |
| max_exempt_amount | INTEGER | e.g. 500 000 BDT |
| percent_of_salary | REAL | e.g. 33.33 for ⅓ |
| description | TEXT | nullable — e.g. "Lower of Tk 5,00,000 or ⅓ of gross salary (6th Sch Part A Para 27)" |

Exempted income = MIN(max_exempt_amount, round(gross_salary × percent_of_salary / 100)).

### 2.28 RebateRule (Section 78 investment rebate parameters)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| rate_set_id | TEXT FK → TaxRateSet | required |
| label | TEXT | e.g. "Section 78 — Finance Act 2026" |
| rate_percent_of_A | REAL | rebate as % of total taxable income "A", e.g. 3.0 |
| rate_percent_of_B | REAL | rebate as % of eligible investment "B", e.g. 10.0 |
| max_rebate_amount | INTEGER | absolute cap, e.g. 750 000 BDT |

Rebate = lowest of:
- round(A × rate_percent_of_A / 100)
- round(B × rate_percent_of_B / 100)
- max_rebate_amount

where A = total_taxable_income (excl. exempted/reduced‑rate/minimum‑tax income).

### 2.29 EligibleInvestment (instrument approved for tax rebate)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| rate_set_id | TEXT FK → TaxRateSet | required |
| instrument_code | TEXT | enum: `DPS`, `SANCHAYAPATRA`, `LIFE_INSURANCE`, `PF`, `GPS`, `PENSION`, `GOVT_SECURITIES`, `MUTUAL_FUND`, `LISTED_SHARES`, `ICB_UNIT`, `CHARITABLE_DONATION` |
| instrument_name | TEXT | user‑facing label, e.g. "Deposit Pension Scheme (DPS)" |
| max_annual_limit | INTEGER | nullable — per‑instrument cap (e.g. 120 000 for DPS, 500 000 for govt securities) |
| is_active | BOOLEAN | |
| description | TEXT | nullable |

### 2.30 MinimumTaxRule (minimum tax by location)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| rate_set_id | TEXT FK → TaxRateSet | required |
| location_code | TEXT | enum: `MAJOR_CITY`, `OTHER_CITY`, `OTHER_AREA`, `NEW_TAXPAYER` |
| location_label | TEXT | e.g. "Dhaka & Chattogram city corporation" |
| min_tax_amount | INTEGER | e.g. 5 000 BDT |

If `total_taxable_income > tax_free_threshold`, minimum tax applies per location. `NEW_TAXPAYER`
is a flat lower rate independent of location (e.g. 1 000 BDT for first‑time filers in 2026‑27).

### 2.31 FilingTimelineRule (incentive/penalty per quarter)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| rate_set_id | TEXT FK → TaxRateSet | required |
| quarter_number | INTEGER | 1–4 |
| quarter_label | TEXT | e.g. "Jul–Sep (Q1)" |
| start_month | INTEGER | 7 for Q1, 10 for Q2, 1 for Q3, 4 for Q4 |
| end_month | INTEGER | 9, 12, 3, 6 respectively |
| rebate_percent | REAL | nullable — filing incentive as % of tax (e.g. 5.0) |
| rebate_max_amount | INTEGER | nullable — cap on the filing rebate (e.g. 25 000) |
| penalty_percent | REAL | nullable — late‑filing penalty as % of tax |
| penalty_max_amount | INTEGER | nullable — cap on the penalty |

Q1 (Jul–Sep): rebate_percent 5.0, rebate_max 25 000, no penalty.
Q2 (Oct–Dec): all null — neutral.
Q3 (Jan–Mar): penalty_percent 2.0, penalty_max 3 000.
Q4 (Apr–Jun): penalty_percent 5.0, penalty_max 5 000.

---

### 2.32 Statement (uploaded bank/credit card statement)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| account_id | TEXT FK → Account | nullable — set for account statements (Savings/Current/DPS/FDR/LOAN) |
| card_id | TEXT FK → Card | nullable — set for credit card statements |
| month | INTEGER | 1–12 |
| year | INTEGER | |
| file_path | TEXT | local path to the stored PDF |
| file_name | TEXT | original uploaded filename |
| file_size | INTEGER | bytes |
| file_hash | TEXT | SHA-256 — prevents duplicate uploads, detects tampering |
| uploaded_at | DATETIME | |
| status | TEXT | enum: `UPLOADED`, `PARSING`, `PARSED`, `RECONCILING`, `RECONCILED`, `HAS_ISSUES` |
| notes | TEXT | nullable |

Exactly one of `account_id` or `card_id` must be set (CHECK constraint). A statement belongs to
either an account or a credit card for a given month+year. Re-uploading replaces the previous
statement; the old record is soft-deprecated (its lines keep their data but are marked superseded).

### 2.33 StatementLine (parsed row within a statement)

| Field | Type | Notes |
|---|---|---|
| id | TEXT PK | |
| statement_id | TEXT FK → Statement | required |
| line_number | INTEGER | ordinal position within the statement PDF |
| transaction_date | DATE | |
| value_date | DATE | nullable |
| description | TEXT | raw line from the statement |
| amount | INTEGER | positive value; direction inferred from debit/credit column in statement |
| balance_after | INTEGER | nullable — running balance per the statement's own column |
| counterparty_name | TEXT | nullable |
| reference_number | TEXT | nullable — bank's own transaction ID printed on the statement |
| match_status | TEXT | enum: `UNMATCHED`, `MATCHED`, `NEW_TRANSACTION`, `SUSPICIOUS` |
| transaction_id | TEXT FK → Transaction | nullable — set when a match is found or a new Transaction is created |
| confidence | REAL | nullable — 0.0–1.0, auto‑match confidence (amount + date + time proximity) |
| matched_at | DATETIME | nullable |
| matched_by | TEXT | enum: `AUTO`, `MANUAL` — how the match was resolved |
| superseded_by | TEXT FK → StatementLine (self) | nullable — when a re‑uploaded statement replaces this line |

`match_status` semantics:
- `UNMATCHED` — statement line exists, no matching Transaction found → will create a new Transaction
- `MATCHED` — linked to an existing Transaction, values reconciled
- `NEW_TRANSACTION` — Transaction was created from this line (no SMS match)
- `SUSPICIOUS` — line matched but with low confidence or conflicting values → needs user review

---

## 3. Relationship summary

```
Bank          1───* Branch
Bank          1───* SavingsCertificate
Bank          1───* Account
Branch        1───* Account
Account       1───0..1* Card        (a card may belong to no account: standalone credit cards)
Account       1───* Transaction
Account       1───* Nominee
Account       1───* Investment      (as funding_account)
Account       1───* Lent
Account       1───1 AccountTermDetail   (only when type ∈ {FDR, DPS})
Account       1───1 LoanDetail          (only when type = LOAN)
Card          1───* Transaction
Account       1───0..1* Statement   (monthly statements per account)
Card          1───0..1* Statement   (credit card statements)
Statement     1───* StatementLine
StatementLine 1───0..1 Transaction  (reconciliation match)
TaxSpendingHead 1───* Category
Category      1───* Transaction
Category      1───* Category        (self, parent/child)
Transaction   1───1 Transaction     (self, transfer pairing)
Transaction   *───* Tag             (via TransactionTag)
InvestmentCompany  1───* Investment
SmsParsingRule     1───* ParsedSmsLog
ParsedSmsLog       1───0..1 Transaction
TaxProfile         1───* TaxYear
TaxYear            1───* TaxRebatePlan
TaxRebatePlan      1───* TaxRebateContribution
TaxRebateContribution *───0..1 Account  (DPS only)
TaxRebateContribution *───0..1 SavingsCertificate
Attachment    *───1 (any of: Account, Investment, Loan, Lent, Transaction, SavingsCertificate)
TaxRateSet    1───* TaxSlab
TaxRateSet    1───* TaxFreeThreshold
TaxRateSet    1───* SalaryExemptionRule
TaxRateSet    1───* RebateRule
TaxRateSet    1───* EligibleInvestment
TaxRateSet    1───* MinimumTaxRule
TaxRateSet    1───* FilingTimelineRule
```

---

## 4. Things that should be queries, not tables

Several of your `SAVINGS.xlsx` sheets are summaries that should be computed from `Transaction` +
`Account` once this schema is live, rather than maintained by hand:

- **`2026` monthly sheet** → `GROUP BY` month, category, on `Transaction`
- **`Current Month Breakdown`** → same query, filtered to current month
- **`Current Balance 2025`** → `SUM(current_balance)` across `Account`, grouped by type, with `LOAN`
  and `Lent.OUTSTANDING` subtracted

Example — monthly income/expense by category:

```sql
SELECT
  strftime('%Y-%m', transaction_date) AS month,
  c.name AS category,
  c.type AS category_type,
  SUM(t.amount) AS total
FROM transaction t
JOIN category c ON c.id = t.category_id
WHERE t.status = 'COMPLETED'
GROUP BY month, c.id
ORDER BY month, total DESC;
```

Example — net worth snapshot:

```sql
SELECT
  SUM(CASE WHEN type = 'LOAN' THEN -current_balance ELSE current_balance END) AS net_worth
FROM account
WHERE status = 'ACTIVE';
```

This is the actual payoff of moving off Excel: the monthly rollup stops being a manual task.

---

## 5. Open decisions worth settling before you write the Drift schema

- **Single profile vs. multiple people.** Right now `Account.account_holder_name` is a string. If
  you ever want to track a family member's accounts in the same app with separate views, you'd want
  a `Person`/`Owner` entity above `Account` instead. Worth deciding now — it's a bigger migration later.
- **ID strategy.** Autoincrement `INTEGER` is simpler and is fine as long as this stays a single-device
  local database. Switch to UUID `TEXT` only if cloud sync or multi-device is on the roadmap.
- **Field-level encryption.** Account numbers and any retained card data — encrypted at the app layer
  (e.g. via `sqlcipher` or platform secure storage), or is OS-level disk encryption your accepted
  threat model? Worth deciding once rather than per-field later.
- **Soft delete.** Hard `DELETE` vs. an `is_deleted` flag on Account/Transaction — matters if you
  ever want an audit trail of corrections rather than silent loss.

---

## 6. Suggested build order

1. **Ledger core**: `Bank`, `Account`, `Category`, `Transaction`, `Card` — gets you off Excel for
   day-to-day tracking with manual entry.
2. **Richer tracking**: `AccountTermDetail`, `LoanDetail`, `SavingsCertificate`, `InvestmentCompany`,
    `Investment`, `Lent`, `Nominee`, `Attachment`, `TaxProfile`, `TaxYear`,
    `TaxRebatePlan`, `TaxRebateContribution`.
3. **SMS automation**: `SmsParsingRule`, `ParsedSmsLog`, wiring `Transaction.source` / `raw_sms_id`.
4. **Statement reconciliation**: `Statement`, `StatementLine`, PDF parsing + fuzzy matching engine,
   reconciliation UI. Depends on core ledger (Phase 1) and SMS automation (Phase 3) being in place
   so that statement lines can be matched against SMS‑created transactions.
5. **Tax admin + planner**: `TaxRateSet`, `TaxSlab`, `TaxFreeThreshold`, `SalaryExemptionRule`,
   `RebateRule`, `EligibleInvestment`, `MinimumTaxRule`, `FilingTimelineRule` — admin panel to
   configure each FY's parameters without code changes. TaxYear extended with planning fields.
   Tax computation engine that reads from the active config.
6. **Automation layer**: `RecurringTransactionTemplate`, `Tag`, plus whatever budget/insight views
   sit on top once the ledger has enough history to be useful.
