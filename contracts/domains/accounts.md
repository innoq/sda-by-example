---
contract-type: domain
id: domain/accounts
version: "1.0.0"
status: active
owner-team: core-banking
owner-contact: core-banking@example-bank.de
last-reviewed: 2025-03-01
domain: accounts
enforcement:
  agent-check: true
  ci-check: false
---

# Domain Contract: Accounts

The Accounts domain is the authoritative source of truth for all financial account state.
It owns account lifecycle, balance management, and the hold mechanism that coordinates
with other domains. Every balance change in the system ultimately flows through this domain.

---

## Ubiquitous Language

| Concept | Definition |
|---------|------------|
| `Account` | A financial account held by one or more `AccountHolder`s, uniquely identified by an `IBAN`. Distinct from "customer" — customer identity and KYC are owned by the Identity domain. |
| `AccountHolder` | A legal entity (natural person or registered organization) with contractual ownership of an `Account`. Joint accounts have multiple `AccountHolder`s with equal legal standing. Not the same as a "user" — a user is a system actor; an AccountHolder is a legal owner. |
| `AccountStatus` | The lifecycle state of an `Account`. Enum: `ACTIVE`, `FROZEN`, `CLOSED`, `PENDING_CLOSURE`. Valid transitions: `ACTIVE → FROZEN` (reversible by authorized operation), `FROZEN → ACTIVE` (reversible), `ACTIVE → PENDING_CLOSURE → CLOSED` (irreversible). `CLOSED` is a terminal state. |
| `Balance` | The current state of an account's funds. Comprises three distinct values that must never be conflated: `BookedBalance` (settled transactions only), `AvailableBalance` (`BookedBalance` minus sum of active `Hold` amounts), and `PendingBalance` (sum of pending, unbooked transactions). |
| `BookedBalance` | The balance reflecting only cleared, settled transactions. Updated by the accounts-service when a `BalanceUpdated` event is processed. This is the legally binding balance for account statements. |
| `AvailableBalance` | The balance a customer may currently spend. Equals `BookedBalance` minus the sum of all active `Hold` amounts. Always less than or equal to `BookedBalance`. |
| `Hold` | A temporary reservation of funds that reduces `AvailableBalance` without changing `BookedBalance`. Created by other domains (typically Transactions) via the accounts API. Released when the underlying operation completes or fails. |
| `IBAN` | International Bank Account Number. The primary, permanent external identifier of an `Account`. Immutable after account creation. |
| `Overdraft` | A pre-approved facility allowing `AvailableBalance` to go negative, up to a defined limit. Distinct from a "loan" — overdraft is a balance facility owned by the Accounts domain; loan principal is owned by the Lending domain. |
| `AccountStatement` | A time-bounded, ordered projection of booked movements on an `Account`. A read-only view; not the canonical event log. Statements are generated from booked transactions, never used to derive current balance. |

---

## Invariants

- **Sole write authority:** `Balance` (all three components) is never written directly by any
  service other than `accounts-service`. Credits, debits, and holds are submitted as requests;
  accounts-service is the single writer of all balance state.

- **IBAN immutability:** An `IBAN` is globally unique and immutable after account creation.
  It cannot be transferred to a different account or reused after closure.

- **CLOSED is terminal:** A `CLOSED` account cannot transition to any other `AccountStatus`.
  No operation — including regulatory reversal — may reopen a closed account.

- **AvailableBalance identity:** `AvailableBalance` always equals `BookedBalance` minus
  the sum of all active `Hold` amounts. This identity must hold at every point in time, including
  under concurrent hold placement and release.

- **FROZEN account hold validity:** Placing a `Hold` on a `FROZEN` account is rejected.
  Existing holds placed before freezing remain valid and may be released.

- **Non-negative AvailableBalance without overdraft:** For accounts without an `Overdraft`
  facility, `AvailableBalance` must never fall below zero. A `Hold` request that would violate
  this is rejected.

---

## Published Interface

### REST API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/accounts/{iban}` | Retrieve account details including status and AccountHolder references |
| `GET` | `/accounts/{iban}/balance` | Retrieve all three balance components (BookedBalance, AvailableBalance, PendingBalance) |
| `GET` | `/accounts/{iban}/statement` | Retrieve account statement; query params: `from` (ISO-8601 date), `to` (ISO-8601 date) |
| `POST` | `/accounts/{iban}/holds` | Place a hold on AvailableBalance; requires `correlationId` and `amount` in request body |
| `DELETE` | `/accounts/{iban}/holds/{holdId}` | Release a specific hold; idempotent |

> All endpoints require authentication. Balance and account detail endpoints require
> OAuth 2.0 with user consent scope. See `contracts/security/data-classification.md`.

### Domain Events

| Event | Trigger | Schema |
|-------|---------|--------|
| `AccountOpened` | A new account has been successfully opened and is ACTIVE | `contracts/events/accounts-events.json` |
| `AccountFrozen` | An account has transitioned from ACTIVE to FROZEN | `contracts/events/accounts-events.json` |
| `AccountUnfrozen` | An account has transitioned from FROZEN back to ACTIVE | `contracts/events/accounts-events.json` |
| `AccountClosureInitiated` | An account has entered PENDING_CLOSURE | `contracts/events/accounts-events.json` |
| `AccountClosed` | An account has reached the terminal CLOSED state | `contracts/events/accounts-events.json` |
| `HoldPlaced` | A hold has been successfully placed on an account | `contracts/events/accounts-events.json` |
| `HoldReleased` | A hold has been released (operation completed or failed) | `contracts/events/accounts-events.json` |
| `BalanceUpdated` | The BookedBalance has changed (after a transaction settles) | `contracts/events/accounts-events.json` |

---

## Negative Scope

| Concern | Owning Domain |
|---------|---------------|
| Payment routing and fund transfer execution | Transactions domain |
| Loan origination, credit decisioning, and repayment scheduling | Lending domain |
| Customer identity, KYC, and AML verification | Identity domain (out of scope for this system) |
| Card issuance, card limits, and PIN management | Card Services domain (out of scope for this system) |
| Fraud detection and real-time risk scoring | Fraud Detection domain (out of scope for this system) |
| Tax withholding and reporting | Compliance/Tax domain (out of scope for this system) |

---

## Consumption Rules

- **No direct database access.** All consumers must access account data exclusively via the
  REST API or by subscribing to domain events. Direct SQL access to the accounts database is
  forbidden without exception. Any exception requires architecture board approval and a contract
  amendment (see `contracts/architecture/service-dependencies.md`).

- **Hold requests require `correlationId`.** Every `POST /accounts/{iban}/holds` request must
  include a `correlationId` in the request body that is traceable to a specific operation in
  the calling service (typically a `transactionId` from the Transactions domain). Holds without
  a valid `correlationId` are rejected.

- **Balance data is classified L3-CONFIDENTIAL.** `Balance`, `BookedBalance`, and
  `AvailableBalance` are classified L3-CONFIDENTIAL per `contracts/security/data-classification.md`.
  Consumers receiving balance data must apply the corresponding access control, storage
  encryption, and logging requirements before storing or re-transmitting this data.

- **Event consumers must be idempotent.** All domain events may be delivered more than once
  (at-least-once delivery guarantee). Consumers must process duplicate events without side
  effects. The `eventId` field in each event envelope serves as the idempotency key.

- **Do not derive balance from events.** Consumers must not attempt to reconstruct or
  maintain a local copy of account balance by processing `BalanceUpdated` events.
  Balance state must always be read from `GET /accounts/{iban}/balance`. Event-derived
  balances will be inconsistent under concurrent operations.
