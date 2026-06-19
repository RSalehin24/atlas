# Features

> **Living document** — features are captured incrementally during chat sessions.
> Added by the user as they're discussed and refined.

---

## 1. Bank Statement Reconciliation

### User Story
As a user, I want to upload my monthly bank/credit card/DPS/loan PDF statements so the app can reconcile them against SMS-parsed transactions and correct any discrepancies, because the bank statement is the final source of truth for all transactions.

### Acceptance Criteria

1. User can upload a PDF bank statement for a specific Account (Savings/Current/DPS/FDR/LOAN) or Credit Card for a given month+year.
2. The app parses the PDF into individual StatementLines (date, description, amount, running balance, counterparty).
3. The app auto-matches each StatementLine against existing Transactions on the same Account/Card. Matching criteria:
   - Exact amount match
   - Same transaction date
   - Timestamps within minutes of each other (rarely hours)
4. Statement is final say:
   - Matched lines → update Transaction values (amount, date, description, counterparty) from the statement
   - Unmatched lines → create new Transaction with `source = STATEMENT_IMPORT`
   - SMS-parsed Transactions not on the statement → flagged for manual review
5. Uploaded PDF is stored permanently as a Statement record with file hash to prevent duplicate uploads.
6. All StatementLines persist as an audit trail with match_status, confidence score, and link to the resulting Transaction.
7. User can manually match/unmatch statement lines to transactions in a reconciliation UI.
8. User can re-upload a corrected statement (replaces the previous one for that account+month, marks old lines as superseded).

### Notes
- PDF-only for now (no CSV/Excel). Parsing layer must degrade gracefully.
- `STATEMENT_IMPORT` already exists as a `Transaction.source` enum value.
- Reconciliation is a batch operation per account+month, not real-time.

---

## 2. Tax Rebate Planner & Admin Config

### User Story
At the start of a tax year (July), I want to enter my estimated/probable salary for the year and
see a full projection — income, exemptions, tax, maximum rebate, and a monthly investment plan to
hit the rebate target using my existing DPS accounts or new Sanchayapatra. As the year progresses,
actual income from SMS/statements refines the estimate. As an admin, I update tax parameters once
per year without touching code.

### Acceptance Criteria

