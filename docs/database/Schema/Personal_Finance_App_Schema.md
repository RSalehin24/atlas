# Personal Financial Atlas — Complete Data Schema & Design (v2)

> **One authoritative document.** Supersedes all previous schema drafts.
> Read this before any implementation. Do not guess or re-derive entities.

---

## 0. Gaps Closed in This Revision

| # | Gap | Solution |
|---|---|---|
| 1 | No tax computation engine design | §4 — full engine spec with equations, steps, edge cases |
| 2 | Section 78 "A" not computed correctly | §4.2 — income exclusions are tracked per schedule; `A` = sum of non-excluded, non-reduced, non-final-tax income |
| 3 | Minimum tax floor missing | §4.4 — `MinimumTaxRule` check after slab computation; `minimum_tax_applied` boolean in computation detail |
| 4 | Standard deduction wrongly applied to all income | §4.3 — deduction only on employment income (Sch-1); other schedules keep full amount |
| 5 | Income too coarse (single `other_income` bucket) | `TaxYearIncomeSource` child table with per-schedule breakdown (Sch-1 through Sch-5, Financial Assets, Foreign, etc.) |
| 6 | TPS (Tax at Source) not tracked monthly | `TpsDeduction` entity — monthly deductions summed into `advance_tax_paid` |
| 7 | Cross-year rebate clawback not modelled | `RebateClawbackEvent` — tracks investment broken early, sums rebate claimed across all past FYs |
| 8 | No e-Return export designed | §5 — export schema for NBR's portal + printable IT-10BB |
| 9 | No wealth statement for NBR asset declaration | `WealthAsset` + `WealthCategory` — auto-computed from accounts + manual entries |
| 10 | Surcharge on high net worth missing | `SurchargeRule` entity + computation in TaxYear |
| 11 | EMI amortization not modelled | `EmiAmortizationSchedule` — auto-splits each EMI into interest + principal |
| 12 | Nominee only on Account, not SavingsCertificate | `Nominee` made polymorphic (`owner_type` + `owner_id`) |
| 13 | Multi-year investment not trackable | `TaxRebateContribution` now links to `TaxYear` directly (one DPS → rows in 5 different FYs) |
| 14 | No PDF parsing architecture | §6 — parser engine design for 15+ bank formats, graceful degradation |

---

## 1. Conventions

- **Money**: `INTEGER` in poisha (taka × 100). Never `REAL`.
- **IDs**: `INTEGER` autoincrement (single-device, no sync). If multi-device sync is ever needed, migration to UUID TEXT is a separate effort.
- **Timestamps**: every table gets `created_at` and `updated_at` DATETIME even where not listed.
- **Enums**: `TEXT` with `CHECK` constraint in SQLite, Dart enum in Drift.
- **Soft delete**: no hard `DELETE`. Every table gets `is_deleted BOOLEAN DEFAULT false` unless noted otherwise.
- **Computed fields**: marked *(computed)* — compute from source data, never hand-edit.

---

## 2. Entity Reference

### 2.1 Bank

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| name | TEXT | |
| institution_type | TEXT | enum: `BANK`, `MFS`, `NBFI` |
| swift_code | TEXT | nullable |
| logo_url | TEXT | nullable |
| is_deleted | BOOLEAN | |

### 2.2 Branch

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| bank_id | INTEGER FK → Bank | |
| name | TEXT | |
| routing_number | TEXT | nullable |
| address | TEXT | nullable |
| district | TEXT | nullable |
| contact_number | TEXT | nullable |
| is_deleted | BOOLEAN | |

MFS providers (bKash, Nagad) get a single "Digital" branch placeholder.

### 2.3 Account

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| bank_id | INTEGER FK → Bank | |
| branch_id | INTEGER FK → Branch | nullable (MFS) |
| identifier_name | TEXT | user label e.g. "EBL Salary A/C" |
| account_number | TEXT | encrypt at rest via sqlcipher |
| type | TEXT | enum: `SAVINGS`, `CURRENT`, `DPS`, `FDR`, `LOAN` |
| account_holder_name | TEXT | |
| mobile_number | TEXT | nullable |
| email | TEXT | nullable |
| currency | TEXT | default `BDT` |
| current_balance | INTEGER | savings/current = ledger; DPS/FDR = accumulated; LOAN = outstanding principal |
| overdraft_limit | INTEGER | nullable, CURRENT only |
| status | TEXT | enum: `ACTIVE`, `CLOSED`, `MATURED`, `DEFAULTED` |
| opening_date | DATE | |
| is_deleted | BOOLEAN | |

### 2.4 AccountTermDetail (1:1 with Account, type = FDR or DPS)

| Field | Type | Notes |
|---|---|---|
| account_id | INTEGER PK/FK → Account | |
| start_date | DATE | |
| maturity_date | DATE | |
| tenure_months | INTEGER | |
| interest_rate | REAL | |
| vat_rate | REAL | Bangladesh Bank govt VAT on interest |
| gross_interest | INTEGER | *(computed)* |
| vat_amount | INTEGER | *(computed)* |
| net_interest | INTEGER | *(computed)* |
| maturity_amount | INTEGER | *(computed)* |
| auto_renewal | BOOLEAN | |
| principal_amount | INTEGER | FDR only |
| installment_amount | INTEGER | DPS only |
| installment_date | INTEGER | DPS only, 1–31 |
| total_principal_on_maturity | INTEGER | DPS only, *(computed)* |
| is_tax_rebate_eligible | BOOLEAN | DPS only, default false |

### 2.5 LoanDetail (1:1 with Account, type = LOAN)

| Field | Type | Notes |
|---|---|---|
| account_id | INTEGER PK/FK → Account | |
| loan_type | TEXT | enum: `PERSONAL`, `AUTO`, `HOME`, `CREDIT_CARD_EMI`, `INFORMAL`, `OTHER` |
| disbursed_date | DATE | |
| start_date | DATE | |
| end_date | DATE | |
| principal_amount | INTEGER | |
| interest_rate | REAL | nullable (informal loan may be 0) |
| processing_fee | INTEGER | nullable |
| emi_amount | INTEGER | |
| installment_date | INTEGER | day-of-month |
| duration_months | INTEGER | |
| total_emi_payable | INTEGER | *(computed: emi × duration)* |
| total_paid | INTEGER | *(computed: SUM of matched Transaction amounts)* |
| repayment_status | TEXT | enum: `ONGOING`, `REPAID`, `DEFAULTED` |
| is_interest_tax_deductible | BOOLEAN | default false — home loan interest under Sixth Schedule |
| tax_deductible_interest_this_year | INTEGER | nullable — interest paid this FY qualifying for deduction |

