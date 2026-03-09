---
contract-type: security
id: security/data-classification
version: "1.0.0"
status: active
owner-team: information-security
owner-contact: ciso@example-bank.de
last-reviewed: 2025-03-01
data-assets:
  - account-data
  - transaction-data
  - credit-data
  - customer-pii
  - authentication-credentials
regulatory: [GDPR, PSD2, PSD2-RTS, BAFIN-MaRisk, BAFIN-BAIT, ISO-27001]
enforcement:
  agent-check: true
  ci-check: false
  tool-hint: >
    OPA policy bundle: /policies/security/data-classification.rego.
    Run policy check: opa eval -b /policies/security -d input.json 'data.security.classification.allow'
---

# Security Contract: Data Classification

This contract establishes the data classification taxonomy for the banking platform and
derives the technical security controls required for each classification level. All services
handling customer or financial data must classify that data using this taxonomy and apply the
corresponding controls before storing, processing, or transmitting it.

> **Disclaimer:** This contract is a demonstration example for Spec-Driven Architecture.
> It is not legal advice. Regulatory references are intended to illustrate the structure
> of a compliance-aware security contract, not to constitute a complete compliance assessment.

---

## Data Classification Levels

Four classification levels are defined. Every data asset handled by this system must be
assigned to exactly one level. When in doubt, assign the higher level.

| Level | Label | Definition | Examples |
|-------|-------|------------|---------|
| **L1** | PUBLIC | No sensitivity. Freely shareable without restriction. No personal data, no financial data. | Published interest rate tables, product descriptions, open API specs, privacy policy |
| **L2** | INTERNAL | Business-sensitive but carrying no personal data obligation and no direct regulatory constraint. Not for external publication but no harm if accidentally disclosed. | Internal process documentation, non-personal aggregated metrics (e.g., total transactions per day), team handbooks |
| **L3** | CONFIDENTIAL | Personal data under GDPR Art. 4(1) or financial data subject to banking confidentiality (Bankgeheimnis). Disclosure causes measurable harm to individuals or the bank's legal position. | Full name, postal address, email address, IBAN, account balance, transaction history, loan application data, customer date of birth |
| **L4** | RESTRICTED | Highest sensitivity. GDPR special category data (Art. 9) or data requiring HSM-level cryptographic protection. Disclosure causes severe harm: regulatory violation, identity theft, financial fraud. | Internal credit scores, biometric authentication data, SCA authentication factors, full card numbers (PAN), authentication credentials (hashed passwords, OAuth secrets), health data used in insurance underwriting |

**Classification notes:**

- IBAN is L3 even when appearing without name or balance. Alone, an IBAN is a personal
  identifier under GDPR (linked to an identifiable person through the account relationship).
- Transaction metadata (amount, date, merchant) is L3. Transaction patterns over time
  may be L4 if used for profiling.
- Aggregated, anonymized data (e.g., total transaction volume by region) may be L2
  if individual re-identification is demonstrably impossible.

---

## Authentication & Authorization Requirements

### By Classification Level

| Level | Service-to-Service Auth | User-Facing Auth | User Consent Required | Access Logging |
|-------|------------------------|------------------|-----------------------|----------------|
| **L1** | No auth required for read; service token for write | None | No | No |
| **L2** | OAuth 2.0 Bearer (service token) + mTLS for internal calls | None (no user data) | No | No |
| **L3** | OAuth 2.0 Bearer (service token) + mTLS | OAuth 2.0 with user consent scope; field-level access control enforced at API layer | Yes — GDPR Art. 6 lawful basis must be documented (contractual necessity or explicit consent) | Yes — log `data_subject_id`, `accessor_service`, `timestamp`, `fields_accessed` (GDPR Art. 30 record of processing) |
| **L4** | OAuth 2.0 Bearer + mTLS + dedicated L4 access scope | OAuth 2.0 + user consent + verified MFA in authentication chain; dedicated L4 scope in access token | Yes — GDPR Art. 6 + explicit consent for special category data (Art. 9) | Yes — as L3, plus quarterly access review by CISO team; all access events retained for 5 years (BAFIN-BAIT) |

### PSD2-Specific Rules

Third-Party Provider (TPP) access under PSD2 Open Banking:

- TPPs accessing account information (L3) require:
  - OAuth 2.0 with `accounts:read` scope
  - Explicit, revocable customer consent (consent ID must be included in the access token)
  - TPP must be registered with the EBA (European Banking Authority) register
- TPPs initiating payments (L3/L4) require:
  - OAuth 2.0 with `payments:write` scope
  - PSD2-compliant SCA (Strong Customer Authentication) per RTS Art. 97
- TPP access logs must be retained for 5 years (PSD2 Art. 69)
- Consent revocation must be technically enforceable within 24 hours of customer request

---

## Encryption Requirements

### At Rest

| Level | Requirement | Key Management |
|-------|-------------|----------------|
| **L1** | No encryption required | N/A |
| **L2** | No encryption required | N/A |
| **L3** | AES-256, database tablespace encryption minimum | Managed key store (approved cloud KMS or on-premises equivalent) |
| **L4** | AES-256, field-level encryption per data subject (not just tablespace) | HSM required — FIPS 140-2 Level 3 or equivalent. Software-only key management is not sufficient for L4. |

### In Transit

