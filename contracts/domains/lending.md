---
contract-type: domain
id: domain/lending
version: "1.0.0"
status: active
owner-team: credit
owner-contact: credit@example-bank.de
last-reviewed: 2025-03-01
domain: lending
regulatory: [BAFIN-MaRisk, EU-CRD-IV, GDPR]
enforcement:
  agent-check: true
  ci-check: false
---

# Domain Contract: Lending

The Lending domain owns the full lifecycle of credit products: from application and credit
decisioning through loan agreement, disbursement, repayment, and closure. It operates under
strict regulatory obligations (BAFIN-MaRisk, EU CRD IV) that govern how credit decisions are
made, documented, and audited. It does not own fund movement — disbursements and repayments
are executed via the Transactions domain.

---

## Ubiquitous Language

| Concept | Definition |
|---------|------------|
| `LoanApplication` | A customer's formal request for credit. Contains the requested amount, term, purpose, and customer consent for credit assessment. Immutable after submission. Distinct from a `LoanOffer` (the bank's response) and a `LoanAgreement` (the signed contract). |
| `LoanOffer` | The bank's conditional offer of credit in response to a `LoanApplication`. Contains final terms: amount, interest rate, fee schedule, repayment term. Valid for a limited acceptance window. Expires if not accepted. |
| `LoanAgreement` | A fully executed credit contract between the bank and the borrower, created when a `LoanOffer` is accepted. Immutable after signing. The authoritative source of all credit terms. Any change to terms creates a `LoanAmendment`, never modifies the original `LoanAgreement`. |
| `LoanAmendment` | A formal modification to a `LoanAgreement` (e.g., term extension, rate change for variable-rate loans). A `LoanAmendment` supplements — it does not replace — the original `LoanAgreement`. Both are required for complete credit terms. |
| `CreditDecision` | The outcome of the credit assessment process for a `LoanApplication`. Enum: `APPROVED`, `CONDITIONALLY_APPROVED`, `REJECTED`. Every `CreditDecision` must include a structured `reason` field — this is a hard regulatory requirement under MaRisk BA 7. |
| `CreditScore` | The internal risk assessment score derived by the lending domain's credit models. This is not a third-party bureau score (Schufa etc.) — bureau data is one input into the model. The `CreditScore` is proprietary and may never be transmitted to external parties or exposed in events. |
| `LoanStatus` | The lifecycle state of a `LoanAgreement`. Enum: `APPLICATION` (application received), `UNDERWRITING` (credit assessment in progress), `APPROVED` (offer made, awaiting acceptance), `ACTIVE` (agreement signed, disbursement made), `IN_ARREARS` (one or more installments overdue), `DEFAULTED` (90+ days in arrears, regulatory classification), `CLOSED` (fully repaid or written off). |
| `Disbursement` | The transfer of loan principal from the bank to the borrower's account. Executed via the Transactions domain. A `Disbursement` may only be initiated after the `LoanAgreement` reaches `ACTIVE` status. |
| `RepaymentSchedule` | The projected installment plan derived from the `LoanAgreement`. Lists each `Installment` with its due date, principal component, and interest component. Recalculated on `EarlyRepayment` or rate adjustment for variable-rate loans. |
| `Installment` | A single scheduled repayment consisting of a principal component and an interest component. The split between principal and interest follows the amortization method defined in the `LoanAgreement` (annuity, straight-line, bullet, etc.). |
| `EarlyRepayment` | A full or partial repayment ahead of the `RepaymentSchedule`. Subject to a prepayment fee if defined in the `LoanAgreement`. Triggers recalculation of the remaining `RepaymentSchedule`. |
| `Arrears` | A state where one or more `Installment`s are overdue. `IN_ARREARS` triggers specific regulatory reporting obligations and customer communication requirements under MaRisk. |

---

## Invariants

- **LoanAgreement immutability:** A `LoanAgreement` is immutable after the customer's digital
  signature is recorded. Rate adjustments, term extensions, and fee changes are represented as
  `LoanAmendment` records that reference the original agreement — never as modifications.

- **CreditDecision requires auditable reason:** Every `CreditDecision` must record a structured
  `reason` field at the time of decision. This is a hard regulatory requirement under
  BAFIN-MaRisk BA 7. A `CreditDecision` without a `reason` is invalid and must never be
  persisted or emitted as an event.

- **Disbursement requires ACTIVE LoanAgreement:** A `Disbursement` may only be initiated via
  the Transactions domain after the associated `LoanAgreement` has reached `ACTIVE` status.
  Initiating a `Disbursement` before `ACTIVE` is a process violation.

- **CreditScore is never externally exposed:** The internal `CreditScore` must never appear
  in API responses, events, or log output. It is an internal model artifact. Consumers receive
  `CreditDecision` (outcome + reason); the score that led to it is not part of the Published
  Interface.