### 2.6 EmiAmortizationSchedule

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| loan_detail_id | INTEGER FK → LoanDetail | |
| installment_number | INTEGER | ordinal 1..N |
| due_date | DATE | |
| emi_amount | INTEGER | |
| interest_component | INTEGER | *(computed from amortization formula)* |
| principal_component | INTEGER | *(computed)* |
| outstanding_balance | INTEGER | remaining principal after this payment |
| is_paid | BOOLEAN | |
| paid_date | DATE | nullable |
| transaction_id | INTEGER FK → Transaction | nullable, set when payment is recorded |

The app fetches this schedule when recording an EMI transaction and automatically splits into:
- DEBIT Transaction (interest) → Category = Loan Interest Paid (EXPENSE)
- DEBIT Transaction (principal) → Category = {LoanType} Principal (LOAN_REPAYMENT)
- Linked via `linked_transfer_id`

### 2.7 Nominee (polymorphic)

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| owner_type | TEXT | enum: `ACCOUNT`, `SAVINGS_CERTIFICATE` |
| owner_id | INTEGER | app-layer FK (SQLite cannot enforce polymorphic FK) |
| name | TEXT | |
| relationship | TEXT | e.g. Spouse, Son, Father |
| percentage_share | REAL | shares across nominees on one owner should sum to 100 |
| contact_number | TEXT | nullable |
| date_of_birth | DATE | nullable |
| guardian_name | TEXT | nullable, if nominee is a minor |
| is_deleted | BOOLEAN | |

### 2.8 Card

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| account_id | INTEGER FK → Account | nullable (0..1 — debit cards link to account; credit may not) |
| type | TEXT | enum: `DEBIT`, `CREDIT` |
| network | TEXT | enum: `VISA`, `MASTERCARD`, `AMEX`, `TAKAPAY`, `DINERS_CLUB` |
| card_number_masked | TEXT | last 4 digits only, e.g. "•••• 4521" |
| card_number_encrypted | TEXT | nullable — full PAN encrypted at app layer if needed |
| card_holder_name | TEXT | |
| expire_month | TEXT(2) | |
| expire_year | TEXT(2) | |
| balance | INTEGER | |
| credit_limit | INTEGER | nullable, credit only |
| billing_cycle_day | INTEGER | nullable, credit only |
| payment_due_date | DATE | nullable, credit only |
| minimum_due | INTEGER | nullable, credit only |
| annual_fee | INTEGER | nullable |
| status | TEXT | enum: `ACTIVE`, `BLOCKED`, `EXPIRED`, `CANCELLED` |
| is_deleted | BOOLEAN | |

**CVV is never persisted.** Full stop.

### 2.9 Transaction

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| account_id | INTEGER FK → Account | |
| card_id | INTEGER FK → Card | nullable |
| category_id | INTEGER FK → Category | nullable |
| type | TEXT | enum: `DEBIT`, `CREDIT` |
| amount | INTEGER | always positive; direction from `type` |
| currency | TEXT | |
| balance_after | INTEGER | nullable — running balance |
| transaction_date | DATETIME | |
| value_date | DATE | nullable |
| description | TEXT | raw narration |
| counterparty_name | TEXT | nullable |
| counterparty_account | TEXT | nullable |
| method | TEXT | enum: `CASH`, `NPSB`, `BEFTN`, `RTGS`, `EFTN`, `MOBILE_BANKING`, `CARD`, `INTERNAL_TRANSFER`, `CHEQUE`, `STANDING_INSTRUCTION`, `OTHER` |
| reference_number | TEXT | nullable |
| status | TEXT | enum: `PENDING`, `COMPLETED`, `FAILED`, `REVERSED` |
| source | TEXT | enum: `MANUAL`, `SMS_PARSED`, `STATEMENT_IMPORT`, `API` |
| raw_sms_id | INTEGER FK → ParsedSmsLog | nullable |
| statement_line_id | INTEGER FK → StatementLine | nullable |
| reconciled_at | DATETIME | nullable |
| linked_transfer_id | INTEGER FK → Transaction (self) | nullable — pairs legs of internal transfer |
| recurring_template_id | INTEGER FK → RecurringTransactionTemplate | nullable |
| attachment_id | INTEGER FK → Attachment | nullable |
| notes | TEXT | nullable |
| is_deleted | BOOLEAN | |

### 2.10 Category

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| rate_set_id | INTEGER FK → TaxRateSet | nullable — null = user-created, persists across years |
| parent_category_id | INTEGER FK → Category (self) | nullable |
| name | TEXT | |
| tax_head_id | INTEGER FK → TaxSpendingHead | nullable |
| type | TEXT | enum: `INCOME`, `EXPENSE`, `TRANSFER`, `INVESTMENT`, `LOAN`, `LOAN_REPAYMENT` |
| icon | TEXT | nullable |
| color | TEXT | nullable |
| is_system | BOOLEAN | default false |
| is_deleted | BOOLEAN | |

Seeded lists are unchanged from v1 (see previous schema for full tables). Key principle: all NBR-related seeded categories are scoped to `rate_set_id`. User categories live across FYs.

### 2.11 TaxSpendingHead

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| rate_set_id | INTEGER FK → TaxRateSet | required — which FY this head belongs to |
| name | TEXT | NBR head name |
| code | TEXT | section/schedule reference e.g. "IT-10BB:1", "Sch-1", "Section 44(1)(a)" |
| head_type | TEXT | enum: `INCOME`, `DEDUCTION`, `REBATE_ELIGIBLE`, `ALLOWANCE`, `EXEMPTION` |
| description | TEXT | nullable |
| is_deleted | BOOLEAN | |

