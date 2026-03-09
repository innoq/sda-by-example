---
template-type: ops
template-version: "1.0.0"
description: >
  Template for Ops Contracts in Spec-Driven Architecture. An Ops Contract defines
  the operational requirements that every covered service must fulfill: logging
  standards, metrics conventions, distributed tracing, SLOs, and alerting thresholds.
  It makes observability a first-class governance concern rather than a team-by-team
  convention that erodes under deadline pressure.
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
  - regulatory
sections:
  - logging-standard
  - metrics-standard
  - distributed-tracing
  - slos
  - alerting-thresholds
---

# Contract Template: Ops

This template defines how to build and interpret an Ops Contract.
It is consumed by the `sda-contract-builder` skill (to guide contract creation)
and by the `sda-contract-gatekeeper` skill (to interpret and validate contracts).

---

## Section: Logging Standard

**Purpose:**
Defines the mandatory log format and field set for all covered services. Consistent
structured logging is the foundation of operational insight in distributed systems.
Without a shared standard, correlating logs across services becomes manual work, audit
trails break, and regulatory log retention requirements (BAFIN-BAIT, PSD2) cannot be
verified automatically. The logging standard makes log correlation a property of the
system, not of the operator.

**Builder — Questions:**
1. What log format is required? (Structured JSON is strongly recommended for agent-parseable logs.)
2. What fields are mandatory in every log entry?
   (At minimum: `timestamp`, `level`, `service`, `traceId`, `correlationId`, `message`)
3. What log levels are defined and when should each be used?
4. What data must NEVER appear in logs?
   (Personal data, financial identifiers, credentials — cross-reference the security contract.)
5. For any sensitive identifiers that must appear in logs (e.g., for correlation):
   what masking rules apply?
6. Are there regulatory requirements for log retention? (e.g., BAFIN-BAIT audit trail)

**Gatekeeper — Interpretation:**
- Missing mandatory fields in a logging implementation are **RED**: "Non-compliant log format —
  missing required field [field name]."
- Unmasked sensitive data in log output (IBAN, credit scores, tokens) is **RED**:
  "GDPR Art. 25 violation — data minimisation by design requires masking [field]."
- Wrong log format (plain text instead of structured JSON) is **RED**.
- Missing log retention configuration is **YELLOW**: "Retention not configured —
  verify compliance with [regulatory reference]."
- Overly verbose logging of L3/L4 data at DEBUG level is **YELLOW**: "DEBUG logs must
  not bypass data classification rules in production environments."

**Constraints:**
- Must specify a concrete format (e.g., structured JSON) not just "structured."
- Must list all mandatory fields with their data types.
- Must explicitly list at least the top 3 categories of data that must never appear in logs.
- Masking rules must be specific (e.g., "last 4 digits of IBAN only") not vague.

**Example:**

```markdown
### Format

Structured JSON on stdout. One JSON object per line. No multi-line log entries.

### Mandatory Fields

| Field | Type | Description |
|-------|------|-------------|
| `timestamp` | ISO-8601 with timezone | Log event time, e.g. `2025-03-01T14:32:00.123Z` |
| `level` | Enum | `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL` |
| `service` | String | Service name, e.g. `accounts-service` |
| `version` | SemVer | Deployed service version |
| `traceId` | String | W3C trace-id from `traceparent` header |
| `spanId` | String | W3C span-id from `traceparent` header |
| `correlationId` | String | Business-level correlation ID (e.g., transactionId) |
| `message` | String | Human-readable summary of the log event |
| `context` | Object | Optional structured context (see masking rules) |

### Forbidden Log Content

The following must never appear in log output, even at DEBUG level:

- Full IBANs (mask as `DE**...XXXX` — country code + last 4 digits only)
- Account balances or credit scores
- OAuth tokens, API keys, or any credentials
- Full names combined with financial data (GDPR pseudonymisation requirement)

### Retention

Log data must be retained for a minimum of 5 years per BAFIN-BAIT §4.3.
Retention configuration must be validated at service deployment time.
```

---

## Section: Metrics Standard

**Purpose:**
Defines the naming convention and mandatory metric set for all covered services. Consistent
metrics enable cross-service dashboards, automated SLO burn-rate alerts, and capacity planning
that works across the entire system. Without a shared convention, every team invents its own
names and a system-wide view becomes impossible to build without manual translation.

**Builder — Questions:**
1. What metrics naming convention is used?
   (Recommended: `<service>_<subsystem>_<name>_{unit}` following Prometheus conventions)
2. What metrics are mandatory for every service?
   (At minimum: request count, request duration histogram, error rate)
