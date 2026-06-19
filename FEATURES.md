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
As a user, I want to enter my approximate total salary for a tax year and see a full tax breakdown — total income, exempted income, taxable income, tax before rebate, maximum rebate possible, and the investment required to achieve it — with suggestions for DPS/Sanchayapatra based on time remaining in the year. As an admin, I want to update tax parameters (slabs, rates, rebate percentages, limits) every year without modifying the app code.

### Acceptance Criteria

**Tax Computation (user-facing):**
1. User inputs gross annual salary (and optionally interest/other income).
2. App shows:
   - Total income (salary + interest + other)
   - Exempted income (standard deduction = lower of `max_amount` or `percent_of_salary` of gross)
   - Taxable income (total − exempted)
   - Tax before rebate (computed against current FY slabs)
   - Maximum possible rebate (lowest of: 3% of A, `rebate_pct_B`% of required investment, or max rebate cap)
   - Investment required for max rebate: maximum_rebate ÷ (`rebate_pct_B` / 100)
3. If user has partially or fully invested (DPS / Sanchayapatra tracked in app), show:
   - Already invested
   - Remaining gap to max rebate
   - Estimated rebate at current investment

**Investment Suggestions:**
4. Based on months remaining in the tax year, suggest specific investments:
   - Sanchayapatra (safest, government-backed, lump sum) — recommended first
   - DPS (bank deposit pension scheme) — monthly installments, max Tk 1.2 lakh/year
   - Monthly capacity if DPS: gap ÷ months_remaining
5. For each suggestion, show: instrument name, total to invest, monthly amount (for DPS), maturity requirements (hold to maturity for rebate).

**Admin Config (settings panel):**
6. `TaxRateSet` per financial year with:
   - Progressive slab boundaries and rates
   - Tax-free thresholds by taxpayer category
   - Salary exemption rule (max amount, percent of salary)
   - Rebate rule (% of A, % of B, max rebate cap)
   - Eligible investment instruments with per-instrument annual limits
   - Minimum tax by location type
   - Filing timeline incentives/penalties per quarter
7. Admin activates one `TaxRateSet` per current FY; app uses it for all computations.
8. Year-end rollover: duplicate previous config, adjust numbers, activate.

### Notes
- Tax laws for AY 2026-27 & 2027-28 (5-year forward roadmap published in Budget June 2026):
  - Slabs: 0% up to 375K, 10%/300K, 15%/400K, 20%/500K, 25%/2M, 30% rest
  - Rebate: lowest of 3% of A, 10% of B (reduced from 15%), or Tk 7.5L (reduced from Tk 10L)
  - Standard deduction: lower of Tk 5,00,000 or ⅓ of gross salary
  - DPS max: Tk 1.2L/yr; Govt securities max: Tk 5L
  - Minimum tax: Tk 5,000 (general); Tk 1,000 (new taxpayer)
- Filing timeline: Q1 (Jul-Sep) = 5% rebate up to Tk 25K; Q2 = neutral; Q3 = 2% penalty up to Tk 3K; Q4 = 5% penalty up to Tk 5K
- Rebate eligible investments must be held until maturity; early withdrawal forfeits rebate (added as tax in withdrawal year).

---

## Schema changes

### New entities (v1.0)

**Statement**, **StatementLine** — see Feature 1 above.

**TaxRateSet** — master config per financial year; admin activates one at a time.
**TaxSlab**, **TaxFreeThreshold**, **SalaryExemptionRule**, **RebateRule**, **EligibleInvestment**, **MinimumTaxRule**, **FilingTimelineRule** — config entities consumed by the tax computation engine.

### Modified entities

**Transaction** — added `statement_line_id`, `reconciled_at`.
**TaxYear** — added `exempted_income`, `tax_before_rebate`, `max_possible_rebate`, `investment_required_for_max_rebate`.