Seeded list unchanged from v1 (9 IT-10BB expenditure heads + 10 income heads + allowances + rebate instruments). Every new TaxRateSet duplicates the heads, admin adjusts changed ones.

### 2.12 Tag / TransactionTag

| Table | Fields |
|---|---|
| Tag | id PK, name, color, is_deleted |
| TransactionTag | transaction_id FK, tag_id FK (composite PK) |

### 2.13 RecurringTransactionTemplate

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| account_id | INTEGER FK → Account | |
| category_id | INTEGER FK → Category | nullable |
| amount | INTEGER | |
| type | TEXT | enum: `DEBIT`, `CREDIT` |
| frequency | TEXT | enum: `WEEKLY`, `MONTHLY`, `QUARTERLY`, `YEARLY` |
| day_of_period | INTEGER | |
| description | TEXT | |
| next_due_date | DATE | |
| end_date | DATE | nullable |
| auto_create | BOOLEAN | |
| is_active | BOOLEAN | |
| is_deleted | BOOLEAN | |

### 2.14 SmsParsingRule

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| bank_id | INTEGER FK → Bank | |
| sender_id_pattern | TEXT | e.g. "SCBBD", "DBBLAlert" |
| regex_pattern | TEXT | |
| field_mapping | TEXT | JSON: capture group → field |
| account_type | TEXT | nullable, scope to one account type |
| sample_sms | TEXT | nullable |
| is_active | BOOLEAN | |
| is_deleted | BOOLEAN | |

### 2.15 ParsedSmsLog

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| raw_sms_text | TEXT | |
| sender | TEXT | |
| received_at | DATETIME | |
| matched_rule_id | INTEGER FK → SmsParsingRule | nullable |
| parse_status | TEXT | enum: `PARSED`, `UNPARSED`, `FAILED`, `IGNORED` |
| created_transaction_id | INTEGER FK → Transaction | nullable |
| notes | TEXT | nullable |
| is_deleted | BOOLEAN | |

### 2.16 SavingsCertificate

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| bank_id | INTEGER FK → Bank | issuing institution |
| certificate_number | TEXT | |
| category_code | TEXT | enum: `5Y_BSP`, `3Y_BSP`, `FAMILY_SAVINGS`, `PENSIONER`, `MONTHLY_PROFIT`, `OTHER` |
| name | TEXT | holder-facing label |
| holder_name | TEXT | primary holder |
| joint_holder_name | TEXT | nullable |
| joint_holder_relation | TEXT | nullable |
| holder_gender | TEXT | nullable |
| date_of_birth | DATE | nullable — Pensioner requires min age |
| bank_account_for_profit | TEXT | nullable |
| face_value | INTEGER | purchase principal |
| issue_date | DATE | |
| schedule_maturity_date | DATE | |
| tenure_months | INTEGER | nullable |
| interest_rate | REAL | nullable — profit rate at purchase |
| vat_rate | REAL | nullable |
| expected_maturity_amount | INTEGER | *(computed)* |
| profit_payment_frequency | TEXT | enum: `MONTHLY`, `QUARTERLY`, `HALF_YEARLY`, `ANNUALLY`, `AT_MATURITY` |
| encashment_date | DATE | nullable |
| is_tax_rebate_eligible | BOOLEAN | default true |
| status | TEXT | enum: `ACTIVE`, `ENCASHED`, `MATURED` |
| is_deleted | BOOLEAN | |

Nominees are stored in the `Nominee` table (§2.7) with `owner_type = 'SAVINGS_CERTIFICATE'`.

### 2.17 Lent

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| direction | TEXT | enum: `LENT_OUT`, `BORROWED` |
| counterparty_name | TEXT | |
| account_id | INTEGER FK → Account | |
| method | TEXT | reuses Transaction.method enum |
| amount | INTEGER | |
| lent_date | DATE | |
| expected_return_date | DATE | nullable |
| actual_return_date | DATE | nullable |
| status | TEXT | enum: `OUTSTANDING`, `PARTIALLY_RETURNED`, `RETURNED`, `WRITTEN_OFF` |
| proof_reference | TEXT | nullable |
| notes | TEXT | nullable |
| is_deleted | BOOLEAN | |

### 2.18 InvestmentCompany

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| name | TEXT | |
| address | TEXT | nullable |
| contact_info | TEXT | nullable |
| website | TEXT | nullable |
| is_deleted | BOOLEAN | |

### 2.19 Investment

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| investment_company_id | INTEGER FK → InvestmentCompany | |
| funding_account_id | INTEGER FK → Account | nullable |
| investment_datetime | DATETIME | |
| method | TEXT | reuses Transaction.method enum |
| order_reference | TEXT | nullable |
| principal_amount | INTEGER | |
| expected_return_date | DATE | nullable |
| actual_return_date | DATE | nullable |
| return_amount | INTEGER | nullable |
| profit | INTEGER | nullable, *(computed)* |
| status | TEXT | enum: `ACTIVE`, `MATURED`, `DEFAULTED` |
| is_deleted | BOOLEAN | |

### 2.20 Attachment (polymorphic)

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| owner_type | TEXT | enum: `ACCOUNT`, `INVESTMENT`, `LOAN`, `LENT`, `TRANSACTION`, `SAVINGS_CERTIFICATE` |
| owner_id | INTEGER | app-layer FK |
| name | TEXT | |
| file_path | TEXT | |
| mime_type | TEXT | pdf, png, jpg, jpeg |
| uploaded_at | DATETIME | |
| is_deleted | BOOLEAN | |

### 2.21 Statement

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| account_id | INTEGER FK → Account | nullable |
| card_id | INTEGER FK → Card | nullable |
| month | INTEGER | 1–12 |
| year | INTEGER | |
| file_path | TEXT | |
| file_name | TEXT | |
| file_size | INTEGER | bytes |
| file_hash | TEXT | SHA-256 |
| uploaded_at | DATETIME | |
| status | TEXT | enum: `UPLOADED`, `PARSING`, `PARSED`, `RECONCILING`, `RECONCILED`, `HAS_ISSUES` |
| notes | TEXT | nullable |
| is_deleted | BOOLEAN | |

Exactly one of `account_id` or `card_id` must be set.