| Requirement | Detail |
|-------------|--------|
| Minimum TLS version | TLS 1.2 for all communication |
| Preferred TLS version | TLS 1.3 |
| L4 service-to-service | Mutual TLS (mTLS) required — both parties must present valid certificates |
| Certificate authority | Internal CA or commercially trusted CA; self-signed certificates forbidden in production |
| TLS for event bus | All connections to the message bus broker must use TLS 1.2+ |

### Key Rotation

| Level | Rotation Interval | Revocation After Rotation |
|-------|------------------|--------------------------|
| **L3** | Annual | 30-day grace period; old key retired after all data re-encrypted |
| **L4** | Every 90 days | Immediate revocation after re-encryption completes; no grace period |

Key rotation events must be logged and auditable. Rotation failures must trigger a P1 alert.

---

## Data Residency

### Permitted Processing Locations

| Level | Permitted Locations | Conditions |
|-------|---------------------|------------|
| **L1, L2** | No geographic restriction | — |
| **L3, L4** | European Economic Area (EEA) only | Default policy — see non-EEA transfer rules below |

### Non-EEA Transfer Rules

All transfers of L3 or L4 data outside the EEA require one of:

1. **Adequacy decision** by the European Commission (GDPR Art. 45) — the destination
   country must be on the current EC adequacy list.
2. **Standard Contractual Clauses (SCCs)** (2021 version) combined with a documented
   **Transfer Impact Assessment (TIA)** evaluating whether the destination country's
   surveillance laws undermine the SCC protections (ECJ Schrems II ruling, C-311/18).

**Currently approved non-EEA transfers:** None. Any new non-EEA transfer of L3/L4 data
requires written DPO (Data Protection Officer) approval and must be documented in the
GDPR Art. 30 Record of Processing Activities.

### Cloud Provider Constraints

- US-based cloud regions: prohibited for L3/L4 data without documented SCC + TIA.
  The existence of US CLOUD Act and FISA §702 surveillance programs requires TIA
  to demonstrate SCCs are effective.
- Approved cloud providers for L3/L4: maintained by the Information Security team;
  consult before selecting a new cloud service for any workload handling L3+ data.
- BAFIN-BAIT §5: Material outsourcing of IT functions (including cloud) to third parties
  requires advance notification to BAFIN and an exit strategy. Verify with Compliance
  before new cloud service agreements for core banking functions.

---

## Compliance Notes

### Applicable Regulatory Frameworks

| Framework | Relevance to This Contract |
|-----------|---------------------------|
| **GDPR (EU 2016/679)** | L3/L4 data classification; lawful basis for processing; data subject rights; data residency (Ch. V); breach notification (Art. 33/34) |
| **PSD2 (EU 2015/2366) + RTS (EU 2018/389)** | TPP access controls; SCA requirements; consent management; TPP audit log retention |
| **BAFIN-MaRisk (AT 7.2)** | Credit data handling and retention; access controls for credit decisioning data |
| **BAFIN-BAIT (§4, §5)** | IT systems security; logging requirements; cloud outsourcing notification; access review requirements |
| **ISO 27001** | Information security management system baseline; risk assessment foundation |

### GDPR Data Subject Rights

The following rights must be technically implementable for all L3/L4 data without manual
workaround. Service implementations must not technically block these rights.

| Right | Article | Technical Requirement |
|-------|---------|----------------------|
| Right of access | Art. 15 | API endpoint per domain to export all personal data for a given `data_subject_id`; response within 30 days |
| Right to erasure | Art. 17 | Erasure workflow with automated legal hold check; data subject to regulatory retention periods cannot be erased until retention expires |
| Right to data portability | Art. 20 | Machine-readable export in JSON or CSV covering all L3 data provided by the data subject |
| Right to object to automated decisions | Art. 22 | Human review endpoint mandatory for all automated credit decisions (`POST /loan-applications/{id}/human-review`) |
| Right to restriction of processing | Art. 18 | Processing restriction flag per data subject; restricted data must not be used for any new processing beyond storage |

### Regulatory Data Retention Periods

Data must not be deleted before the minimum retention period and must not be retained beyond
the maximum where specified.

| Data Category | Minimum Retention | Maximum Retention | Regulation |
|---------------|------------------|------------------|------------|
| Transaction records | 10 years from transaction date | No maximum specified | BAFIN §238 HGB (commercial code) |
| Credit application and decision data | 6 years post-application closure | — | BAFIN-MaRisk AT 7 |
| Customer PII (post-relationship end) | 5 years | 10 years | BAFIN §257 HGB |
| TPP access logs | 5 years | — | PSD2 Art. 69 |
| Security and audit event logs | 5 years | — | BAFIN-BAIT §4.3 |
| SCA logs | 18 months | — | PSD2 RTS Art. 36 |

### Data Breach Notification

In the event of a confirmed or suspected data breach involving L3/L4 data:

- **GDPR Art. 33:** Notify the competent supervisory authority (in Germany: the relevant
  State DPA or BfDI) within 72 hours of becoming aware of the breach. If notification is
  not possible within 72 hours, provide the notification with reasons for the delay.
- **GDPR Art. 34:** If the breach is likely to result in high risk to natural persons (e.g.,
  L4 financial or credential data), notify affected data subjects without undue delay.
- **BAFIN-BAIT §4 Abs. 3:** Significant IT operational incidents (including breaches
  affecting core banking functions) must be reported to BAFIN.
- The incident response runbook is maintained by the Information Security team and reviewed
  quarterly. All engineers must complete annual incident response training.
