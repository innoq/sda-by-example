---
template-type: domain
template-version: "1.0.0"
description: >
  Template for Domain Contracts in Spec-Driven Architecture. A Domain Contract
  defines the fachliche Semantik of a domain as seen from the outside: its
  Ubiquitous Language, invariants, published interface, and consumption rules.
  It is the authoritative source of truth for what a domain owns and guarantees.
contract-frontmatter-required:
  - contract-type
  - id
  - version
  - status
  - owner-team
  - owner-contact
  - last-reviewed
  - domain
  - enforcement.agent-check
contract-frontmatter-optional:
  - regulatory
sections:
  - ubiquitous-language
  - invariants
  - published-interface
  - negative-scope
  - consumption-rules
---

# Contract Template: Domain

This template defines how to build and interpret a Domain Contract.
It is consumed by the `sda-contract-builder` skill (to guide contract creation)
and by the `sda-contract-gatekeeper` skill (to interpret and validate contracts).

---

## Section: Ubiquitous Language

**Purpose:**
Defines the precise, canonical meaning of every core concept this domain exposes.
Ubiquitous Language is the primary tool for eliminating ambiguity at domain boundaries.
When two teams use the same word for different things — or different words for the same thing —
the Ubiquitous Language section is what resolves the conflict. Without it, agents and developers
make assumptions that silently violate domain boundaries.

**Builder — Questions:**
1. What are the 3–7 most important concepts that consumers of this domain need to understand?
   (List concept names first, then define each.)
2. For each concept: provide a precise definition in 1–2 sentences.
   The definition should answer: "What exactly is this, and what is it NOT?"
3. For concepts that are easily confused with similar terms in adjacent domains:
   add an explicit disambiguation note.
4. Are there important state enumerations (e.g., `AccountStatus`)? If so, list all values
   and their valid transition paths.

**Gatekeeper — Interpretation:**
- When reviewing an implementation, check whether it uses terminology from this section.
  If an implementation uses a term differently from its definition here, flag as **YELLOW**.
- If an implementation introduces a concept that is clearly within this domain's scope
  but not listed here, flag as **YELLOW**: "Unnamed concept — consider contract update."
- If an implementation uses a synonym for a defined concept (e.g., "user" instead of
  "AccountHolder"), flag as **YELLOW**: "Ubiquitous Language violation — use canonical term."
- State machine violations (transitions not listed in the enum) are **RED**.

**Constraints:**
- Minimum 3 concepts, recommended 5–8.
- Each concept must have a definition.
- Disambiguations are required where the concept name could be confused with a concept
  in an adjacent domain.

**Example:**

```markdown
| Concept | Definition |
|---------|------------|
| `Account` | A financial account held by one or more AccountHolders, identified by IBAN. Distinct from "customer" — customer identity is owned by the Identity domain. |
| `AccountStatus` | Enum: `ACTIVE`, `FROZEN`, `CLOSED`, `PENDING_CLOSURE`. Valid transitions: ACTIVE → FROZEN (reversible); ACTIVE → PENDING_CLOSURE → CLOSED (irreversible). |
| `Balance` | The current available balance. Distinct from `BookedBalance` (cleared transactions only) and `AvailableBalance` (BookedBalance minus active holds). |
```

---

## Section: Invariants

**Purpose:**
Invariants are rules that are always true for this domain, regardless of who calls the API
or which event is received. They represent the domain's non-negotiable guarantees.
Invariants are what the domain promises to uphold — and what every consumer can rely on.
An invariant violation is a bug in the domain, not in the consumer.

**Builder — Questions:**
1. What rules are always true for the core entities of this domain?
   (Think: what would be a data corruption scenario if violated?)
2. Who is the sole authority for writing/mutating each core entity?
   (Write authority is almost always an invariant.)
3. Are there state transition rules (beyond the enum list in Ubiquitous Language)?
   List them explicitly.
4. Are there mathematical or accounting identities that must always hold?
   (e.g., "sum of X always equals Y")
5. What makes an entity permanently terminal? (e.g., "a CLOSED account cannot be reopened")

**Gatekeeper — Interpretation:**
- **Any invariant violation is RED.** There are no exceptions.
- Pay particular attention to "sole writer" invariants: if an implementation proposes
  to write to an entity that another domain owns, this is RED.
- Mathematical/accounting identities should be checked if the implementation involves
  balance or aggregation logic.
- Terminal state violations (e.g., mutating a CLOSED entity) are RED.

**Constraints:**
- Minimum 3 invariants.
- Each invariant must be stated as a falsifiable rule (not a vague guideline).
- Write-authority invariants are mandatory for all core entities.

**Example:**

```markdown
- `Balance` is never written directly by any service other than `accounts-service`.
  Credits, debits, and holds are submitted as requests; accounts-service is the sole writer.
- A `CLOSED` account cannot transition to any other status.
- `AvailableBalance` always equals `BookedBalance` minus the sum of all active `Hold` amounts.
- An `IBAN` is globally unique and immutable after account creation.
```

---

## Section: Published Interface

**Purpose:**
Defines exactly what this domain exposes to consumers: REST endpoints, events, and any
other integration points. This section is the domain's contract with the outside world.
Consumers may only use what is listed here — anything not listed is an implementation detail
that may change without notice.