### 2.22 StatementLine

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| statement_id | INTEGER FK → Statement | |
| line_number | INTEGER | ordinal |
| transaction_date | DATE | |
| value_date | DATE | nullable |
| description | TEXT | |
| amount | INTEGER | positive; direction from debit/credit column |
| balance_after | INTEGER | nullable |
| counterparty_name | TEXT | nullable |
| reference_number | TEXT | nullable |
| match_status | TEXT | enum: `UNMATCHED`, `MATCHED`, `NEW_TRANSACTION`, `SUSPICIOUS` |
| transaction_id | INTEGER FK → Transaction | nullable |
| confidence | REAL | nullable, 0.0–1.0 |
| matched_at | DATETIME | nullable |
| matched_by | TEXT | enum: `AUTO`, `MANUAL` |
| superseded_by | INTEGER FK → StatementLine (self) | nullable |
| is_deleted | BOOLEAN | |

### 2.23 TaxProfile

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| employment_type | TEXT | enum: `PRIVATE_JOB`, `GOVT_JOB`, `BUSINESS`, `FREELANCER`, `OTHER` |
| tin_number | TEXT | nullable |
| tax_circle | TEXT | nullable |
| tax_zone | TEXT | nullable |
| employer_name | TEXT | nullable |
| employer_address | TEXT | nullable |
| employer_tin | TEXT | nullable |
| contact_number | TEXT | nullable |
| email | TEXT | nullable |
| taxpayer_category | TEXT | enum: `GENERAL`, `WOMAN`, `SENIOR_CITIZEN`, `THIRD_GENDER_DISABLED`, `FREEDOM_FIGHTER`, `PARENT_OF_DISABLED` — maps to TaxFreeThreshold |
| is_deleted | BOOLEAN | |

### 2.24 TaxYearIncomeSource (NEW — per-schedule income breakdown)

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| tax_year_id | INTEGER FK → TaxYear | |
| tax_head_id | INTEGER FK → TaxSpendingHead | reference to the specific income head |
| schedule_code | TEXT | e.g. "Sch-1", "Sch-2", "Sch-3", "Sch-4", "Sch-5", "FINANCIAL_ASSETS", "OTHER_SOURCES", "FIRM_AOP", "MINOR_SPOUSE", "FOREIGN" |
| gross_amount | INTEGER | total raw income under this schedule |
| exempted_amount | INTEGER | Sixth Schedule exemptions applied |
| reduced_rate_amount | INTEGER | portion taxed at reduced rate (if any) |
| minimum_tax_amount | INTEGER | portion subject to minimum tax only |
| final_tax_amount | INTEGER | income already taxed at source (final) |
| taxable_amount | INTEGER | *(computed: gross − exempted − reduced − minimum − final)* |
| tax_computed | INTEGER | *(computed)* tax on this schedule's portion |

This is the core of Section 78 "A" computation:
`A = SUM(taxable_amount)` across all schedules.

### 2.25 TaxYear

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| financial_year | TEXT | e.g. "2026-2027" |
| tax_profile_id | INTEGER FK → TaxProfile | |
| status | TEXT | enum: `DRAFT`, `COMPUTED`, `FILED` |
| notes | TEXT | nullable |
| — Income aggregation — | | |
| total_gross_income | INTEGER | *(computed: SUM of all TaxYearIncomeSource.gross_amount)* |
| total_exempted_income | INTEGER | *(computed: SUM of exempted_amount)* |
| standard_deduction_applied | INTEGER | *(computed: MIN(max_exempt, % of Sch-1 income) per SalaryExemptionRule)* |
| total_taxable_income_A | INTEGER | *(computed: SUM(taxable_amount) across schedules)* |
| — Tax before rebate — | | |
| tax_before_rebate | INTEGER | *(computed: apply active TaxSlabs on total_taxable_income_A)* |
| minimum_tax_applied | INTEGER | nullable — min tax amount if floor was triggered |
| — Interest & VAT — | | |
| vat_on_interest | INTEGER | *(computed: sum of govt VAT on DPS/FDR/SavingsCertificates)* |
| — TPS & advance — | | |
| advance_tax_paid | INTEGER | *(computed: SUM of TpsDeduction for this FY)* |
| — Rebate — | | |
| total_rebate_claimed | INTEGER | *(computed: SUM of TaxRebateContribution.rebate_claimed for this FY)* |
| max_possible_rebate | INTEGER | *(computed: per RebateRule)* |
| investment_required_for_max_rebate | INTEGER | *(computed)* |
| — Clawback — | | |
| rebate_clawback_amount | INTEGER | *(computed: SUM of RebateClawbackEvent where source_tax_year_id = this)* |
| — Surcharge — | | |
| surcharge_amount | INTEGER | *(computed: per SurchargeRule on net wealth)* |
| — Filing adjustments — | | |
| filing_incentive_amount | INTEGER | *(computed: per FilingTimelineRule if filed in rebate quarter)* |
| filing_penalty_amount | INTEGER | *(computed: per FilingTimelineRule if late)* |
| — Final — | | |
| net_tax_payable | INTEGER | *(computed: tax_before_rebate − total_rebate_claimed − advance_tax_paid + rebate_clawback + surcharge + penalty − incentive)* |

### 2.26 TaxYearComputationDetail (NEW — 1:1 with TaxYear)

| Field | Type | Notes |
|---|---|---|
| tax_year_id | INTEGER PK/FK → TaxYear | |
| section_78_A | INTEGER | total_taxable_income = SUM(taxable_amount across all schedules) |
| section_78_B | INTEGER | total eligible investment for rebate |
| rebate_computation | TEXT | JSON: { "three_percent_of_a": N, "ten_percent_of_b": N, "max_rebate_cap": N, "lowest": N } |
| slab_breakdown | TEXT | JSON array: [{ "slab": "0-375K", "rate": 0, "amount": N, "tax": 0 }, ...] |
| minimum_tax_checked | INTEGER | the min tax amount if floor was triggered |
| surcharge_wealth_threshold_crossed | BOOLEAN | |
| filing_quarter | INTEGER | 1–4 |
| income_schedule_summary | TEXT | JSON summary of TaxYearIncomeSource values |

