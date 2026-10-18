# Personal Financial Atlas — Detailed Application Plan

## 1. Goal and agreed scope

Build a personal finance and tax preparation application with **full functionality on Android and web**, sharing one encrypted financial record.

**Delivery priority: financial management first, with NBR-compatible expenditure capture included.** Build transaction capture from SMS, emails, statements, and manual entries, then accounts, reconciliation, loans, lending, investments, charges, waivers, and cash-flow planning. The financial release must classify expenditure under official NBR heads, track relevant asset acquisitions, and produce source-backed expenditure summaries. Deliver this before automated tax-liability computation and full return preparation. Those later features consume the reviewed financial record; daily financial use must not depend on configuring the later tax engine. Explicit reporting-period dates and verified catalog applicability are required for a complete NBR expenditure summary.

See [FINANCIAL_USER_JOURNEYS.md](FINANCIAL_USER_JOURNEYS.md) for the detailed onboarding, input forms, financial user stories, and acceptance criteria.

The application should answer four questions:

- Where did my money come from, and where did it go?
- What do I own, owe, expect to receive, and need to pay?
- Which fees, waivers, cashback, and rewards apply to my financial products?
- Is my tax return complete, explainable, and ready for me to submit?

### Confirmed decisions

| Area | Decision |
|---|---|
| Audience | Personal use: one Google account and one taxpayer profile |
| Platforms | Android and responsive web |
| Synchronization | Firebase |
| Backups | Encrypted backups in Google Drive |
| Privacy | Encrypt private financial information before upload |
| Unlocking | Vault passphrase or supported vault-unlock passkey saved through Google Password Manager or Apple Passwords; recovery key and optional Android biometric unlock |
| Processing | Financial parsing, classification, and calculations happen on devices |
| Automation | Automatically handle established rules; review exceptions |
| Rule changes | Propose updates with sources; require approval before activation |
| NBR expenditure coverage | Official expenditure heads, mapped detail categories, asset-acquisition treatment, versioned periods, and expenditure summaries in the financial release |
| Later tax coverage | Reconstruct the two supplied income years and prepare the next applicable full return |
| Submission | Prepare a complete package and portal-entry guide; you submit |
| Android distribution | Private installation initially |
| Input sources | SMS, financial emails and attachments, PDF statements, manual entries |

Gmail is the initial email-provider default, subject to confirmation. Multiple connected Gmail mailboxes may feed the same personal vault; signing into the application and granting access to each mailbox are separate actions. Other providers require their own connector and authorization design.

The first release covers personal financial management and its NBR-compatible expenditure reporting. Full individual return preparation follows later. Public onboarding, business bookkeeping, direct bank connections, and automatic NBR submission are outside this release.

## 2. Architecture, storage, and security

### Application structure

