---
contract-type: domain
id: domain/transactions
version: "1.0.0"
status: active
owner-team: payments
owner-contact: payments@example-bank.de
last-reviewed: 2025-03-01
domain: transactions
regulatory: [PSD2, PSD2-RTS]
enforcement:
  agent-check: true
  ci-check: false
---

# Domain Contract: Transactions

The Transactions domain owns the execution of fund transfers between accounts. It is the
single point of truth for all payment instructions, their lifecycle, and their settlement
status. The domain implements PSD2 Strong Customer Authentication (SCA) requirements and
manages the coordination with the Accounts domain for balance reservation and settlement.

---

## Ubiquitous Language

| Concept | Definition |
|---------|------------|
| `TransferRequest` | A customer's formally submitted instruction to move funds from one account to another. Immutable after submission — modifications require explicit cancellation and resubmission. Distinct from a `Transaction` (the execution artifact). |
| `Transaction` | The execution record created when a `TransferRequest` is accepted and processing begins. A `Transaction` represents what happened; a `TransferRequest` represents what was asked for. One `TransferRequest` results in exactly one `Transaction`. |
| `TransactionStatus` | The lifecycle state of a `Transaction`. Enum: `PENDING` (accepted, not yet processing), `PROCESSING` (hold placed, clearing in progress), `COMPLETED` (settled, BookedBalance updated), `FAILED` (rejected or error during processing), `CANCELLED` (explicit cancellation — only valid from `PENDING`). |
| `TransactionType` | The payment scheme used for settlement. Enum: `SEPA_CREDIT_TRANSFER` (standard SEPA, T+1 settlement), `SEPA_INSTANT` (SEPA Instant, target 10-second settlement), `INTERNAL` (within the same bank, immediate), `DIRECT_DEBIT` (creditor-initiated, requires active Mandate). Each type has distinct SLA, settlement window, and recall rules. |
| `SCAChallenge` | A PSD2 Strong Customer Authentication challenge issued to the payment initiator. Required for transactions that exceed the PSD2 contactless exemption threshold (€30 per transaction or €100 cumulative), all remote electronic payments above €30, and all first-time payments to a new beneficiary. |
| `SettlementWindow` | The daily cut-off window for batch submission to the clearing house. SEPA Credit Transfer: 16:00 CET (outgoing), 13:00 CET (incoming). Transactions submitted after cut-off are settled the following business day. |
| `Mandate` | A signed authorization by a debtor permitting a specific creditor to initiate Direct Debit transactions. Mandates are valid until explicitly revoked. Distinct from a `TransferRequest` — a Mandate is a persistent authorization, not a single payment instruction. |
| `RejectionReason` | A structured rejection code when a `Transaction` enters `FAILED` status. Values follow ISO 20022 reason codes (e.g., `AC01` invalid account, `AM04` insufficient funds, `FF01` invalid file format). Reason codes are stable and machine-parseable. |
| `TransactionFee` | The fee charged to the initiating account at transaction completion. Deducted from `BookedBalance` as a separate ledger entry, not as part of the transfer amount. |
| `RecallRequest` | A post-settlement request to reverse a completed SEPA Credit Transfer. Distinct from cancellation (pre-settlement). Subject to creditor acceptance; not guaranteed. |

---

## Invariants

- **TransferRequest immutability:** A `TransferRequest` is immutable after submission.
  Amount, accounts, and beneficiary details cannot be modified. Any change requires explicit
  cancellation (`PENDING` status only) and submission of a new `TransferRequest`.

- **Double-entry accounting identity:** For every `Transaction` that reaches `COMPLETED`,
  the total amount debited from the source account equals the total amount credited to the
  destination account plus any applicable `TransactionFee`. The net sum is always zero.

- **SCA precedes PROCESSING:** A `Transaction` requiring SCA (per PSD2 thresholds) must
  not transition from `PENDING` to `PROCESSING` until the associated `SCAChallenge` is
  successfully completed. No exceptions for internal or test transactions in production.

- **Hold precedes debit:** Before transitioning to `PROCESSING`, the transactions-service
  must place a `Hold` on the source account via the Accounts domain API. The transition to
  `PROCESSING` is contingent on hold placement succeeding. If the Accounts domain rejects
  the hold, the `Transaction` transitions to `FAILED` with reason code `AM04`.

