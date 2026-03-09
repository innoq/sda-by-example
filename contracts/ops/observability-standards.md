---
contract-type: ops
id: ops/observability-standards
version: "1.0.0"
status: active
owner-team: platform-engineering
owner-contact: platform@example-bank.de
last-reviewed: 2025-03-01
services:
  - api-gateway
  - accounts-service
  - transactions-service
  - lending-service
  - notification-service
regulatory: [BAFIN-BAIT, PSD2-RTS]
enforcement:
  agent-check: true
  ci-check: false
---

# Ops Contract: Observability Standards

This contract defines the mandatory observability requirements for all services in the banking
platform: logging format, metrics naming, distributed tracing, SLOs, and alerting thresholds.
Compliance with this contract is required for production deployment approval. Observability
requirements are derived in part from BAFIN-BAIT (audit trail obligations for IT systems) and
PSD2 RTS (logging for payment services).

---

## Logging Standard

### Format

All log output must be **structured JSON on stdout**. One JSON object per line. No multi-line
log entries. No plain-text log lines in production. Log aggregation pipelines depend on this
format for parsing.

### Mandatory Fields

Every log entry, at every log level, must include the following fields:

| Field | Type | Format | Description |
|-------|------|--------|-------------|
| `timestamp` | String | ISO-8601 with timezone (`2025-03-01T14:32:00.123Z`) | Time of the log event, UTC preferred |
| `level` | Enum | `DEBUG` \| `INFO` \| `WARN` \| `ERROR` \| `FATAL` | Severity level |
| `service` | String | Kebab-case service name (`accounts-service`) | Emitting service |
| `version` | String | SemVer (`1.4.2`) | Deployed service version — enables log correlation during deployments |
| `traceId` | String | W3C trace-id (32 hex chars) | From `traceparent` header; empty string if no trace context |
| `spanId` | String | W3C span-id (16 hex chars) | From `traceparent` header; empty string if no trace context |
| `correlationId` | String | Business-level ID (e.g., `transactionId`, `loanId`) | Enables cross-service log correlation at the business process level |
| `message` | String | Human-readable summary | Must be meaningful without the `context` field |
| `context` | Object | Key-value pairs | Optional additional structured context; subject to masking rules below |

### Log Level Usage Guidelines

| Level | When to Use |
|-------|-------------|
| `DEBUG` | Detailed diagnostic information; disabled in production by default; must comply with masking rules even at DEBUG |
| `INFO` | Normal operational events (request received, event published, process completed) |
| `WARN` | Unexpected but recoverable conditions; retries, fallbacks, deprecated API usage |
| `ERROR` | Failures requiring attention; exceptions, integration failures, SLA breaches |
| `FATAL` | Service cannot continue operating; followed by process exit |

### Forbidden Log Content

The following data categories must never appear in log output at any log level, including DEBUG.
This is a GDPR Art. 25 (data minimisation by design) and BAFIN-BAIT security requirement.

| Forbidden Category | Examples |
|-------------------|---------|
| Full IBANs | `DE89370400440532013000` — must be masked |
| Account balances or credit scores | `balance: 12500.00`, `score: 742` |
| Authentication credentials | OAuth tokens, API keys, passwords (even hashed) |
| Full card numbers (PAN) | Must never appear in any log at any level |
| SCA authentication factors | OTP codes, biometric data |
| Full personal name combined with financial data | Never log name+IBAN together |

### Masking Rules

Where an identifier must appear in logs for correlation purposes, apply the following masking:

| Data Type | Masking Rule | Example |
|-----------|-------------|---------|
| IBAN | Country code + `**...` + last 4 digits | `DE**...1234` |
| Account ID (internal UUID) | Not masked — internal identifier, not personal | `acct_3fa85f64` |
| Transaction amount | Not masked — required for operational debugging | `amount: 250.00` |
| Customer name | Mask to initials if needed for debugging | `J. S.` |
| Email address | Domain part only | `***@example-bank.de` |

---

## Metrics Standard

### Naming Convention

Pattern: `{service}_{subsystem}_{metric_name}_{unit}`

Rules:
- `{service}`: service name with hyphens replaced by underscores (e.g., `accounts_service`)
- `{subsystem}`: component within the service (e.g., `http`, `db`, `event`)
- `{metric_name}`: descriptive name in snake_case
- `{unit}`: Prometheus unit suffix — `_seconds`, `_bytes`, `_total`, `_ratio`
- No abbreviations unless universally understood (`http` not `hypertext_transfer_protocol`)
- All counters must end in `_total`
- All histograms must have `_seconds` or `_bytes` suffix depending on the measured quantity

### Mandatory Metrics: All HTTP Services

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `{service}_http_requests_total` | Counter | `method`, `endpoint_pattern`, `status_code` | Total HTTP requests. Use `endpoint_pattern` (e.g., `/accounts/{iban}`) not literal paths. |
| `{service}_http_request_duration_seconds` | Histogram | `method`, `endpoint_pattern` | Request duration. Buckets: 10ms, 50ms, 100ms, 250ms, 500ms, 1s, 2.5s, 5s, 10s |
| `{service}_http_errors_total` | Counter | `method`, `endpoint_pattern`, `error_type` | HTTP errors; `error_type` enum: `client_error`, `server_error`, `timeout` |

