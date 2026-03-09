---
template-type: architecture
template-version: "1.0.0"
description: >
  Template for Architecture Contracts in Spec-Driven Architecture. An Architecture
  Contract defines the allowed and forbidden structural relationships between services:
  dependency directions, communication patterns, and layer boundaries. It is the
  authoritative source for what the system's topology looks like — and what it must
  never become.
contract-frontmatter-required:
  - contract-type
  - id
  - version
  - status
  - owner-team
  - owner-contact
  - last-reviewed
  - services
  - enforcement.agent-check
contract-frontmatter-optional:
  - enforcement.tool-hint
sections:
  - allowed-dependencies
  - forbidden-dependencies
  - allowed-communication-patterns
  - layer-boundaries
  - exception-process
---

# Contract Template: Architecture

This template defines how to build and interpret an Architecture Contract.
It is consumed by the `sda-contract-builder` skill (to guide contract creation)
and by the `sda-contract-gatekeeper` skill (to interpret and validate contracts).

---

## Section: Allowed Dependencies

**Purpose:**
Defines exactly which services may depend on which other services, and with what
communication mechanism. Precision matters here: "A may call B" is weaker than
"A may call B only to place holds via POST /accounts/{iban}/holds". The more precise
the allowed dependency, the easier it is to detect violations — and the clearer
the intent becomes to future implementors.

**Builder — Questions:**
1. List all service-to-service dependencies that are currently in use or planned.
   Format: `caller-service → callee-service`
2. For each dependency: which communication mechanism is used?
   (REST sync, gRPC sync, async event consumption, async command, etc.)
3. For each dependency: is the caller allowed to use the full API of the callee,
   or only specific endpoints? If only specific endpoints, list them.
4. Are there dependencies on shared infrastructure (message bus, service mesh, API gateway)?
   List those too.

**Gatekeeper — Interpretation:**
- If a proposed dependency is not listed here, it is not allowed: **RED**.
  The absence of a dependency in this list is a negative constraint, not an oversight.
- If a proposed dependency is listed but uses a different communication mechanism
  than specified, flag as **RED**: "Mechanism mismatch."
- If a proposed dependency uses more than the specified endpoints of the callee,
  flag as **YELLOW**: "Dependency scope exceeds allowed interface."

**Constraints:**
- Each dependency entry must specify caller, callee, and communication mechanism.
- REST dependencies should note whether the full API or specific endpoints are allowed.
- Infrastructure dependencies (message bus, etc.) must be listed separately.

**Example:**

```markdown
| Caller | Callee | Mechanism | Scope |
|--------|--------|-----------|-------|
| `api-gateway` | `accounts-service` | REST (sync) | Full Published Interface |
| `api-gateway` | `transactions-service` | REST (sync) | Full Published Interface |
| `transactions-service` | `accounts-service` | REST (sync) | Hold endpoints only: `POST/DELETE /accounts/{iban}/holds` |
| `lending-service` | `transactions-service` | REST (sync) | Disbursement initiation only: `POST /transfers` |
| `notification-service` | event bus | Async consume | `AccountOpened`, `TransactionCompleted`, `LoanAgreementSigned` |
```

---

## Section: Forbidden Dependencies

**Purpose:**
Explicitly lists what must never happen. The Forbidden Dependencies section is often
the most important section for enforcement — it is what automated tools (ArchUnit, Deptrac,
OPA) most directly check. Unlike Allowed Dependencies, forbidden dependencies do not require
a mechanism qualifier: if the dependency exists at all, in any form, it is a violation.