**Builder — Questions:**
1. What REST API endpoints does this domain expose to other services?
   (List method, path, and one-line description. Do not include internal endpoints.)
2. What domain events does this domain emit?
   (List event names. Reference a schema file if one exists or will exist.)
3. Are there any other integration points (gRPC services, GraphQL, Kafka topics)?
4. For events: which are "fact" events (something happened) vs. "request" events
   (asking another domain to do something)?

**Gatekeeper — Interpretation:**
- If a consumer calls an endpoint not listed here, flag as **RED**: "Undocumented endpoint —
  not part of the Published Interface."
- If a consumer subscribes to an event not listed here, flag as **YELLOW**: "Event not in
  Published Interface — may be an implementation detail, not a stable contract."
- If an implementation proposes adding a new endpoint or event that fits this domain's scope,
  flag as **YELLOW**: "New integration point — requires contract update before implementation."

**Constraints:**
- At minimum, list the primary CRUD-style endpoints for core entities.
- Event names should follow the pattern `<Aggregate><PastTense>` (e.g., `AccountOpened`).
- Schema references should point to a file path (even if the file is planned, not yet written).

**Example:**

```markdown
### REST API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/accounts/{iban}` | Retrieve account details by IBAN |
| `GET` | `/accounts/{iban}/balance` | Retrieve current balance (all three balance types) |
| `POST` | `/accounts/{iban}/holds` | Place a hold on available balance |
| `DELETE` | `/accounts/{iban}/holds/{holdId}` | Release a hold |

### Domain Events

| Event | Trigger |
|-------|---------|
| `AccountOpened` | A new account has been successfully opened |
| `AccountFrozen` | An account has been frozen (ACTIVE → FROZEN) |
| `HoldPlaced` | A hold has been placed on an account's balance |
| `BalanceUpdated` | The booked balance has changed (after a transaction settles) |

Event schema: `contracts/events/accounts-events.json`
```

---

## Section: Negative Scope

**Purpose:**
Explicitly states what this domain does NOT own. Negative scope is as important as positive
scope: it prevents scope creep, clarifies responsibilities at domain boundaries, and gives
consumers a clear map of where to go for what. Without negative scope, ambiguous responsibilities
accumulate silently — until two teams both think they own the same thing, or neither does.

**Builder — Questions:**
1. What concerns might a naive observer assume belong to this domain, but actually don't?
   (Think about what consumers frequently ask you to do that you redirect elsewhere.)
2. For each concern you list: which domain does own it?
   (Use the canonical domain name, with a pointer if available.)
3. Are there data assets that sound like they belong here but don't?
   (e.g., "customer name" might sound like it belongs to an accounts domain,
   but could be owned by an identity domain.)

**Gatekeeper — Interpretation:**
- If an implementation assigns a listed responsibility to this domain, flag as **RED**:
  "Scope violation — [concern] is owned by [other domain]."
- If an implementation creates a coupling to a concern listed here as negative scope,
  flag as **YELLOW**: "Indirect scope boundary violation — review ownership."

**Constraints:**
- Minimum 3 entries.
- Each entry must name the owning domain (even if approximate).

**Example:**

```markdown
| Concern | Owning Domain |
|---------|---------------|
| Payment routing and transfer execution | Transactions domain |
| Loan origination and repayment scheduling | Lending domain |
| Customer identity verification (KYC/AML) | Identity domain (out of scope for this system) |
| Card issuance and PIN management | Card Services domain (out of scope for this system) |
```

---

## Section: Consumption Rules

**Purpose:**
Defines the rules that every consumer of this domain must follow. These are not optional
guidelines — they are preconditions for correct consumption. Violations here typically cause
data integrity issues, security problems, or regulatory non-compliance. Consumption Rules
are the domain's requirements for its consumers, as opposed to the domain's internal invariants.

**Builder — Questions:**
1. Are there prohibited access patterns? (Most commonly: direct database access.)
2. Are there required fields or headers on API calls?
   (e.g., "every Hold request must include a correlationId")
3. Are there idempotency requirements on event consumers?
4. Are there data classification constraints on data received from this domain?
   (Reference the security contract if applicable.)
5. Are there SLA or ordering guarantees consumers must not assume?
   (e.g., "events may arrive out of order")

**Gatekeeper — Interpretation:**
- Direct database access rules are always **RED** if violated.
- Missing required fields (like `correlationId`) are **RED**.
- Idempotency requirement violations are **RED** (data duplication risk).
- Data classification downstream rules are **YELLOW** to **RED** depending on the
  classification level involved (cross-reference security contract).
- SLA assumption violations are **YELLOW**: flag for review with consumer team.

**Constraints:**
- Must include at least one rule about data access (API-only vs. direct DB).
- Should reference the security contract for any data classified as L3 or L4.

**Example:**

```markdown
- **No direct database access.** All consumers must use the REST API or subscribed events.
  Direct SQL access to the accounts database is forbidden without exception.
- **Hold requests require `correlationId`.** Every `POST /accounts/{iban}/holds` request
  must include a `correlationId` traceable to a transaction in the Transactions domain.
- **Balance data is classified L3-CONFIDENTIAL** per `contracts/security/data-classification.md`.
  Consumers must apply data classification rules before storing or re-transmitting balance data.
- **Event consumers must be idempotent.** Domain events may be delivered more than once.
  Consumers must handle duplicate delivery without side effects.
```