### 2.27 TpsDeduction (NEW — monthly Tax at Source tracking)

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| tax_year_id | INTEGER FK → TaxYear | |
| month | INTEGER | 1–12 where 1 = July (BD financial year) |
| amount | INTEGER | TPS deducted that month |
| employer_name | TEXT | nullable |
| notes | TEXT | nullable |
| is_deleted | BOOLEAN | |

### 2.28 RebateClawbackEvent (NEW — cross-year clawback tracking)

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| source_tax_year_id | INTEGER FK → TaxYear | the current year where clawback is added |
| investment_type | TEXT | enum: `DPS`, `SAVINGS_CERTIFICATE` |
| investment_id | INTEGER | FK to Account or SavingsCertificate |
| original_financial_years | TEXT | JSON array of FY strings, e.g. ["2026-2027", "2027-2028", "2028-2029"] |
| original_rebate_claimed_per_fy | TEXT | JSON object { "2026-2027": 12000, "2027-2028": 12000, ... } |
| total_rebate_clawed_back | INTEGER | sum of all original rebates |
| reason | TEXT | e.g. "Early encashment of 5Y BSP on 2029-03-15" |
| created_at | DATETIME | |

### 2.29 TaxRebatePlan

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| tax_year_id | INTEGER FK → TaxYear | |
| financial_year | TEXT | e.g. "2026-2027" |
| max_eligible_amount | INTEGER | govt ceiling for this FY |
| target_investment | INTEGER | user's planned total |
| notes | TEXT | nullable |
| is_active | BOOLEAN | |

### 2.30 TaxRebateContribution (NEW — per-instrument, per-year contribution)

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| tax_year_id | INTEGER FK → TaxYear | direct link (not through plan) so one DPS appears in multiple FYs |
| plan_id | INTEGER FK → TaxRebatePlan | nullable — groups contributions belonging to one plan |
| dps_account_id | INTEGER FK → Account | nullable, must be DPS-type |
| savings_certificate_id | INTEGER FK → SavingsCertificate | nullable |
| monthly_installment | INTEGER | nullable, for DPS |
| start_date | DATE | |
| end_date | DATE | |
| total_investment | INTEGER | *(computed: installment × months or certificate face value)* |
| rebate_claimed | INTEGER | *(computed: from RebateRule applied to total_investment)* |
| notes | TEXT | nullable |

At least one of `dps_account_id` / `savings_certificate_id` must be set.
**Why `tax_year_id` directly**: a 5-year DPS creates 5 rows with different `tax_year_id`, enabling cross-year clawback queries.

### 2.31 WealthCategory (NEW — seeded + user-added)

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| name | TEXT | e.g. "Cash at Bank", "Real Estate", "Motor Vehicle" |
| is_liability | BOOLEAN | true = debt/liability |
| is_system | BOOLEAN | true = seeded by app |
| sort_order | INTEGER | for display |
| is_deleted | BOOLEAN | |

Seeded system categories:

| Name | Is Liability | Computation Source |
|---|---|---|
| Cash at Bank | false | SUM(Account.current_balance) WHERE type SAVINGS/CURRENT |
| FDR & DPS | false | SUM(Account.current_balance) WHERE type FDR/DPS |
| Savings Certificates | false | SUM(SavingsCertificate.face_value) WHERE ACTIVE |
| Market Investments | false | SUM(Investment.principal_amount) WHERE ACTIVE |
| Lent Out | false | SUM(Lent.amount) WHERE direction=LENT_OUT AND status OUTSTANDING |
| Real Estate | false | manual entry only |
| Motor Vehicle | false | manual entry only |
| Jewelry | false | manual entry only |
| Electronic Equipment | false | manual entry only |
| Other Assets | false | manual entry only |
| Home Loan | true | SUM(Account.current_balance) WHERE type=LOAN AND loan_type=HOME |
| Personal Loan | true | SUM(Account.current_balance) WHERE type=LOAN AND loan_type=PERSONAL |
| Auto Loan | true | SUM(Account.current_balance) WHERE type=LOAN AND loan_type=AUTO |
| Borrowed Money | true | SUM(Lent.amount) WHERE direction=BORROWED AND status OUTSTANDING |
| Other Liabilities | true | manual entry only |

### 2.32 WealthAsset (NEW — per-FY wealth statement line item)

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| tax_year_id | INTEGER FK → TaxYear | |
| wealth_category_id | INTEGER FK → WealthCategory | |
| description | TEXT | nullable |
| amount | INTEGER | |
| source | TEXT | enum: `COMPUTED` (auto from accounts), `MANUAL` (user-entered) |
| notes | TEXT | nullable |
| is_deleted | BOOLEAN | |

On TaxYear creation, the app auto-generates WealthAssets for all `COMPUTED` categories. The user reviews, adds manual items (real estate, vehicle, jewelry), and the app:
- Computes `total_net_wealth = SUM(assets) − SUM(liabilities)`
- Checks `total_net_wealth > 40L` → flag "Wealth statement mandatory for NBR e-Return"
- Checks `total_net_wealth > surcharge_threshold` → applies SurchargeRule

### 2.33 TaxRateSet

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| financial_year | TEXT | e.g. "2026-2027" |
| label | TEXT | e.g. "Finance Act 2026 — 5-year roadmap Year 1" |
| is_active | BOOLEAN | only one at a time |
| notes | TEXT | nullable |
| is_deleted | BOOLEAN | |

### 2.34 TaxSlab

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| rate_set_id | INTEGER FK → TaxRateSet | |
| slab_order | INTEGER | evaluated ascending |
| min_income | INTEGER | inclusive |
| max_income | INTEGER | nullable = rest |
| rate_percent | REAL | e.g. 10.0 for 10% |

### 2.35 TaxFreeThreshold

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| rate_set_id | INTEGER FK → TaxRateSet | |
| category_code | TEXT | enum: `GENERAL`, `WOMAN`, `SENIOR_CITIZEN`, `THIRD_GENDER_DISABLED`, `FREEDOM_FIGHTER`, `PARENT_OF_DISABLED` |
| category_label | TEXT | human-readable |
| exempt_amount | INTEGER | |

`PARENT_OF_DISABLED` is an additional exemption on top of the primary category.