Use **Flutter and Dart** for a shared application and calculation engine. Adapt navigation and layouts for mobile and desktop screens. Flutter supports Android and browser applications from a shared codebase. [Flutter platform documentation](https://docs.flutter.dev/platform-integration)

Separate the system into:

- **Presentation:** screens, forms, review queues, reports.
- **Domain:** ledger, repayment calculations, classification, tax rules, return validation.
- **Data:** encrypted persistence, import adapters, synchronization, backups.
- **Platform adapters:** Android SMS and biometrics; browser file handling and local processing.

Use dependency injection at these boundaries. Keep tax and financial calculations independent of Flutter, Firebase, and document libraries.

### Google services

| Service | Responsibility |
|---|---|
| Firebase Authentication | Google sign-in |
| Cloud Firestore | Encrypted records, revisions, synchronization metadata |
| Cloud Storage for Firebase | Encrypted statements, receipts, and other attachments |
| Firebase Hosting | Web application |
| Google Drive | Encrypted backup archives |
| Gmail API | Optional, read-only access to connected financial emails and attachments |
| Scheduled Google Cloud task | Check approved public rule-source pages for changes |

The scheduled task may inspect public NBR publications and bank terms. It must never receive private financial documents or vault keys.

Use the narrow `drive.file` permission for app-created backups and explicitly selected restore files. Do not request access to the entire Drive. [Google Drive scope guidance](https://developers.google.com/workspace/drive/api/guides/api-specific-auth)

### Encryption and unlocking

- Generate a random vault encryption key on the device.
- Encrypt records and attachments with authenticated AES-256-GCM through an established cryptographic library.
- Use a fresh nonce for each encryption and bind ciphertext to its record ID, schema version, and key version.
- Wrap the vault key separately using the passphrase-derived key and a randomly generated recovery key.
- Support Google Password Manager and Apple Passwords as either primary credential manager, using the same application profile. For Google, guide saving under the app's Google account; Apple storage uses the Apple account/iCloud. Provider/account choice is user-controlled and cannot be independently verified by the app.
- Offer native Android credential saving and browser-compatible password autofill; test save/retrieve during onboarding. Password-manager storage is consented provider synchronization, not plaintext storage in the application backend.
- Support passkey-based vault unlocking only when the selected platform/provider exposes a verified client-side PRF/key-wrapping capability. Normal passkey authentication alone does not decrypt financial records. Store a separate wrapped vault key per supported passkey. Offer a saved-passphrase fallback only if one was enrolled; a passkey-only vault uses the independently tested recovery key or a trusted unlocked device on unsupported-PRF devices, then enrolls a compatible method.
- Version the encryption format and key-derivation parameters.
- Store only wrapped vault keys in the application cloud. Never send plaintext passphrases, recovery keys, private passkey material, PRF outputs, or unwrapped vault keys to the application backend.
- Require a recovery-key confirmation during onboarding.
- Use Android Keystore-backed storage for optional local biometric unlocking.
- Require vault unlocking in each browser session; do not persist decrypted records or unwrapped keys in browser storage.
- Lock after five minutes of inactivity by default; allow a shorter configurable timeout.

A Google password reset must not bypass vault encryption. Recovery requires a working saved vault passphrase, an enrolled encryption-capable passkey, the recovery key, or an already unlocked trusted device. Adding a new passkey after losing all decryption credentials cannot recover an old encrypted vault. Multiple manager credentials must wrap the same vault key and remain attached to the original owner profile.

Use Android Credential Manager and the stable web application's credential origin. Keep passkey enrollment/verification behind a focused credential service with expiring challenges, origin/RP checks, and a maintained WebAuthn implementation; it processes authentication metadata, not financial plaintext. Enroll only after the existing owner is authenticated and the vault is unlocked. Feature-test native/browser/provider support rather than promising universal PRF availability. Apple manager integration is through supported web/browser flows within the Android/web scope; it does not require adding a native iOS application. [Google platform support](https://developers.google.com/identity/passkeys/supported-environments), [Apple passkeys](https://developer.apple.com/passkeys/), [WebAuthn PRF](https://www.w3.org/TR/webauthn-3/#prf-extension)

Present only credential-save routes supported by the current platform and distinguish a successful save, a successful authentication, and a successful vault decryption. An Apple-primary choice does not imply a native Apple provider integration on Android. Test enrollment on a PRF-capable device followed by recovery/unlock on an unsupported device before release.

Changing a passphrase rewraps the vault key. Suspected key compromise requires generating a new key and re-encrypting current data. Previously downloaded copies cannot be remotely erased.

### Local storage and offline behavior

Use Drift/SQLite for persistent encrypted record envelopes and the offline outbox. Build decrypted reporting views in memory while unlocked. Drift supports web storage through browser-compatible SQLite infrastructure. [Drift web documentation](https://drift.simonbinder.eu/platforms/web/)

- Android and trusted browsers can retain encrypted records for offline use.
- An untrusted-browser mode keeps records only for the current session.
- Offline edits save locally and show their synchronization status.
- Locked devices may synchronize already encrypted changes.
- SMS parsing and financial calculations wait until the vault is unlocked.
- Logout clears decrypted state and offers removal of the local encrypted cache.

### Synchronization and conflicts

Do not rely on silent last-write-wins for financial records. Firestore documents that behavior for normal offline synchronization. [Firestore offline documentation](https://firebase.google.com/docs/firestore/manage-data/enable-offline)

Use this application-level protocol:

1. Save an edit and its outbox entry atomically on the device.
2. Assign a unique mutation ID and expected record revision.
3. When online, apply the encrypted mutation through a Firestore transaction.
4. Reject revision mismatches and retain both versions for review.
5. Retry safely using the same mutation ID.
6. Commit linked financial changes together, such as transfers and repayment allocations.

Firestore transactions require connectivity; offline changes therefore remain in the local outbox until they can be checked. [Firestore transaction documentation](https://firebase.google.com/docs/firestore/manage-data/transactions)

Use soft-delete tombstones so deleted records do not reappear from another device. Derived totals are recomputed locally rather than synchronized as competing balances.

### Security boundaries

- Restrict the personal deployment to the configured owner UID.
- Deny unauthenticated access to records and attachments.
- Keep amounts, descriptions, filenames, categories, tax details, and document contents inside encrypted payloads.
- Expose only necessary routing metadata, such as opaque IDs, revisions, sizes, and synchronization timestamps.
- Test authorization rules and reject malformed envelopes.
- Keep financial data out of logs, analytics, crash reports, notifications, and URLs.
- Bundle processing assets with the application; avoid third-party scripts on unlocked financial screens.
- Maintain separate development and production projects.

## 3. Financial model and application behavior

### A. One financial ledger

Replace isolated transaction and loan summaries with a journal containing balanced postings. Keep the bookkeeping mechanics behind ordinary user actions such as “record purchase,” “transfer money,” or “pay EMI.”

Maintain asset, liability, income, expense, and opening-equity accounts internally.

Examples:

| Action | Financial effect |
|---|---|
| Transfer between your bank accounts | Move money between assets |
| Credit-card purchase | Record spending and increase card liability |
| Pay a credit-card bill | Reduce bank balance and card liability |
| Receive a loan | Increase bank balance and loan liability |
| Pay an EMI | Allocate principal, interest, and fees |
| Lend money | Reduce cash/bank balance and increase a receivable |
| Receive repayment | Reduce the receivable; record interest separately |
| Purchase an investment | Move money into an investment asset |
| Receive investment proceeds | Separate returned principal, income, fees, and withholding |

This prevents purchases and subsequent card repayments from becoming duplicate expenses.

Persist money as integer minor units. Use exact integer arithmetic, explicit rounding rules, and decimal-string serialization where necessary for consistent Android/web results. Retain original currency, booked BDT equivalent, and conversion evidence for foreign transactions.

### B. NBR expenditure categories and financial treatment

Keep three concepts separate:

- **Financial action:** expense, income, transfer, debt movement, investment, or asset acquisition.
- **NBR expenditure/asset treatment:** the official report head and applicable form/period version, or an explicit unresolved treatment.
- **Detail category:** optional user-facing detail such as groceries or rent, explicitly mapped beneath official/report treatment.
- **Product eligibility:** whether it counts toward a card benefit or waiver.

Support transaction splits. A single payment may contain several expense categories, principal, interest, taxes, or fees.

Official NBR expenditure heads are the required reporting structure from the financial release, not an optional later mapping from an arbitrary category list. Seed the nine IT-10BB (2023) heads verified from the [official form, page 7](https://nbr.gov.bd/uploads/form/Individual_Return_IT-11Ga%282023%292.pdf); preserve source labels and item references. See the user-journey document for the head table. The verified 2023 form structure is a baseline; confirm the applicable source and period before claiming current-year completeness.

Model an official head catalog, period-aware mappings, expenditure allocations, retained nonfinancial assets, and report snapshots within the financial module. User detail categories must have an explicit official/report mapping, and must not introduce invented official heads. Keep ambiguous fees, gifts, subscriptions, and exceptional items unresolved until their treatment is verified. Never force every debit into lifestyle expenditure.

Electronics and furniture acquisitions need linked asset cost/date/evidence and the relevant asset-report treatment, as the form separately includes furniture and electronic assets. Distinguish acquisitions, repairs/services, and mixed purchases. Report cash outflow, asset additions, and lifestyle consumption as separate views of the same record without counting the acquisition twice. Transfers, principal repayments, refundable deposits, and investments require their own financial treatment.

Provide a financial-release NBR expenditure summary with configurable income-period dates, official form order, allocated totals, comments, source traceability, asset additions, and unresolved items. Pin exports to catalog/mapping versions. A complete summary requires verified applicable mappings and adequate reviewed source coverage; it does not imply a complete tax return or a calculated tax liability.

Capture source deductions through payslips, certificates, and reviewed manual components as well as transaction alerts. Reconcile gross income/profit, evidenced deductions, and a net bank receipt without adding another receipt or an invented bank debit. Store reporting-date rules for purchases, postings, deductions, and cross-period refunds. Expenditure readiness checks include cash/unconnected-source confirmation, source coverage, deduction evidence, and unresolved asset treatment; user completeness confirmations remain explicitly user-attested.

Classification records must include the applied rule, rule version, explanation, confidence, and any user override.

Apply automation in this order:

1. Explicit user correction.
2. Established user rule.
3. Source-specific transaction rules.
4. Merchant/category suggestions.
5. Review queue when ambiguous.

Only deterministic, unambiguous rules auto-apply. Suggestions must not silently become confirmed report treatments. Changes to an official catalog or mapping require a source-backed proposal and approval; historical reports remain reproducible.

### C. Accounts and products

Support:

- Bank and MFS accounts, cash, prepaid balances.
- Debit and credit cards.
- DPS, FDR, and savings certificates.
- Institutional loans and card installment plans.
- Informal borrowing and lending.
- Investment organizations, individual investments, and proof documents.
- Nominees, maturity dates, renewal instructions, and account status.

Balances come from opening entries and posted ledger movements. Imported statement balances provide reconciliation evidence.

### D. Transaction and document intake

Provide manual entry, SMS XML import, Android SMS capture, and PDF statement import.

Include financial email capture and attachment import in the initial financial management release.

Use a common pipeline:

**Import → preserve source → extract observations → validate → identify duplicates → reconcile → classify → post or request review.**

Requirements:

- Preserve the original document/message and its source location.
- Deduplicate identical files by hash.
- Detect overlapping statements and repeated SMS imports.
- Parse XML with external entities disabled.
- Distinguish transaction date, posting date, and statement period.
- Preserve pending, failed, reversed, and completed states.
- Keep parser errors and incomplete extraction visible.
- Support local PDF password entry without storing the password.
- Provide document viewing and manual correction on both platforms.

Start with DBBL and City account statements, the supplied card statement format, and SMS senders found in the export.

Use local PDF text extraction first. Add on-device English OCR for scanned documents, with mandatory review of uncertain fields. Provide manual entry for unreadable or unsupported scripts rather than dropping the document. No cloud OCR or external AI processing is included.

Android SMS capture must be optional. While locked, queue message references and rescan after unlocking. Import only recognized financial messages; do not upload the entire personal inbox. Retain XML import if permissions are denied or unavailable. Google Play treats SMS-based money management as a reviewed permission exception. [Google Play SMS policy](https://support.google.com/googleplay/android-developer/answer/10208820?hl=en)

#### Email intake

- Connect each Gmail mailbox with separate, revocable OAuth consent using `gmail.readonly`. Do not request send, modify, or delete permission. Google sign-in alone does not authorize mailbox reading.
- Let the owner select sender patterns, labels, subjects, and an initial date range. These are application filters: the OAuth permission itself permits mailbox-wide reading and is not restricted to those senders or labels.
- Read transaction alerts, receipts, bills, EMI notices, investment confirmations, fee/waiver notices, and statement attachments. An order confirmation or payment reminder is evidence of an expected event, not proof of a completed payment.
- Fetch and parse message bodies and supported attachments on the unlocked device. Upload only encrypted selected evidence and derived records. Do not place mailbox tokens, raw emails, or attachment plaintext in Firebase services or scheduled jobs.
- Identify messages by mailbox plus provider message ID; retain extraction locations and attachment hashes. A message can describe several transactions or contain a statement, so do not assume one email equals one transaction.
- Link SMS alerts, email alerts, receipts, and statement lines as evidence for one economic event. Check duplicates across sources before creating a ledger entry; preserve disagreements for review.
- Refresh on connection, user request, and periodically while the app is open and unlocked. Use incremental mailbox history with a saved cursor and a bounded rescan when history expires. Android background work is best effort; the web app does not promise ingestion after its tab closes. Catch up at the next unlock.
- Keep OAuth credentials in platform-protected storage; keep web authorization state in memory by default. Handle expired or revoked authorization by requesting reconnection without losing imported records.
- Treat email HTML and attachments as untrusted input: sanitize previews, block remote images and active content, validate size/type, and never execute attachments or follow embedded instructions.
- Provide manual `.eml` and PDF-attachment import on both platforms when mailbox access is unavailable. Gmail access is optional; other input sources remain usable.

Gmail supports listing messages, retrieving their contents, and downloading attachments. `gmail.readonly` is a restricted scope; determine applicable personal-use exceptions and deployment requirements before enabling the connector, and do not promise that verification or assessment requirements are waived. [Gmail message access](https://developers.google.com/workspace/gmail/api/guides/list-messages), [attachment access](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.messages.attachments/get), [scope requirements](https://developers.google.com/workspace/gmail/api/auth/scopes)

### E. Statement reconciliation

Treat statements as authoritative evidence for their covered account and period, while preserving previous values and user classifications.

- Match using account/card, amount, direction, dates, references, and counterparty.
- Auto-match only a unique, unambiguous candidate.
- Send repeated equal amounts and competing candidates for review.
- Create missing transactions from unmatched statement lines after duplicate checks.
- Flag SMS transactions absent from a fully parsed, applicable statement.
- Verify opening balance, movements, closing balance, and overlapping coverage.
- Preserve superseded statements and their history.
- Allow manual match, unmatch, correction, and reprocessing.

A partial parse must never mark an account reconciled.

### F. Loans, lending, investments, and planning

**Loans and EMIs**

- One loan record feeds schedules, payment tracking, and summaries.
- Support reducing-balance interest, flat-rate plans, and fee-based installment products.
- Track scheduled versus actual principal, interest, fees, and outstanding amounts.
- Handle partial payments, early settlement, refinancing, and rounding adjustments.
- Show contractual cost separately from actual payments.

**Lending and borrowing**

- Record each advance and repayment.
- Support unknown due dates and partial repayment.
- Derive outstanding balances instead of relying on a “returned” flag.
- Link receipts and proof documents.

**Investments**

- Separate expected maturity proceeds from realized receipts.
- Track principal, fees, withholding, distributions, losses, and early encashment.
- Link each investment to its funding transactions and evidence.
- Handle multi-year rebate contributions and applicable clawbacks.

**Planning**

- Separate actual, scheduled, and hypothetical amounts.
- Forecast cash after bills, EMIs, savings commitments, and planned purchases.
- Provide purchase scenarios without posting hypothetical transactions.
- Show available cash separately from total assets and unused credit.

### G. Fees, waivers, and rewards

Introduce versioned product policies for:

- Annual card fees, account maintenance, SMS fees, excise duty, late fees, processing fees, and associated taxes.
- Waiver targets based on eligible spending, transaction count, or combined conditions.
- Cashback, points, bonus campaigns, caps, and claim deadlines.

Each policy records its source, effective dates, counting period, exclusions, refund treatment, and whether a claim is required.

Show:

- Eligible amount/count achieved.
- Remaining target and time.
- Expected fee or benefit.
- Qualification status.
- Claim status.
- Actual posted waiver, reversal, or reward.

Do not infer waiver thresholds from statement history. Products without verified terms show “policy not configured.”

### H. Minimum domain interfaces

| Interface/type | Responsibility |
|---|---|
| `JournalEntry` / `Posting` | Balanced financial movements and splits |
| `ImportedObservation` | Extracted facts, confidence, and source reference |
| `ImportParser` | Convert a supported source into observations and issues |
| `ReconciliationService` | Propose matches and report balance discrepancies |
| `ClassificationService` | Suggest financial categories and tax treatments |
| `ExpenditureCatalog` / `ExpenditureAllocation` | Official NBR heads, explicit period-aware mappings, and traceable split allocations |
| `ExpenditureReportService` | Produce NBR expenditure summaries, asset-addition views, and unresolved-item checks within financial management |
| `ProductPolicyEvaluator` | Calculate fee, waiver, and reward progress |
| `TaxEngine` | Calculate a return using explicit inputs and a rule version |
| `ReturnValidator` | Identify missing information and inconsistencies |
| `SyncMutation` | Carry an encrypted change with revision preconditions |
| `BackupService` | Create, verify, and restore encrypted archives |

Use globally unique IDs instead of device-local autoincrement IDs.

## 4. Tax preparation and yearly rule changes

### Explicit periods and rule versions

Model these separately:

- Income-period start and end.
- Assessment-year identifier.
- Original and extended filing deadlines.
- Rule effective dates.
- Return-form version.

Do not infer periods from a year label or assume twelve months. Contributions, income allocation, and projections must use actual dates.

Maintain multiple historical rule sets simultaneously. Every calculation receives an explicit rule version; avoid one global active configuration governing every year.

### Official rule sources

Create a source register containing document title, issuing authority, publication date, effective period, URL, section/page reference, file hash, and review status.

Research the applicable Act, amendments, SROs, annual guidance, forms, and portal instructions before activating a year’s rules. Existing repository examples must be revalidated rather than copied into production.

Use the [official NBR publications](https://nbr.gov.bd/taxtypes/income-tax/income-tax-paripatra/eng/ban) and [individual return form](https://nbr.gov.bd/uploads/form/Individual_Return_IT-11Ga%282023%292.pdf) as starting references.

### Rule-update workflow

1. Detect a changed public source or import an official publication.
2. Create a proposed rule revision.
3. Show changed provisions, sources, and affected calculations.
4. Run boundary tests and compare results with the previous version.
5. Require owner approval before activation.
6. Recalculate drafts and show differences.
7. Preserve prepared and filed snapshots.

Rates, limits, periods, mappings, and supported formula parameters are configurable. A new legal mechanism may require an engine update. Unsupported provisions must block a “ready” result instead of being approximated silently.

### Full return workspace

Cover:

- Taxpayer details and applicable declarations.
- Income schedules and exemptions.
- Investment rebate eligibility and limits.
- Source deductions, advance payments, challans, and applicable adjustments.
- Minimum tax, surcharge, and filing adjustments.
- Lifestyle expenditure.
- Assets, liabilities, opening wealth, gifts, losses, and sources of funds.
- Evidence checklist and portal-entry guidance.

Transactions provide suggestions, but certificates and declarations remain separate evidence. For example, net salary credited to a bank must reconcile with gross salary, deductions, and the employer certificate.

Every reported figure must trace to source transactions, documents, declarations, adjustments, and applied rules.

### Calculation and validation

Implement deterministic, source-backed calculations for each supported period.

Validate:

- Taxable income and exemptions by schedule.
- Correct deduction application without double counting.
- Rebate limits and actual eligible contributions.
- Minimum-tax ordering and applicability.
- Correct surcharge bracket.
- Source-tax claims without duplicate use.
- Asset and liability balances at period end.
- Opening wealth plus funds received minus expenditure/losses against closing wealth.
- Consistent totals across return sections.

Keep tax liability, tax already paid, outstanding payment, and excess payment separate.

### Return lifecycle and exports

Use:

**Draft → Needs review → Ready → Prepared → Filed**

- Only mark a return ready when required validations pass.
- Freeze prepared data, rules, calculations, and evidence references.
- Record filing manually with acknowledgment and submitted return.
- Preserve amendments as new linked versions.
- Keep imported submitted figures separate from reconstructed figures until their status is confirmed.

Produce a human-readable return package, applicable schedules, calculation explanation, evidence index, and portal-field guide. Include a machine-readable archive for the application; do not imply that NBR accepts that archive as an upload.

For unpublished future rules, provide clearly labeled projections. Do not describe them as filing-ready.

## 5. Delivery sequence, migration, and verification

### Implementation phases

| Phase | Deliverable | Exit condition |
|---|---|---|
| 1. Financial specification and foundations | Revised ledger schema, verified NBR expenditure catalog/source register, input-source inventory, dependency list, Flutter scaffold, CI | Android/web builds and documented financial workflows |
| 2. Encrypted vertical slice | Google/Apple manager saving, supported passkey unlock, fallback/recovery, first account and transaction with NBR mapping, offline sync, backup/restore | Same record restored and edited safely across devices; primary manager paths tested |
| 3. Transaction capture and reconciliation | Ledger splits and transfers, manual entry, XML import, Android SMS capture, Gmail connector, email/attachment import, initial PDF parsers, official expenditure/asset allocation, review queue | The same transaction seen in SMS, email, and a statement produces one financial event with linked evidence |
| 4. Financial management | EMI schedules, lending, investments, recurring commitments, budgets, purchase planning | Actual balances and forecasts remain distinct; obligations derive from recorded payments |
| 5. Charges and benefits | Fees, waiver policies, cashback, points, claims, local OCR and input exception handling | Product examples distinguish qualification from posted benefits; partial extraction remains visible |
| 6. Financial management release | NBR expenditure summaries, retained-asset views, recovery exercise, security checks, production setup, private APK and web deployment | Financial acceptance passes on both platforms with expenditure reporting and no tax-liability engine dependency |
| 7. Tax rules and historical reconstruction | Official source register, verified historical rules, income/expense/wealth mapping, calculation trace, validation | Historical reconstruction explains differences from supplied returns without overwriting submitted figures |
| 8. Tax preparation release | Full return workspace, next-return preparation, rule-change proposals, export package | Tax release acceptance passes using verified rules for each supported period |

Approve the concrete dependency manifest before installing application dependencies, as required by the repository rules. Package selection should favor maintained, cross-platform libraries; cryptography must not be implemented from scratch.

### Historical migration

- Preserve original files unchanged.
- Treat identical workbook copies as one source.
- Import workbook records into a review staging area rather than posting monthly summaries as individual transactions.
- Link repeated loan/EMI rows to a single obligation.
- Import projections as forecasts, not realized income.
- Capture declared opening balances with explicit dates and evidence.
- Preserve the workbook and return discrepancies already identified as unresolved review items.
- Keep personal documents and identifiers out of source control and automated test fixtures.
- Use anonymized fixtures for regression tests.

### Verification

**Ledger and financial calculations**

- Transfers, card purchases/payments, refunds, reversals, and pending items.
- Split transactions and partial repayments.
- Zero-interest, fee-based, and early-settled loans.
- Final-installment rounding and investment principal/profit separation.

**Imports and reconciliation**

- Reimported XML, duplicate PDFs, overlapping statements.
- Repeated same-value transactions.
- Password-protected, scanned, malformed, and partially parsed PDFs.
- Opening/closing balance disagreement.
- Statement replacement without losing history.
- Duplicate events across SMS, emails, receipt attachments, and statements.
- Multiple mailboxes, repeated message fetches, several events in one email, and attachments shared across messages.
- Authorization denial/revocation, history-cursor expiry, reconnect, and catch-up after a locked or closed application.
- Order confirmations and reminders do not become completed payments; unsafe email content cannot execute or fetch remote content.

**Tax**

- Financial-release NBR head mappings for rent, food, transport, utilities, education, exceptional expenditure, and interest/principal splits.
- Electronics acquisition versus repairs, mixed receipts, refunds, retained asset costs, and no duplicate lifestyle reporting.
- Expenditure-period changes, catalog gaps, versioned mappings, unresolved amounts, and reproducible report snapshots.

- Official examples and independent expected calculations for every supported year.
- Boundaries around slabs, exemptions, rebate limits, and surcharge bands.
- Changes to period length and filing deadlines.
- Cross-year investments, source-tax claims, and applicable clawbacks.
- Both identified return-total discrepancies.
- Rule updates preserving filed snapshots.

**Security and synchronization**

- Cross-platform encryption test vectors and corrupted-ciphertext rejection.
- Wrong passphrase, recovery-key restoration, and lost-device scenarios.
- Google Password Manager and Apple Passwords save/retrieve as primary; canceled save, wrong manager-account selection, unsupported PRF, credential deletion, new-device unlock, and independent recovery.
- Unauthorized Firebase reads/writes.
- Concurrent edits, retries, interruptions, and offline tombstones.
- Atomic transfer/repayment mutations.
- No plaintext persistence, logs, or uploads.
- Backup restored into a clean installation.

### Release acceptance

**First release: financial management is ready when you can:**

1. Capture transactions through manual entry, SMS, email, and PDF import; review incomplete or ambiguous extraction.
2. Link multiple pieces of evidence to one transaction without double-counting purchases, transfers, or repayments.
3. Manage accounts, cards, loans, EMIs, lending, investments, recurring bills, fees, waivers, and rewards.
4. View reconciled balances, upcoming commitments, and cash-flow forecasts with actual and planned amounts separated.
5. Work offline, resolve synchronization conflicts, and restore an encrypted backup on another device.
6. Categorize spending under verified NBR expenditure heads, track relevant asset acquisitions, and export traceable period summaries with unresolved items clearly identified.
7. Use financial management and expenditure reporting without running tax-liability computation or full return preparation.

**Subsequent release: tax preparation is ready when you can:**

1. Import the supplied financial sources and review unmatched items.
2. See consistent accounts, loans, investments, lending, fees, and forecasts.
3. Trace return figures to their evidence and rule versions.
4. Explain or resolve differences from the previous returns.
5. Prepare the next return when its applicable rules are verified.
6. Work offline and synchronize without silent financial overwrites.
7. Recover the complete encrypted record on a new device.

Run formatting checks, static analysis, unit and integration tests, Firebase rule tests, and Android/web builds in CI. Use separate staging deployment, privacy-safe operational logs, billing alerts, and a tested encrypted backup before production migrations.

Defaults are English UI, BDT reporting, Asia/Dhaka date handling, no external AI processing, and no automatic deletion of original documents or backups.