- **DEFAULTED requires regulatory approval to reverse:** A `LoanAgreement` in `DEFAULTED`
  status cannot transition back to `ACTIVE` without an explicit regulatory approval workflow.
  Technical implementations must enforce this; no operational override is permitted.

- **Outstanding balance identity:** The outstanding principal balance at any point is:
  `original_disbursed_amount − sum(principal_component of all COMPLETED Installments) − sum(EarlyRepayment principal amounts)`.
  This identity must hold and is auditable at any point in the loan's lifecycle.

---

## Published Interface

### REST API

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/loan-applications` | Submit a new LoanApplication |
| `GET` | `/loan-applications/{applicationId}` | Retrieve application status and offer details |
| `POST` | `/loan-applications/{applicationId}/accept-offer` | Accept a LoanOffer; creates a LoanAgreement |
| `POST` | `/loan-applications/{applicationId}/human-review` | Request human review of a REJECTED automated decision (GDPR Art. 22 right) |
| `GET` | `/loans/{loanId}` | Retrieve LoanAgreement details and current LoanStatus |
| `GET` | `/loans/{loanId}/repayment-schedule` | Retrieve current RepaymentSchedule |
| `POST` | `/loans/{loanId}/early-repayment` | Initiate an EarlyRepayment |

> The `POST /loan-applications/{id}/human-review` endpoint is a GDPR Art. 22 requirement.
> It must remain available as long as the CreditDecision is challengeable.

### Domain Events

| Event | Trigger | Schema |
|-------|---------|--------|
| `LoanApplicationSubmitted` | A new LoanApplication was received | `contracts/events/lending-events.json` |
| `CreditDecisionMade` | A CreditDecision was reached (any outcome) | `contracts/events/lending-events.json` |
| `LoanAgreementSigned` | A LoanOffer was accepted and LoanAgreement created | `contracts/events/lending-events.json` |
| `DisbursementInitiated` | A Disbursement was initiated via the Transactions domain | `contracts/events/lending-events.json` |
| `InstallmentDue` | An Installment's due date has arrived | `contracts/events/lending-events.json` |
| `InstallmentPaid` | An Installment was fully or partially paid | `contracts/events/lending-events.json` |
| `LoanEnteredArrears` | A LoanAgreement transitioned to IN_ARREARS | `contracts/events/lending-events.json` |
| `LoanDefaulted` | A LoanAgreement transitioned to DEFAULTED (90+ days in arrears) | `contracts/events/lending-events.json` |
| `LoanClosed` | A LoanAgreement reached terminal CLOSED status | `contracts/events/lending-events.json` |

---

## Negative Scope

| Concern | Owning Domain |
|---------|---------------|
| Account balance management and payment execution | Accounts and Transactions domains |
| Credit bureau queries (Schufa, CRIF) | Identity/KYC domain (out of scope for this system) |
| Regulatory reporting submissions (e.g., Bundesbank granular credit data) | Compliance domain (out of scope for this system) |
| Debt collection and recovery process | Collections domain (out of scope for this system) |
| Mortgage and real estate collateral management | Mortgage domain (out of scope for this system) |
| Customer-facing communication (letters, emails) | Notification domain |

---

## Consumption Rules

- **CreditDecision events contain no CreditScore.** The `CreditDecisionMade` event includes
  the decision outcome and structured reason but never the raw or derived `CreditScore`.
  Consumers must not attempt to reconstruct the score from decision history. The score
  is an internal model output, not a publishable asset.

- **GDPR Art. 22 must be technically implementable.** Any consumer or integration that
  surfaces lending outcomes to customers must ensure the `POST /loan-applications/{id}/human-review`
  endpoint is accessible. Automated credit decisions are subject to the GDPR right to
  challenge, and this right must not be technically blocked.

- **Disbursement events must be reconciled with Transactions events.** Consumers tracking
  loan disbursements must correlate `DisbursementInitiated` events with `TransactionCompleted`
  events from the Transactions domain to confirm actual fund movement. `DisbursementInitiated`
  does not guarantee settlement; it means the process was started.

- **No direct database access.** All consumers must use the REST API or subscribed events.
  The lending database — including credit decision history and score model data — is an
  internal implementation detail subject to change.

- **Credit data is classified L3-CONFIDENTIAL (application data) and L4-RESTRICTED (scores).**
  See `contracts/security/data-classification.md`. Consumers receiving credit data must apply
  the corresponding security controls. Loan application data (L3) requires OAuth + consent scope.

- **Retain CreditDecision reasons as received.** Consumers who store `CreditDecisionMade`
  events for audit purposes must store the `reason` field without modification. This field
  is part of the BAFIN-MaRisk audit trail and must not be summarized or truncated.