### 2.36 SalaryExemptionRule

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| rate_set_id | INTEGER FK → TaxRateSet | |
| max_exempt_amount | INTEGER | e.g. 500,000 BDT |
| percent_of_salary | REAL | e.g. 33.33 |
| description | TEXT | nullable |

Exemption applied ONLY to Sch-1 (employment) income. Other schedules are not eligible.

### 2.37 RebateRule

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| rate_set_id | INTEGER FK → TaxRateSet | |
| label | TEXT | e.g. "Section 78 — Finance Act 2026" |
| rate_percent_of_A | REAL | % of total taxable income, e.g. 3.0 |
| rate_percent_of_B | REAL | % of eligible investment, e.g. 10.0 |
| max_rebate_amount | INTEGER | absolute cap, e.g. 750,000 BDT |

Rebate = lowest of:
- round(A × rate_percent_of_A / 100)
- round(B × rate_percent_of_B / 100)
- max_rebate_amount

### 2.38 EligibleInvestment

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| rate_set_id | INTEGER FK → TaxRateSet | |
| instrument_code | TEXT | enum: `DPS`, `5Y_BSP`, `3Y_BSP`, `FAMILY_SAVINGS`, `PENSIONER`, `MONTHLY_PROFIT`, `LIFE_INSURANCE`, `PF`, `GPS`, `PENSION`, `GOVT_SECURITIES`, `MUTUAL_FUND`, `LISTED_SHARES`, `ICB_UNIT`, `CHARITABLE_DONATION`, `UNIVERSAL_PENSION` |
| instrument_name | TEXT | user-facing label |
| filer_category | TEXT | enum: `FIRST_TIME`, `REPEAT`, `ALL` |
| max_annual_limit | INTEGER | nullable |
| min_purchase_amount | INTEGER | nullable |
| is_joint_allowed | BOOLEAN | |
| joint_max_annual_limit | INTEGER | nullable |
| eligibility_rules | TEXT | nullable JSON: { "min_age": 50, "max_age": null, "residency": "BANGLADESH", "gender": null } |
| is_active | BOOLEAN | |
| description | TEXT | nullable |

### 2.39 SurchargeRule (NEW)

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| rate_set_id | INTEGER FK → TaxRateSet | |
| threshold_low | INTEGER | starting net wealth for this bracket |
| threshold_high | INTEGER | nullable — next bracket start |
| surcharge_rate | REAL | e.g. 10.0 for 10% |
| description | TEXT | nullable |

Example (from Finance Act 2026):
- Net wealth 4–8 Cr → 10% surcharge on tax
- Net wealth 8–12 Cr → 20%
- Net wealth 12–20 Cr → 30%
- Net wealth > 20 Cr → 35%

### 2.40 MinimumTaxRule

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| rate_set_id | INTEGER FK → TaxRateSet | |
| location_code | TEXT | enum: `MAJOR_CITY`, `OTHER_CITY`, `OTHER_AREA`, `NEW_TAXPAYER` |
| location_label | TEXT | e.g. "Dhaka & Chattogram city corporation" |
| min_tax_amount | INTEGER | e.g. 5,000 BDT |

### 2.41 FilingTimelineRule

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| rate_set_id | INTEGER FK → TaxRateSet | |
| quarter_number | INTEGER | 1–4 |
| quarter_label | TEXT | |
| start_month | INTEGER | |
| end_month | INTEGER | |
| rebate_percent | REAL | nullable |
| rebate_max_amount | INTEGER | nullable |
| penalty_percent | REAL | nullable |
| penalty_max_amount | INTEGER | nullable |

---

## 3. Relationship Summary

```
Bank                     1───* Branch
Bank                     1───* Account
Bank                     1───* SavingsCertificate
Bank                     1───* SmsParsingRule
Branch                   1───* Account

Account                  1───0..1 AccountTermDetail    (FDR/DPS only)
Account                  1───0..1 LoanDetail           (LOAN only)
Account                  1───0..* Card                 (0..1 for debit)
Account                  1───* Transaction
Account                  1───* Statement
Account                  1───* Lent
Account                  1───* Investment (as funding)
Account                  1───* TaxRebateContribution   (DPS leg)

LoanDetail               1───* EmiAmortizationSchedule

Card                     1───* Transaction
Card                     1───* Statement

TaxSpendingHead          1───* Category
Category                 1───* Transaction
Category                 1───* Category (self, parent/child)

TaxRateSet               1───* TaxSpendingHead
TaxRateSet               1───* Category (system-seeded)
TaxRateSet               1───* TaxSlab
TaxRateSet               1───* TaxFreeThreshold
TaxRateSet               1───* SalaryExemptionRule
TaxRateSet               1───* RebateRule
TaxRateSet               1───* EligibleInvestment
TaxRateSet               1───* SurchargeRule           (NEW)
TaxRateSet               1───* MinimumTaxRule
TaxRateSet               1───* FilingTimelineRule

TaxProfile               1───* TaxYear
TaxYear                  1───* TaxYearIncomeSource     (NEW — income per schedule)
TaxYear                  1───1 TaxYearComputationDetail (NEW — full computation trace)
TaxYear                  1───* TpsDeduction            (NEW — monthly TPS tracking)
TaxYear                  1───* WealthAsset             (NEW — per-FY wealth items)
TaxYear                  1───* TaxRebatePlan
TaxYear                  1───* TaxRebateContribution   (per-instrument, per-year)
TaxYear                  *───* SavingsCertificate      (via TaxRebateContribution)
TaxYear                  1───* RebateClawbackEvent     (NEW — clawback source)

TaxRebatePlan            1───* TaxRebateContribution   (grouping)

WealthCategory           1───* WealthAsset              (NEW)

Statement                1───* StatementLine
StatementLine            1───0..1 Transaction          (reconciliation match)

InvestmentCompany        1───* Investment

SmsParsingRule           1───* ParsedSmsLog
ParsedSmsLog             1───0..1 Transaction

Transaction              1───1 Transaction             (self, transfer pairing)
Transaction              *───* Tag                      (via TransactionTag)

Nominee (polymorphic)    *───1 Account or SavingsCertificate
Attachment (polymorphic) *───1 Account / Investment / Lent / Transaction / SavingsCertificate
```

---