3. What labels/dimensions are mandatory on each metric?
4. Are there domain-specific mandatory metrics?
   (e.g., event publish/consume counts for event-driven services)
5. What metrics backend/format is used? (Prometheus, OpenTelemetry, Datadog, etc.)

**Gatekeeper — Interpretation:**
- Missing mandatory metrics are **YELLOW**: "Required metric not exposed —
  add [metric name] before production deployment."
- Naming convention violations are **YELLOW**: "Metric name does not follow convention.
  Expected: [correct name], found: [actual name]."
- Missing mandatory labels are **YELLOW**: "Required label [label name] missing from
  [metric name]."
- Using a different metrics backend than specified is **YELLOW** unless the contract
  explicitly forbids alternatives, in which case **RED**.

**Constraints:**
- Naming convention must be formally specified (pattern, not just examples).
- Unit suffixes must follow Prometheus conventions: `_seconds`, `_bytes`, `_total`, `_ratio`.
- Must list mandatory metrics per service category (HTTP services, event consumers, etc.).

**Example:**

```markdown
### Naming Convention

Pattern: `{service}_{subsystem}_{metric_name}_{unit}`

- `{service}`: service name with hyphens replaced by underscores (e.g., `accounts_service`)
- `{unit}`: Prometheus suffix — `_seconds`, `_bytes`, `_total`, `_ratio`
- No custom abbreviations; use full words

### Mandatory Metrics: All HTTP Services

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `{service}_http_requests_total` | Counter | `method`, `endpoint`, `status_code` | Total HTTP requests |
| `{service}_http_request_duration_seconds` | Histogram | `method`, `endpoint` | Request latency |
| `{service}_http_errors_total` | Counter | `method`, `endpoint`, `error_type` | HTTP errors (4xx, 5xx) |

### Mandatory Metrics: Event-Driven Services

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `{service}_event_published_total` | Counter | `event_type`, `status` | Events published |
| `{service}_event_consumed_total` | Counter | `event_type`, `status` | Events consumed |
| `{service}_event_consumer_lag` | Gauge | `topic`, `consumer_group` | Consumer lag |
```

---

## Section: Distributed Tracing

**Purpose:**
Defines the distributed tracing standard that enables end-to-end request visibility across
services. In a distributed system, understanding a single customer request requires stitching
together spans from multiple services. Without a shared tracing standard, this is impossible:
trace IDs are not propagated, spans from different services cannot be correlated, and debugging
production issues becomes a forensic exercise. Distributed tracing is also a regulatory
requirement: BAFIN-BAIT mandates audit trails for payment-relevant operations.

**Builder — Questions:**
1. Which tracing standard is used? (W3C TraceContext, B3, AWS X-Ray, OpenTelemetry, etc.)
2. Which HTTP headers must be propagated between services?
3. What sampling strategy is used?
   (Fixed percentage, head-based, tail-based, 100% for errors, etc.)
4. Are there specific flows that require 100% sampling? (Regulatory requirement)
5. What tracing backend is used? (Jaeger, Zipkin, Datadog, etc.)
6. Are there `tracestate` vendor extension fields in use?

**Gatekeeper — Interpretation:**
- Missing `traceparent` header propagation in inter-service calls is **RED**:
  "W3C TraceContext propagation missing — trace correlation broken."
- Incorrect sampling configuration for regulated flows (e.g., payment flows not at 100%)
  is **RED**: "Regulatory requirement violated — [service] must use 100% sampling in production."
- Using a different tracing standard than specified is **YELLOW**: "Non-standard tracing
  implementation — requires migration plan."
- Missing span attributes on domain-relevant operations is **YELLOW**: "Span attributes
  insufficient for operational debugging."

**Constraints:**
- Must specify the exact header name(s) to propagate.
- Sampling rules must distinguish between environment (dev/staging/prod) and traffic type.
- If regulatory sampling requirements exist, they must be explicit and linked to the regulation.

**Example:**

```markdown
### Standard

W3C TraceContext (RFC 7230). All inter-service HTTP calls must propagate:
- `traceparent`: mandatory
- `tracestate`: optional, prefix bank-specific keys with `bank-`

### Sampling

| Context | Rate |
|---------|------|
| ERROR traces | 100% (all environments) |
| Normal traffic (dev/staging) | 100% |
| Normal traffic (production) | 10% |
| Payment flows (transactions-service, production) | 100% — BAFIN-BAIT audit trail requirement |

### Span Attributes

All spans in domain services must include:
- `service.name`: service name
- `service.version`: deployed version
- `business.correlation_id`: business-level correlation ID (e.g., transactionId)
```

---

## Section: SLOs