**Start-of-year planning (July):**
1. User enters an estimated annual income (or the app carries forward last year's figure).
2. App computes and displays:
   - Total estimated income
   - Exempted income (standard deduction per active `SalaryExemptionRule`)
   - Taxable income
   - Tax before rebate (applied against active `TaxSlabs`)
   - Maximum possible rebate (lowest of 3% of A, `RebateRule.rate_percent_of_B`% of B, or `RebateRule.max_rebate_amount`)
   - Investment required for max rebate
3. App checks what the user already has in place for the current year:
   - Existing DPS accounts with their monthly installment amounts
   - Sanchayapatra holdings already purchased
   - Any other rebate-eligible investments tracked in `TaxRebateContribution`
4. App calculates the gap:
   - Already-invested total for the year
   - Remaining gap = investment_required − already_invested
5. App suggests a monthly plan for the rest of the tax year:

   | Step | Instrument | Why |
   |---|---|---|
   | 1st | DPS — increase existing or open new | Max Tk 1.2L/yr (Tk 10K/mo). Monthly, manageable, most common. |
   | 2nd | Sanchayapatra (lump sum) | Govt-backed, safest, no monthly commitment. Fills any gap DPS can't cover. |
   | 3rd | Govt securities / Mutual funds | Only if both DPS and Sanchayapatra maxed out. |

   For each suggestion:
   - Instrument name
   - Total amount to invest
   - Monthly installment amount (for DPS)
   - Number of months remaining
   - Maturity requirement reminder (must hold to maturity per Finance Act 2026)

**Throughout-the-year tracking:**
6. As SMS-parsed and statement-reconciled transactions accumulate, the app tracks actual income
   against the estimate.
7. If actual income diverges significantly (e.g. salary raise, bonus received), the app:
   - Recomputes the tax projection
   - Warns the user: "You're on track to earn Tk X more than estimated — your tax will increase
     by Tk Y. Consider investing Tk Z more in DPS/Sanchayapatra."
8. At any point, the user can update the estimate manually (e.g. after a promotion or job change).

**Admin Config (settings panel):**
9. `TaxRateSet` per financial year with:
   - Progressive slab boundaries and rates
   - Tax-free thresholds by taxpayer category
   - Salary exemption rule (max amount, percent of salary)
   - Rebate rule (% of A, % of B, max rebate cap)
   - Eligible investment instruments with per-instrument annual limits
   - Minimum tax by location type
   - Filing timeline incentives/penalties per quarter
10. Admin activates one `TaxRateSet` per current FY; app uses it for all computations.
11. Year-end rollover: duplicate previous config, adjust numbers, activate.

### Notes
- The planner is an **approximation** — it uses estimated income at the start and refines as real
  data comes in. The final `TaxYear` record is still filed with actuals after June 30.
- Tax laws for AY 2026-27 & 2027-28 (5-year forward roadmap published in Budget June 2026):
  - Slabs: 0% up to 375K, 10%/300K, 15%/400K, 20%/500K, 25%/2M, 30% rest
  - Rebate: lowest of 3% of A, 10% of B (reduced from 15%), or Tk 7.5L (reduced from Tk 10L)
  - Standard deduction: lower of Tk 5,00,000 or ⅓ of gross salary
  - DPS max: Tk 1.2L/yr (repeat filer); Tk 80K/yr or similar (first-time filer)
  - 5Y BSP solo max: Tk 3,00,000; joint max: Tk 6,00,000
  - Minimum tax: Tk 5,000 (general); Tk 1,000 (new taxpayer)
- Filing timeline: Q1 (Jul-Sep) = 5% rebate up to Tk 25K; Q2 = neutral; Q3 = 2% penalty up to Tk 3K; Q4 = 5% penalty up to Tk 5K
- Rebate eligible investments must be held until maturity; early withdrawal forfeits rebate (added as tax in withdrawal year).
- DPS recommendation priority: safest option first. Sanchayapatra (govt guaranteed) > DPS (bank
  deposit, insured) > market-linked instruments.

**Example — correct NBR calculation for Tk 9,60,000 gross salary (AY 2026-27):**

| Step | Calculation | Amount |
|---|---|---|
| Gross salary | | Tk 9,60,000 |
| Standard deduction | MIN(5,00,000, 9,60,000 ÷ 3 = 3,20,000) | Tk 3,20,000 |
| Taxable income | 9,60,000 − 3,20,000 | Tk 6,40,000 |
| Tax on first 3,75,000 | @ 0% | Tk 0 |
| Tax on next 3,00,000 (3,75,001–6,75,000) | @ 10% | Tk 30,000 |
| Actually taxable is 6,40,000 — falls within this slab | 6,40,000 − 3,75,000 = 2,65,000 @ 10% | Tk 26,500 |
| Tax before rebate | | **Tk 26,500** |
| A = total taxable income (excl. exempted) | 6,40,000 | |
| 3% of A | | Tk 19,200 |
| Max rebate cap | from `RebateRule` | Tk 7,50,000 |
| Max possible rebate = MIN(19,200, 10% of B, 7,50,000) | depends on B | ≤ Tk 19,200 |
| Investment needed for max rebate (Tk 19,200) | 19,200 ÷ 10% | Tk 1,92,000 |
| DPS capacity (repeat filer, max 1.2L/yr) | | Tk 1,20,000 |
| DPS rebate: 10% of 1,20,000 | | Tk 12,000 |
| Remaining rebate gap | 19,200 − 12,000 | Tk 7,200 |
| Additional investment needed | 7,200 ÷ 10% | Tk 72,000 |
| **Suggestion:** DPS Tk 10K/mo for 12 mo → Tk 1.2L → rebate Tk 12,000 | | |
| **Plus:** 5Y BSP Tk 72,000 (lump sum, solo, within 3L limit) → rebate Tk 7,200 | | |
| **Total rebate claimed** | 12,000 + 7,200 | Tk 19,200 |
| **Net tax payable** | 26,500 − 19,200 | **Tk 7,300** |

If already paying an EMI of Tk 5,000/month → monthly DPS capacity reduces. Planner adjusts:
- Available monthly cash = salary − expenses − EMI
- If EMI leaves room for only Tk 5,000/mo DPS → DPS total = Tk 60,000 → rebate Tk 6,000
- Remaining Tk 13,200 rebate needs Tk 1,32,000 additional → 5Y BSP Tk 72,000 + remaining Tk 60,000 via other instruments
- Planner warns: "Your EMI commitments limit DPS to Tk 60K/yr. Consider higher Sanchayapatra allocation."

Sanchayapatra purchase eligibility:
- **5Y BSP**: min Tk 1,000, max Tk 3,00,000 (solo) / Tk 6,00,000 (joint). Open to all individuals.
- **Pensioner**: minimum age 50. Max varies.
- **Family Savings**: requires minor child as second holder.
- The app checks eligibility against certificate category rules before suggesting.

---

## Schema changes (v2 — comprehensive redesign)

### New entities

**Core Ledger**: `EmiAmortizationSchedule` — auto-splits EMI into interest + principal.

**Tax Engine**:
- `TaxYearIncomeSource` — income breakdown by NBR schedule (Sch-1 through Sch-5, Financial Assets, Foreign, etc.) enabling proper Section 78 "A" computation
- `TaxYearComputationDetail` — full audit trail of the tax computation (slab breakdown, rebate formula, surcharge check)
- `TpsDeduction` — monthly Tax at Source tracking, reconciled against computed tax
- `RebateClawbackEvent` — cross-year rebate forfeiture tracking when an investment is broken early (Finance Act 2026)
- `TaxRebateContribution` — links to `TaxYear` directly (not through `TaxRebatePlan`) so a 5-year DPS creates rows in 5 different FYs for clawback queries
- `WealthCategory` — seeded asset/liability categories for NBR wealth statement
- `WealthAsset` — per-FY wealth snapshot line items (auto-computed from accounts + manual entries)
- `SurchargeRule` — progressive surcharge brackets based on net wealth (4Cr+, 8Cr+, etc.)

**Config**: `SurchargeRule` — 9th child entity under `TaxRateSet`.

### Modified entities

| Entity | Change |
|---|---|
| **Transaction** | Added `statement_line_id`, `reconciled_at` |
| **TaxYear** | Replaced coarse income fields with `TaxYearIncomeSource` child table. Added `standard_deduction_applied`, `minimum_tax_applied`, `rebate_clawback_amount`, `surcharge_amount`, `filing_incentive_amount`, `filing_penalty_amount` |
| **Nominee** | Polymorphic: `owner_type` + `owner_id` supports both Account and SavingsCertificate |
| **TaxProfile** | Added `taxpayer_category` enum (maps to `TaxFreeThreshold`) |
| **TaxRebatePlan** | Simplified — removed per-instrument detail (moved to `TaxRebateContribution`) |
| **TaxRebateContribution** | Links directly to `TaxYear` instead of through `TaxRebatePlan`. Carries `rebate_claimed` |
| **SalaryExemptionRule** | Standard deduction now applies ONLY to Sch-1 employment income |

### Year-scoped entities

`TaxSpendingHead` and `Category` carry `rate_set_id FK → TaxRateSet`. Every FY defines its own NBR heads and seeded categories. Admin duplicates previous config, adjusts for Finance Act changes, activates new set. User-created categories (`rate_set_id = null`) persist across years.

### v2 design principles

1. **Income schedules drive tax**: `TaxYearIncomeSource` per schedule enables Section 78 "A" = sum of taxable amounts after per-schedule exemptions, exclusions, and reduced-rate/final-tax adjustments.
2. **Multi-year tracking**: `TaxRebateContribution` links to `TaxYear` directly (not through plan), so querying "all rebates claimed by this DPS across all FYs" is a simple SELECT for clawback computation.
3. **Computation is audit-trailed**: `TaxYearComputationDetail` stores slab breakdown, rebate formula candidates, and surcharge check as JSON — user can see how every number was derived.
4. **Wealth statement is a snapshot**: Auto-computed from accounts at FY-end, supplemented by manual entries (real estate, vehicles, jewelry). Stored per FY so next year's wealth is a fresh snapshot.