## 4. Tax Computation Engine

### 4.1 Trigger

- Manual: user taps "Compute Tax" on TaxYear screen
- Automatic: after each new Transaction is recorded, the engine re-runs for the current FY (status stays DRAFT until user reviews)
- Start-of-year: automatic estimate run when user sets estimated income for the FY

### 4.2 Step-by-step Logic

```
INPUT:
  - Active TaxRateSet (is_active = true)
  - All child configs (TaxSlab, TaxFreeThreshold, SalaryExemptionRule, RebateRule,
    EligibleInvestment, SurchargeRule, MinimumTaxRule, FilingTimelineRule)
  - TaxProfile (taxpayer_category, employment_type)
  - TaxYearIncomeSource rows (income by schedule)
  - TaxRebateContribution rows (investments for this FY)
  - WealthAsset rows (net wealth)
  - TpsDeduction rows (monthly TPS)
  - Previous FY data (for clawback check)

STEP 1 — Compute total income by schedule
  For each TaxYearIncomeSource:
    taxable_amount = gross_amount - exempted_amount - reduced_rate_amount
                     - minimum_tax_amount - final_tax_amount
    Write tax_computed after slab application (Step 3)

STEP 2 — Section 78 "A"
  A = SUM(taxable_amount) across all schedules
  Special: firm_aop_income is taxed separately, excluded from A

STEP 3 — Apply standard deduction
  IF taxpayer has employment income (Sch-1):
    standard_deduction = MIN(
      SalaryExemptionRule.max_exempt_amount,
      ROUND(Sch-1.gross_amount × SalaryExemptionRule.percent_of_salary / 100)
    )
    A = A - standard_deduction
  ELSE:
    standard_deduction = 0

STEP 4 — Apply tax slabs
  remaining = A
  tax_before_rebate = 0
  FOR slab IN TaxSlabs ORDER BY slab_order:
    bracketed = MIN(remaining, slab.max_income - slab.min_income) or remaining
    tax_before_rebate += ROUND(bracketed × slab.rate_percent / 100)
    remaining -= bracketed
    BREAK IF remaining <= 0

STEP 5 — Check tax-free threshold
  threshold = TaxFreeThreshold WHERE category_code = TaxProfile.taxpayer_category
  IF A <= threshold.exempt_amount:
    tax_before_rebate = 0
    GOTO STEP 9

STEP 6 — Apply minimum tax
  min_tax = MinimumTaxRule WHERE location_code matches user's city type
  IF tax_before_rebate < min_tax.min_tax_amount:
    minimum_tax_applied = min_tax.min_tax_amount
    tax_before_rebate = min_tax.min_tax_amount
  ELSE:
    minimum_tax_applied = null

STEP 7 — Compute rebate
  B = SUM(TaxRebateContribution.total_investment for this FY)
  rebate_candidates = [
    ROUND(A × RebateRule.rate_percent_of_A / 100),
    ROUND(B × RebateRule.rate_percent_of_B / 100),
    RebateRule.max_rebate_amount
  ]
  total_rebate_claimed = MIN(rebate_candidates)
  max_possible_rebate = MIN(rebate_candidates)
  investment_required = CEILING(max_possible_rebate / (RebateRule.rate_percent_of_B / 100))

STEP 8 — Compute rebate clawback (cross-year)
  rebate_clawback = SUM(RebateClawbackEvent WHERE source_tax_year_id = this)
  IF any investment broken early this FY:
    For each broken DPS/SavingsCertificate:
      Find all TaxRebateContribution WHERE dps_account_id = X OR savings_certificate_id = X
        AND tax_year_id != current
      Sum rebate_claimed across all those
      Create RebateClawbackEvent with the total

STEP 9 — Compute surcharge
  total_net_wealth = SUM(WealthAsset WHERE is_liability=false) -
                     SUM(WealthAsset WHERE is_liability=true)
  surcharge_amount = 0
  FOR surcharge IN SurchargeRule ORDER BY threshold_low ASC:
    IF total_net_wealth >= surcharge.threshold_low:
      surcharge_amount = ROUND(tax_before_rebate × surcharge.surcharge_rate / 100)
      BREAK

STEP 10 — Filing incentive/penalty
  (determined by user selecting which quarter they filed or plan to file)
  rule = FilingTimelineRule WHERE quarter_number = user's filing quarter
  IF rule.rebate_percent:
    filing_incentive = MIN(
      ROUND(tax_before_rebate × rule.rebate_percent / 100),
      rule.rebate_max_amount
    )
  IF rule.penalty_percent:
    filing_penalty = MIN(
      ROUND(tax_before_rebate × rule.penalty_percent / 100),
      rule.penalty_max_amount
    )

STEP 11 — Final
  vat = SUM(vat_amount from AccountTermDetail + SavingsCertificate for this FY)
  advance_tax = SUM(TpsDeduction.amount for this FY)
  net_tax_payable = tax_before_rebate - total_rebate_claimed - advance_tax
                    + rebate_clawback + surcharge_amount + filing_penalty
                    - filing_incentive

  IF net_tax_payable < 0: net_tax_payable = 0 (refund scenario — TPS overpaid)

OUTPUT:
  Populate all fields in TaxYear, TaxYearComputationDetail.
  Status remains DRAFT until user reviews and confirms.
```

### 4.3 Edge Cases

| Case | Handling |
|---|---|
| User has no income this FY | All zeros. Status = DRAFT. No filing required. |
| User has only exempt income (e.g. agriculturist under threshold) | tax_before_rebate = 0. No filing required. |
| TPS deducted > final tax liability | net_tax_payable = 0. Excess TPS carries forward to next FY (track in TaxYear notes). |
| Multiple employers | Sch-1 income = sum of all salaries. Each TPS deduction tracked separately. |
| Investment broken mid-year | RebateClawbackEvent created. Original investment removed from current FY's B computation. |
| Sanchayapatra matured (not encashed early) | No clawback. Profit credited to Income from Financial Assets. |
| User didn't invest anything for rebate | B = 0. rebate = 0. net_tax_payable = tax_before_rebate - advance_tax. |

---

## 5. e-Return Export Design

The app never files directly (NBR e-Return is web-only). Instead it produces an export file:

