---
contract-type: architecture
id: architecture/service-dependencies
version: "1.0.0"
status: active
owner-team: platform-architecture
owner-contact: architecture@example-bank.de
last-reviewed: 2025-03-01
services:
  - api-gateway
  - accounts-service
  - transactions-service
  - lending-service
  - notification-service
enforcement:
  agent-check: true
  ci-check: false
  tool-hint: >
    Deptrac configuration: /tools/deptrac.yaml.
    ArchUnit rules: /arch-tests/src/main/java/ArchitectureRules.java.
    Run with: ./gradlew architectureTest
---

# Architecture Contract: Service Dependencies

This contract defines the allowed and forbidden structural relationships between all services
in the banking platform. It governs which services may call which, using which communication
mechanism, and at which level of API specificity. Deviations from this contract require the
exception process defined in the final section.

---

## Allowed Dependencies

The following service-to-service dependencies are permitted. Every dependency not listed
here is implicitly forbidden. Specificity matters: where the scope column lists specific
endpoints, the dependency is only permitted for those endpoints.

| Caller | Callee | Mechanism | Scope |
|--------|--------|-----------|-------|
| `api-gateway` | `accounts-service` | REST (sync) | Full Published Interface (`contracts/domains/accounts.md`) |
| `api-gateway` | `transactions-service` | REST (sync) | Full Published Interface (`contracts/domains/transactions.md`) |
| `api-gateway` | `lending-service` | REST (sync) | Full Published Interface (`contracts/domains/lending.md`) |
| `transactions-service` | `accounts-service` | REST (sync) | Hold endpoints only: `POST /accounts/{iban}/holds`, `DELETE /accounts/{iban}/holds/{holdId}` |
| `lending-service` | `transactions-service` | REST (sync) | Disbursement initiation only: `POST /transfers` |
| `notification-service` | event bus | Async consume | `AccountOpened`, `AccountClosed`, `TransactionCompleted`, `TransactionFailed`, `LoanAgreementSigned`, `LoanEnteredArrears`, `LoanDefaulted` |

**Rationale for scope restrictions:**

- `transactions-service → accounts-service` is restricted to hold endpoints because the
  Transactions domain has no legitimate reason to read account details or statements.
  It only needs to reserve funds. Limiting the scope prevents accidental coupling to
  account data that the Accounts domain may change.

- `lending-service → transactions-service` is restricted to `POST /transfers` because the
  Lending domain's only interaction with payments is disbursement. Giving it access to the
  full Transactions API would allow it to initiate arbitrary transfers — which is not its
  responsibility.

---

## Forbidden Dependencies

The following dependencies are explicitly prohibited. They represent the most common
architectural anti-patterns for this system. Any match is a **RED** violation with no
exceptions outside the documented exception process.

| Forbidden | Reason |
|-----------|--------|
| `lending-service` → `accounts-service` (direct) | Lending must not call Accounts directly for fund operations. All fund movements go through `transactions-service` to maintain the double-entry accounting guarantee. |
| `lending-service` → `accounts-db` | Cross-service database access violates service autonomy and bypasses the Accounts domain's write authority invariant. |
| `transactions-service` → `lending-service` | Reversed dependency direction. The Payments domain must not know about or depend on the Credit domain. |
| `accounts-service` → `transactions-service` | Reversed dependency direction. The Accounts domain must not initiate transactions; it only responds to hold requests. |
| `accounts-service` → `lending-service` | Reversed dependency direction. The Accounts domain must not know about credit products. |
| `notification-service` → `accounts-service` | `notification-service` is event-consumer only. It must never call domain service APIs. |
| `notification-service` → `transactions-service` | Same as above. |
| `notification-service` → `lending-service` | Same as above. |
| any service → any other service's database | General prohibition. All cross-service data access must go through the owning service's Published Interface (REST API or events). No exceptions without formal architecture board approval. |
| any service → `api-gateway` | The API gateway is an ingress-only component. No service may call the gateway. |