The most critical forbidden dependencies are typically:
- Cross-service database access (a service reading another service's DB directly)
- Circular dependencies
- Downstream services calling upstream services (reversed dependency directions)
- Shared mutable state between domain services

**Builder — Questions:**
1. Which service-to-service dependencies are explicitly forbidden?
   (Think: what would be an architectural anti-pattern in this system?)
2. Are there cross-service database access scenarios that must be prevented?
3. Are there circular dependency risks between any two services?
4. Are there "consumer-only" services that must never become callers?
5. Are there legacy dependencies that are being actively removed and must not be re-introduced?

**Gatekeeper — Interpretation:**
- Any match with a forbidden dependency is **RED**. No exceptions, no YELLOW.
- Cross-service database access is always **RED**, even if not explicitly listed
  (it violates the general principle of service autonomy).
- Circular dependencies are always **RED** if detected.
- For legacy-removal dependencies: flag as **RED** with note "This dependency is under
  active removal — do not re-introduce."

**Constraints:**
- This section must list at least the inverse of the most critical allowed dependencies.
- Cross-service database access must be listed as forbidden (it is always forbidden,
  but making it explicit aids enforcement).

**Example:**

```markdown
| Forbidden | Reason |
|-----------|--------|
| `lending-service` → `accounts-service` (direct) | Must route via `transactions-service` for fund operations |
| `lending-service` → `accounts-db` | Cross-service DB access violates service autonomy |
| `transactions-service` → `lending-service` | Reversed dependency direction (payments must not know about credit) |
| `notification-service` → any domain service | notification-service is consumer-only; it must never call domain APIs |
| any service → any other service's database | General prohibition: all cross-service data access via API or events only |
```

---

## Section: Allowed Communication Patterns

**Purpose:**
Defines the set of technical communication patterns that are approved for use in this system.
This section governs the "how" of service communication, complementing the "who calls whom"
of the dependency sections. It prevents the gradual adoption of unapproved patterns (e.g.,
GraphQL federation introduced by one team that then becomes a de-facto standard) and ensures
observability, security, and operational consistency.

**Builder — Questions:**
1. Which synchronous communication patterns are allowed?
   (REST/HTTP, gRPC, GraphQL, etc. — specify scope: external-facing, internal-only, etc.)
2. Which asynchronous communication patterns are allowed?
   (Message queue, event bus, pub/sub — specify the technology and message format.)
3. What API specification standards apply?
   (OpenAPI version, Protobuf style guide, AsyncAPI, etc.)
4. What message format and schema registry standards apply for async communication?
5. Are there patterns that are explicitly forbidden?

**Gatekeeper — Interpretation:**
- Using a pattern not listed as allowed is **YELLOW** (requires architecture board review)
  unless a forbidden dependency check applies, in which case it is **RED**.
- Violations of API specification standards are **YELLOW**: "Non-standard API spec format —
  must be converted to [standard] before release."
- Async patterns without schema registry are **YELLOW**: "Schema must be registered before
  event can be published to production."

**Constraints:**
- At minimum, one synchronous and one asynchronous pattern must be listed.
- Technology names must be specific (e.g., "Apache Kafka with CloudEvents 1.0" not just "events").

**Example:**

```markdown
### Allowed

| Pattern | Scope | Standard |
|---------|-------|----------|
| REST/HTTP | All service-to-service and external | OpenAPI 3.x specification required |
| gRPC | Internal service-to-service only | Protobuf 3, not exposed at API gateway |
| Async events via message bus | All async communication | CloudEvents 1.0 format, Avro schema in schema registry |

### Forbidden

| Pattern | Reason |
|---------|--------|
| GraphQL Federation | Not yet adopted; requires architecture board approval before introduction |
| Synchronous calls for non-critical notifications | Use async events instead to decouple |
| Shared database tables | Cross-service state sharing violates service autonomy |
```

---

## Section: Layer Boundaries

**Purpose:**
Defines the internal architectural layering rules within each service. Layer boundaries prevent
architectural erosion inside a single service — the gradual collapse of layering that turns
a clean hexagonal architecture into a tangle of cross-cutting concerns. This section makes
the expected internal structure explicit so that agents and developers do not make conflicting
assumptions about where business logic, infrastructure adapters, and API controllers belong.

**Builder — Questions:**
1. What internal architectural pattern is used in each service?
   (Hexagonal / Ports & Adapters, Clean Architecture, layered, etc.)
2. List the layers in dependency order (outermost → innermost or top → bottom).
3. Which direction may dependencies flow? (e.g., outer layers may depend on inner,
   inner layers must not depend on outer)
4. Are there specific packages, namespaces, or module names that map to each layer?
5. Are there cross-cutting concerns (logging, tracing, security) and how are they handled?

**Gatekeeper — Interpretation:**
- Dependency flowing in the wrong direction (e.g., domain model importing infrastructure)
  is **RED**: "Layer boundary violation."
- Cross-cutting concern logic placed in a domain model is **YELLOW**: "Business logic
  contamination — move to appropriate layer."
- If a service does not follow the specified pattern, flag as **YELLOW** unless
  the violation is structural (changes require significant refactoring), in which case
  flag as **RED** with a note to create a remediation task.

**Constraints:**
- Must specify at least the primary dependency direction rule.
- If multiple services use different patterns, document each service separately.

**Example:**

```markdown
All services follow a Ports & Adapters (Hexagonal) architecture.
Dependency rule: adapters → application → domain. The domain layer has no outbound dependencies.

| Layer | Examples | May depend on |
|-------|----------|---------------|
| Adapters (inbound) | REST controllers, event consumers | Application layer only |
| Adapters (outbound) | Repository implementations, HTTP clients | Domain layer only |
| Application | Use cases, application services | Domain layer only |
| Domain | Entities, value objects, domain services | Nothing (pure domain model) |

Cross-cutting concerns (logging, tracing, metrics) are handled by adapter-layer decorators.
Domain objects must not import logging or tracing libraries.
```

---

## Section: Exception Process

**Purpose:**
Defines how to deviate from this Architecture Contract when a justified exception is required.
Without an exception process, teams either silently violate contracts (which defeats the purpose)
or feel blocked by governance they cannot challenge. The exception process makes it safe to
disagree with a contract — by providing a clear, transparent path to a documented, reviewed
exception rather than a covert workaround.

**Builder — Questions:**
1. What steps must a team follow to request an exception to this contract?
2. Who must review and approve an exception?
3. How must the exception be documented? (ADR, contract amendment, inline annotation?)
4. What is the review timeline? (How long before a team can expect a decision?)
5. Does a granted exception result in a contract version bump with the exception noted?

**Gatekeeper — Interpretation:**
- When returning a **RED** assessment, always reference the exception process:
  "To proceed, follow the exception process defined in §Exception Process."
- If the implementation claims an exception was granted, ask for the ADR or contract
  amendment reference. Without evidence of a documented exception, maintain **RED**.
- If an exception was granted and documented, downgrade the **RED** to **YELLOW**
  with the exception reference noted in the assessment output.

**Constraints:**
- The exception process must name a specific role or team as the approver (not "the team").
- The documentation requirement must be specific (ADR, Jira ticket, contract PR, etc.).

**Example:**

```markdown
Any deviation from this Architecture Contract requires:

1. **Draft an ADR** (Architecture Decision Record) documenting:
   - The specific rule being violated
   - The technical reason why the exception is necessary
   - The proposed mitigation to limit the architectural damage
   - An exit plan (how will the deviation be resolved in a future iteration?)

2. **Architecture board review.** Submit the ADR to the architecture board.
   Expected review time: 5 business days.

3. **Contract amendment.** If approved, the exception is added inline to this contract
   as a versioned footnote, and the contract version is incremented (minor version bump).

4. **Time-bound.** Exceptions are granted for a maximum of two release cycles.
   After that, either the exception is resolved or a new review is required.
```