### Mandatory Metrics: Event-Driven Services

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `{service}_event_published_total` | Counter | `event_type`, `status` | Events published; `status`: `success`, `failure` |
| `{service}_event_consumed_total` | Counter | `event_type`, `status` | Events consumed; `status`: `success`, `failure`, `skipped` |
| `{service}_event_consumer_lag` | Gauge | `topic`, `consumer_group` | Consumer lag in number of messages |
| `{service}_event_processing_duration_seconds` | Histogram | `event_type` | Time to process a consumed event |

### Mandatory Metrics: Database-Backed Services

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `{service}_db_query_duration_seconds` | Histogram | `operation`, `table` | DB query duration; `operation`: `select`, `insert`, `update`, `delete` |
| `{service}_db_connection_pool_active` | Gauge | `pool_name` | Active connections in the pool |
| `{service}_db_connection_pool_idle` | Gauge | `pool_name` | Idle connections in the pool |

---

## Distributed Tracing

### Standard

**W3C TraceContext** (RFC 7230 / W3C Recommendation). All inter-service HTTP calls must
propagate the following headers:

| Header | Requirement | Description |
|--------|-------------|-------------|
| `traceparent` | **Mandatory** | Contains `version-traceId-parentId-flags`; must be forwarded or generated on every inter-service call |
| `tracestate` | Optional | Vendor-specific extensions; bank-specific keys must use prefix `bank-` (e.g., `bank-correlationId`) |

If a service receives a request without a `traceparent` header, it must generate a new trace
context (new `traceId` + root `spanId`). It must never forward an empty or invalid `traceparent`.

### Sampling Strategy

| Context | Sampling Rate | Rationale |
|---------|--------------|-----------|
| ERROR traces (any service) | 100% | All errors must be fully traceable |
| `transactions-service` (production) | 100% | BAFIN-BAIT §4.3: payment operations require complete audit trail |
| All other services (production) | 10% | Balance between observability and storage cost |
| All services (dev/staging) | 100% | Full visibility during development |

Sampling rate for `transactions-service` must not be reduced below 100% in production without
architecture board approval. This is a regulatory requirement, not a preference.

### Span Attributes

All spans created by domain services must include:

| Attribute | Required | Description |
|-----------|----------|-------------|
| `service.name` | Yes | Matches the `service` field in log output |
| `service.version` | Yes | Deployed version |
| `business.correlation_id` | Yes | Business-level correlation ID (transactionId, loanId, etc.) |
| `db.system` | Yes (DB spans) | Database type, e.g. `postgresql` |
| `db.statement` | No | Must be omitted if statement contains L3/L4 data |

---

## SLOs

Measurement window: rolling 28-day window. SLOs are measured from the API gateway and from
direct service metrics where applicable.

| Service | Availability SLO | Latency p99 SLO | Notes |
|---------|-----------------|-----------------|-------|
| `accounts-service` | 99.9% | 200ms | Core read/write path |
| `transactions-service` | 99.95% | 500ms | Payment criticality; higher availability target |
| `lending-service` | 99.5% | 2,000ms | Underwriting is computationally intensive; async where possible |
| `notification-service` | 99.5% | N/A (async consumer) | Measured as event lag < 30s for 99.5% of events |
| Event bus (publish, `api-gateway` perspective) | 99.95% | 100ms | Delivery guarantee to broker |

### Error Budget Policy

| Situation | Action |
|-----------|--------|
| SLO breach in any single window | Automatic alert to on-call; root cause documented within 48 hours |
| Two consecutive window breaches | Mandatory architecture review within 5 business days; feature work paused until review complete |
| Three consecutive window breaches | Feature freeze for the affected service until root cause is resolved and SLO is green for one full window |

---

## Alerting Thresholds

### Mandatory Alert Rules: All Services

Every service must configure the following alert rules before production deployment.
Alert routing to PagerDuty is mandatory.

| Condition | Duration | Severity | Routing |
|-----------|----------|----------|---------|
| Error rate > 1% of requests | 5 consecutive minutes | **P2** | On-call engineer |
| Latency p99 > 2× SLO target | 10 consecutive minutes | **P2** | On-call engineer |
| Service health check failing | 60 seconds | **P1** | On-call engineer + team lead |
| Event consumer lag > 10,000 messages | 5 consecutive minutes | **P2** | On-call engineer |
| TLS certificate expiry < 14 days | Immediately | **P2** | Platform team |

### Service-Specific Alert Rules

| Service | Condition | Severity | Rationale |
|---------|-----------|----------|-----------|
| `transactions-service` | Error rate > 0.1% | **P1** | Payment criticality; lower threshold than default |
| `transactions-service` | Latency p99 > 1,000ms | **P1** | Payment SLA risk (SEPA Instant 10s target) |
| `lending-service` | `CreditDecisionMade` events not produced for > 30 min during business hours | **P2** | Credit decisioning SLA risk |
| `accounts-service` | `HoldPlaced` failure rate > 0.5% | **P1** | Hold failures block all payment processing |
| any service | Key rotation failure | **P1** | Regulatory obligation; immediate attention required |

### Severity Definitions

| Severity | Response Time | Examples |
|----------|--------------|---------|
| **P1** | Immediate page; respond within 15 minutes | Service down, payment processing blocked, security incident |
| **P2** | Acknowledge within 30 minutes; resolve within 4 hours during business hours | Elevated error rate, latency degradation, consumer lag |
| **P3** | Next business day | Non-critical configuration warnings, cert expiry > 30 days |