---

## Allowed Communication Patterns

### Synchronous

| Pattern | Scope | Standard Required |
|---------|-------|------------------|
| REST/HTTP (JSON) | All service-to-service and external API calls | OpenAPI 3.x specification; must be committed to `/api-specs/` directory |
| gRPC | Internal service-to-service only (not exposed at API gateway) | Protobuf 3; schema in `/proto/` directory; not yet in use, requires architecture board approval to adopt |

### Asynchronous

| Pattern | Scope | Standard Required |
|---------|-------|------------------|
| Domain events via message bus | All async domain event communication | CloudEvents 1.0 envelope format; Avro message schema; schema registered in schema registry before first publish |

### Forbidden Patterns

| Pattern | Reason |
|---------|--------|
| GraphQL (any variant) | Not adopted; requires architecture board approval before any introduction |
| Synchronous calls for notifications | Notification triggers must use async events to decouple notification-service |
| Shared database tables across services | Cross-service state sharing via database violates service autonomy |
| In-process method calls across domain boundaries | Domain services must not be deployed in the same JVM/process with shared in-memory calls across domain boundaries |

---

## Layer Boundaries

All services follow a **Ports & Adapters (Hexagonal) architecture**. The dependency rule
is strict: adapters may depend on the application layer; the application layer may depend
on the domain layer; the domain layer has no outbound dependencies.

| Layer | Responsibility | May Depend On |
|-------|---------------|----------------|
| Inbound Adapters | REST controllers, event consumers, CLI | Application layer interfaces only |
| Outbound Adapters | Repository implementations, HTTP clients, event publishers | Domain model and application layer ports |
| Application Layer | Use cases, application services, orchestration | Domain layer only |
| Domain Layer | Entities, value objects, aggregates, domain services | Nothing — pure domain model, no framework imports |

**Cross-cutting concerns** (logging, tracing, metrics):

- Logging and tracing instrumentation belongs in the adapter layer (inbound and outbound).
- Domain objects must not import logging frameworks, tracing libraries, or metrics clients.
  Domain events are created in the domain layer and logged/published in the adapter layer.
- Transaction management (database transactions) belongs in the application layer, not the domain.

**Package conventions** (for JVM services):

```
com.examplebank.<service>.
  adapters.inbound.rest        ← REST controllers
  adapters.inbound.events      ← Event consumers
  adapters.outbound.persistence ← Repository implementations
  adapters.outbound.http       ← HTTP clients to other services
  application                  ← Use cases, application services
  domain                       ← Entities, value objects, domain services
```

---

## Exception Process

Any deviation from this Architecture Contract — whether a new dependency, a new communication
pattern, or a layer boundary violation — requires the following process before implementation:

1. **Draft an Architecture Decision Record (ADR).** The ADR must document:
   - The specific rule being deviated from (reference section and row in the relevant table)
   - The technical justification for why the deviation is necessary
   - The alternatives considered and why they were rejected
   - The proposed mitigation to limit architectural damage
   - An exit plan: how and when the deviation will be resolved (not "eventually")

2. **Architecture Board review.** Submit the ADR as a pull request to `/adr/` in this repository.
   Tag the pull request with `architecture-exception`. Expected review time: 5 business days.
   The architecture board consists of: Lead Architects of each domain team + Platform Architecture.

3. **Contract amendment.** If approved, the exception is added inline to the relevant section
   of this contract as a versioned footnote:
   ```
   > **Exception [ADR-NNN]:** [brief description] — approved [date], expires [date or condition].
   ```
   The contract version is incremented (minor version bump).

4. **Time-bound.** All exceptions are granted for a maximum of two release cycles (approximately
   8 weeks). After expiry, the exception is either resolved, extended (new ADR required), or
   made permanent (contract amendment without expiry, full board approval required).

5. **The Gatekeeper will continue to flag the deviation as YELLOW** after approval, with the
   ADR reference noted. This ensures the deviation remains visible in the audit trail.