- **CANCELLED is only valid from PENDING:** A `Transaction` can only be cancelled while
  in `PENDING` status. Once `PROCESSING` begins, cancellation is not possible — only a
  `RecallRequest` (post-settlement) may be initiated for completed transactions.

- **Transaction states are monotonic:** A `Transaction` never moves backward in its
  lifecycle. A `COMPLETED` transaction cannot become `PROCESSING`; a `FAILED` transaction
  cannot become `COMPLETED`. State transitions are irreversible.

---

## Published Interface

### REST API

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/transfers` | Submit a new TransferRequest; returns a transactionId. Requires OAuth 2.0 with `payments:write` scope (PSD2). |
| `GET` | `/transfers/{transactionId}` | Retrieve Transaction status and details |
| `DELETE` | `/transfers/{transactionId}` | Cancel a Transaction; only valid in PENDING status; idempotent |
| `GET` | `/transfers/{transactionId}/sca` | Retrieve current SCAChallenge for a pending transaction |
| `POST` | `/transfers/{transactionId}/sca` | Submit SCA response to complete the challenge |
| `GET` | `/mandates/{mandateId}` | Retrieve a Direct Debit mandate |
| `DELETE` | `/mandates/{mandateId}` | Revoke a Direct Debit mandate |

> `POST /transfers` requires PSD2-compliant SCA for transactions above threshold.
> Consumers must implement the SCA redirect/challenge flow.

### Domain Events

| Event | Trigger | Schema |
|-------|---------|--------|
| `TransactionInitiated` | A TransferRequest was accepted and a Transaction created | `contracts/events/transactions-events.json` |
| `TransactionCompleted` | A Transaction settled successfully (PROCESSING → COMPLETED) | `contracts/events/transactions-events.json` |
| `TransactionFailed` | A Transaction failed (any status → FAILED) | `contracts/events/transactions-events.json` |
| `TransactionCancelled` | A Transaction was explicitly cancelled (PENDING → CANCELLED) | `contracts/events/transactions-events.json` |
| `MandateCreated` | A new Direct Debit mandate was registered | `contracts/events/transactions-events.json` |
| `MandateRevoked` | A Direct Debit mandate was revoked | `contracts/events/transactions-events.json` |

---

## Negative Scope

| Concern | Owning Domain |
|---------|---------------|
| Account balance state and hold management | Accounts domain |
| Fraud detection and real-time risk scoring | Fraud Detection domain (out of scope for this system) |
| Loan disbursement scheduling | Lending domain |
| Foreign exchange rate management | FX domain (out of scope for this system) |
| Card payment processing (POS, contactless) | Card Services domain (out of scope for this system) |
| Regulatory reporting submission (SWIFT, TARGET2) | Compliance domain (out of scope for this system) |

---

## Consumption Rules

- **OAuth 2.0 with `payments:write` scope is mandatory for payment initiation.**
  `POST /transfers` requires a valid OAuth 2.0 access token with the `payments:write` scope.
  This is a PSD2 regulatory requirement and cannot be bypassed for any consumer, internal or external.

- **Implement the SCA challenge flow.** Consumers initiating transactions above PSD2 thresholds
  must implement the SCA challenge/response flow. Polling `GET /transfers/{id}/sca` and
  completing `POST /transfers/{id}/sca` is part of the required integration contract.

- **Tolerate late `TransactionCompleted` events.** SEPA Credit Transfer events may arrive up
  to the next business day after initiation (T+1 settlement). Consumers must not assume that
  `TransactionCompleted` follows `TransactionInitiated` within a predictable short window.
  Design consumers to handle eventual consistency on settlement.

- **Event consumers must be idempotent.** All domain events may be delivered more than once.
  The `eventId` field in the event envelope is the idempotency key. Consumers must deduplicate
  on `eventId` before applying any state change.

- **Do not use `transactionId` as a financial reference.** The `transactionId` is a system
  identifier for the Transaction record, not a legally binding payment reference. The
  `endToEndId` field (ISO 20022) is the correct reference for external reconciliation.

- **No direct database access.** All consumers must use the REST API or subscribed events.
  The transactions database schema is an internal implementation detail and may change without
  notice.