**Purpose:**
Defines the Service Level Objectives for each covered service. SLOs make operational
reliability a governance contract, not a hope. They give platform teams the authority to
enforce reliability as a first-class requirement, give development teams clear targets,
and give product teams an honest basis for customer-facing SLA commitments. Without
explicit SLOs, "the service is down" is a matter of opinion; with SLOs, it is a fact.

**Builder — Questions:**
1. For each covered service: what is the availability SLO? (percentage, measurement window)
2. For each covered service: what is the latency SLO? (p50, p99, p999 — at minimum p99)
3. What is the error budget policy when an SLO is breached?
   (Alert threshold, freeze deployments, escalation path)
4. Are there different SLOs for different API endpoints or traffic types?
   (e.g., synchronous payment API vs. batch reporting endpoint)
5. What is the SLO measurement window? (rolling 30-day, calendar month, etc.)

**Gatekeeper — Interpretation:**
- SLO section is primarily **informational context** for the Gatekeeper, not a source of
  RED/GREEN assessments on individual implementations.
- When reviewing a service implementation, include the relevant SLO in the assessment
  context: "This service must meet [SLO]. Ensure load testing covers the p99 latency target."
- If an implementation choice is likely to violate the SLO (e.g., a synchronous call chain
  that adds 5 hops for a p99 target of 200ms), flag as **YELLOW**: "Architecture risk —
  proposed implementation may not meet the [service] p99 SLO of [Xms]. Consider [alternative]."

**Constraints:**
- Must specify at minimum availability and p99 latency per service.
- Measurement window must be specified.
- Error budget policy must be defined (even if just "alert to on-call").

**Example:**

```markdown
Measurement window: rolling 28-day window.

| Service | Availability SLO | Latency p99 SLO | Notes |
|---------|-----------------|-----------------|-------|
| `accounts-service` | 99.9% | 200ms | Core account read/write |
| `transactions-service` | 99.95% | 500ms | Payment criticality |
| `lending-service` | 99.5% | 2000ms | Underwriting is async — sync API can be slower |
| Event bus (publish) | 99.95% | 100ms publish p99 | Delivery guarantee |

### Error Budget Policy

- SLO breach → automatic alert to on-call (PagerDuty)
- Two consecutive window breaches → mandatory architecture review within 5 business days
- Three consecutive window breaches → feature freeze until root cause is resolved
```

---

## Section: Alerting Thresholds

**Purpose:**
Defines the mandatory alert rules that every covered service must configure. Alerts are
the operational immune system: they are what converts SLO breaches from silent data into
actionable incidents. Without mandatory alert thresholds, the SLO section becomes aspirational
rather than operational. The alert thresholds section makes observability contractually
complete — not just "we log and expose metrics" but "we will know when something goes wrong."

**Builder — Questions:**
1. What error rate threshold triggers an alert, and at what severity?
2. What latency threshold triggers an alert?
3. What availability threshold (uptime check failure) triggers an alert?
4. For event-driven services: what consumer lag threshold triggers an alert?
5. What incident management system is used? (PagerDuty, OpsGenie, etc.)
6. Are there service-specific alerts beyond the general rules?

**Gatekeeper — Interpretation:**
- If a service implementation does not reference alert configuration, flag as **YELLOW**:
  "No alert configuration found — mandatory alerts must be configured before production."
- If alert thresholds are significantly weaker than specified (e.g., 5% error rate threshold
  vs. the mandated 1%), flag as **YELLOW**: "Alert threshold below contract requirement."
- If the incident management integration is missing, flag as **YELLOW**: "Incident
  management not configured — alerts have no routing target."

**Constraints:**
- Must specify at minimum error rate, latency, and availability alert thresholds.
- Must name the incident management system and severity levels.
- Must distinguish between P1 (immediate) and P2 (non-immediate) severities at minimum.

**Example:**

```markdown
### Mandatory Alert Rules: All Services

| Condition | Duration | Severity | Action |
|-----------|----------|----------|--------|
| Error rate > 1% | 5 minutes | P2 | PagerDuty alert to on-call |
| Latency p99 > 2× SLO target | 10 minutes | P2 | PagerDuty alert to on-call |
| Service unavailable | 60 seconds | P1 | PagerDuty immediate page |
| Event consumer lag > 10,000 messages | 5 minutes | P2 | PagerDuty alert to on-call |

### Service-Specific Rules

| Service | Condition | Severity | Reason |
|---------|-----------|----------|--------|
| `transactions-service` | Error rate > 0.1% | P1 | Payment criticality — lower threshold than default |
| `lending-service` | `CreditDecision` events not consumed for > 30 min | P2 | Lending SLA risk |
```