### 5.1 JSON Export Schema

```json
{
  "financial_year": "2026-2027",
  "taxpayer": {
    "tin": "123456789012",
    "circle": "E-Tower, Dhaka",
    "zone": "Tax Zone-3",
    "category": "GENERAL"
  },
  "income": [
    { "schedule": "Sch-1", "head": "Salary", "gross": 96000000, "exempt": 32000000, "taxable": 64000000 },
    { "schedule": "Sch-5", "head": "Capital Gain", "gross": 5000000, "exempt": 0, "taxable": 5000000 }
  ],
  "expenditure_IT10BB": [
    { "serial": 1, "head": "Personal & Family Fooding...", "amount": 24000000 },
    { "serial": 2, "head": "Housing Expense", "amount": 12000000 }
  ],
  "tax_computation": {
    "total_income": 101000000,
    "exempted": 32000000,
    "standard_deduction": 32000000,
    "taxable_income_A": 69000000,
    "tax_before_rebate": 2650000,
    "rebate_claimed": 1920000,
    "surcharge": 0,
    "advance_tax": 2200000,
    "net_payable": 20000
  },
  "investments": [
    { "type": "DPS", "amount": 12000000, "rebate_claimed": 1200000 },
    { "type": "5Y_BSP", "amount": 7200000, "rebate_claimed": 720000 }
  ],
  "wealth_statement": {
    "total_assets": { "computed": 500000000, "manual": 350000000, "total": 850000000 },
    "total_liabilities": { "computed": 200000000, "manual": 0, "total": 200000000 },
    "net_wealth": 650000000,
    "requires_filing": true,
    "surcharge_applicable": false
  }
}
```

### 5.2 Printable IT-10BB

The app generates a PDF with the 9 IT-10BB expenditure heads filled from `Category → TaxSpendingHead(IT-10BB:N)` aggregations, plus income schedule summary, tax computation, and investment summary. Format matches NBR's paper form layout.

---

## 6. PDF Statement Parsing Architecture

### 6.1 Design Principle

Each Bangladeshi bank issues statement PDFs in a different layout — and they change layouts without notice. The parser must **never silently drop data**. Every PDF parse, successful or not, is logged and every extracted line is stored.

### 6.2 Pipeline

```
Upload PDF → Store raw file (Statement) → Detect bank/format → Try parsers in order
  ├── Parser matches → Extract lines → Store StatementLines → Auto-match against Transactions
  └── No parser matches → Status = HAS_ISSUES → Notify user → Manual mapping UI
```

### 6.3 Parser Registry

| Bank | Detection Method | Status |
|---|---|---|
| EBL | Text pattern "Eastern Bank" | TODO |
| SCB | Text pattern "Standard Chartered" | TODO |
| DBBL | Text pattern "Dutch-Bangla" | TODO |
| Brac Bank | Text pattern "BRAC Bank" | TODO |
| City Bank | Text pattern "City Bank" | TODO |
| (Generic) | PDF table extraction fallback | TODO |

### 6.4 Graceful Degradation

- **Unparseable PDF**: status = `HAS_ISSUES`. User can view the raw PDF inside the app and manually enter lines.
- **Partial parse**: some lines extracted, some failed. Logged with line numbers. User reviews and fills gaps.
- **Format changed**: old parser fails, all lines go to `UNPARSED`. Admin creates new `SmsParsingRule`-like rule for statement PDFs.

The PDF parsing engine is deferred to Phase 4 of the build order. Phase 1–3 focus on manual entry and SMS parsing.

---

## 7. Build Order (updated)

| Phase | Entities | Depends On |
|---|---|---|
| **1 — Ledger Core** | Bank, Branch, Account, AccountTermDetail, LoanDetail, Category, Transaction, Card, Nominee, EmiAmortizationSchedule | — |
| **2 — Richer Tracking** | SavingsCertificate, InvestmentCompany, Investment, Lent, Attachment, TaxProfile | Phase 1 |
| **3 — SMS Automation** | SmsParsingRule, ParsedSmsLog | Phase 1 |
| **4 — Statement Recon** | Statement, StatementLine, PDF parse engine | Phase 1, 3 |
| **5 — Tax Engine** | TaxRateSet + all child configs, TaxYear, TaxYearIncomeSource, TaxYearComputationDetail, TpsDeduction, RebateClawbackEvent, TaxRebatePlan, TaxRebateContribution, WealthCategory, WealthAsset, SurchargeRule, tax computation engine | Phase 1, 2 |
| **6 — Automation Layer** | RecurringTransactionTemplate, Tag, TransactionTag, budget/insight views | Phase 1 |

---

## 8. Open Decisions (settled)

| Decision | Resolution |
|---|---|
| ID strategy | `INTEGER` autoincrement. Migrate to UUID only if multi-device sync is ever needed. |
| Multi-profile | Out of scope. One user, one `TaxProfile`. |
| Field-level encryption | Not needed — full DB encryption via sqlcipher covers it. |
| Soft delete | Yes — `is_deleted` on all entities. Hard delete never used. |
| CVV persistence | Never stored. |
| e-Return filing | Export only. App does not connect to NBR's portal. |
| PDF statement vs CSV | PDF only. No CSV import. |

---

## 9. Key Design Decisions (why)

1. **TaxYearIncomeSource as child table, not columns on TaxYear** — NBR can add/remove income schedules between FYs. A child table with FK → TaxSpendingHead is flexible without schema migration.

2. **TaxRebateContribution links to TaxYear directly, not through TaxRebatePlan** — a 5-year DPS creates one row per FY. Querying "what did this DPS contribute across all years" is a simple SELECT without joining through plans.

3. **WealthAsset per FY, not live** — wealth statement is a snapshot at FY-end. Next year's wealth is a new set of rows, not live-updating.

4. **RebateClawbackEvent as separate entity** — audit trail. If an investment was broken in FY 2029-30, the clawback needs to be traceable back to the original rebates claimed in 2026-27, 2027-28, 2028-29.

5. **Section 78 "A" uses `taxable_amount` per schedule, not total_income** — exempted (festival bonus), reduced-rate, minimum-tax, final-tax, and firm/AoP income are excluded from A. Each has different rules per schedule.
