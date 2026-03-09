---
template-type: security
template-version: "1.0.0"
description: >
  Template for Security Contracts in Spec-Driven Architecture. A Security Contract
  defines data classification levels, authentication and authorization requirements
  per classification, encryption standards, data residency constraints, and
  regulatory compliance rules. It translates regulatory obligations (GDPR, PSD2,
  BAFIN) into enforceable technical requirements that agents and CI/CD pipelines
  can check.
contract-frontmatter-required:
  - contract-type
  - id
  - version
  - status
  - owner-team
  - owner-contact
  - last-reviewed
  - data-assets
  - regulatory
  - enforcement.agent-check
contract-frontmatter-optional:
  - enforcement.tool-hint
sections:
  - data-classification-levels
  - authentication-authorization-requirements
  - encryption-requirements
  - data-residency
  - compliance-notes
---

# Contract Template: Security

This template defines how to build and interpret a Security Contract.
It is consumed by the `sda-contract-builder` skill (to guide contract creation)
and by the `sda-contract-gatekeeper` skill (to interpret and validate contracts).

---

## Section: Data Classification Levels

**Purpose:**
Establishes the canonical data classification taxonomy for the system. Data classification
is the foundation of all other security controls: without knowing what sensitivity level
a piece of data has, you cannot determine what auth, encryption, or residency rules apply.
The classification levels section makes sensitivity a first-class property that agents,
developers, and automated tools can reference unambiguously.

**Builder — Questions:**
1. How many classification levels does this system need?
   (Recommended starting point: 4 levels — PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED)
2. For each level: what is the label and its precise definition?
3. For each level: list 3–5 concrete examples of data that belongs to this level.
4. Are there any existing regulatory classifications that must map to these levels?
   (e.g., GDPR "personal data" → which level? GDPR "special category" → which level?)
5. For financial systems: are there data assets subject to banking secrecy (Bankgeheimnis)?
   Map these to the appropriate level.

**Gatekeeper — Interpretation:**
- If an implementation stores, transmits, or processes data without specifying its
  classification level, flag as **YELLOW**: "Data asset classification unknown —
  determine classification before applying security controls."
- If an implementation treats L3 or L4 data as if it were L1 or L2 (e.g., logging
  it in plain text, returning it without auth), flag as **RED**: "Data classification
  downgrade — [data field] is classified [level], not [assumed level]."
- When reviewing API responses: check if the response includes data classified higher
  than the endpoint's auth requirement allows.
- Cross-reference with the Ops Contract logging standard: flag any log entries that
  include L3/L4 data in unmasked form as **RED**.

**Constraints:**
- Minimum 3 classification levels.
- Each level must have a definition and concrete examples.
- GDPR personal data (Art. 4(1)) must be assigned a level of at minimum L3.
- GDPR special category data (Art. 9) must be assigned L4.

**Example:**

```markdown
| Level | Label | Definition | Examples |
|-------|-------|------------|---------|
| L1 | PUBLIC | No sensitivity. Freely shareable. | Product brochures, published interest rates, open API specs |
| L2 | INTERNAL | Business-sensitive but no personal data or regulatory obligation | Internal process docs, non-personal aggregated metrics, team wikis |
| L3 | CONFIDENTIAL | Personal data under GDPR or financial data subject to banking confidentiality | Full name, address, IBAN, account balance, transaction history, email address |
| L4 | RESTRICTED | Highest sensitivity. Special category data under GDPR or data requiring HSM-level protection | Credit scores, biometric authentication data, SCA factors, full card numbers (PAN), authentication credentials |
```

---

## Section: Authentication & Authorization Requirements

**Purpose:**
Defines the minimum authentication and authorization controls required for accessing data
at each classification level. This section bridges the gap between data classification and
technical implementation: it answers "if I have L3 data, what must I check before returning it?"
For regulated systems, this section operationalizes regulatory obligations (PSD2 SCA, GDPR
lawful basis, BAFIN access controls) as implementable technical requirements.

**Builder — Questions:**
1. For each classification level: what authentication mechanism is required?
   (None / API key / OAuth 2.0 Bearer / OAuth 2.0 + user consent / MFA / etc.)
2. For each classification level: what authorization scope or role is required?
3. For user-facing data (L3, L4): is user consent required beyond authentication?
   (GDPR lawful basis: consent vs. legitimate interest vs. contractual necessity)
4. Are there access logging requirements per classification level?
   (Required by GDPR Art. 30 for personal data processing; BAFIN-BAIT for financial data)
5. For L4 data: are there periodic access review requirements?
   (Recommended: quarterly access reviews for privileged access)

**Gatekeeper — Interpretation:**
- Implementation accessing L3 data without OAuth 2.0 and explicit scope is **RED**:
  "L3 data requires OAuth 2.0 with user consent scope — missing auth control."
- Implementation accessing L4 data without MFA verification is **RED**:
  "L4 data requires MFA in the authentication chain — missing auth control."
- Implementation storing L3/L4 access events without audit log is **RED**:
  "GDPR Art. 30 requires logging of personal data access — audit log missing."
- Service-to-service call accessing L3 data without mTLS (for internal APIs) is **YELLOW**:
  "mTLS recommended for L3 service-to-service access — evaluate risk."
- Overly broad authorization scopes (e.g., `admin` scope for reading account balance)
  are **YELLOW**: "Authorization scope too broad — apply principle of least privilege."

**Constraints:**
- Must specify auth requirements for all defined classification levels.
- L3 minimum: OAuth 2.0 with user-consent scope.
- L4 minimum: OAuth 2.0 + MFA.
- Access logging requirements must reference a specific log field (not just "log access").

**Example:**

```markdown
| Level | Authentication | Authorization | Access Logging |
|-------|---------------|---------------|----------------|
| L1 | None required for read | Service token for write | No |
| L2 | OAuth 2.0 Bearer (service token) | Service identity claim | No |
| L3 | OAuth 2.0 + user consent scope | Consent ID in token; field-level access control | Yes — log `data_subject_id`, `accessor_service`, `timestamp`, `fields_accessed` (GDPR Art. 30) |
| L4 | OAuth 2.0 + user consent + MFA verified | MFA verification claim in token; dedicated L4 scope | Yes — as L3, plus quarterly access review by CISO team (BAFIN-BAIT requirement) |

Third-party provider (TPP) access under PSD2:
- TPP accessing L3 account data requires: OAuth 2.0 with `accounts:read` scope + explicit customer consent ID
- TPP access logs must be retained for 5 years (PSD2 Article 69)
```

---

## Section: Encryption Requirements

**Purpose:**
Defines the encryption requirements for data at rest and in transit, keyed to classification
level. Encryption requirements translate data classification into cryptographic controls:
they answer "how must this data be protected while stored and while moving between services?"
In a regulated environment, encryption requirements are often directly derived from regulatory
mandates — making them explicit in the contract ensures regulatory obligations are not
accidentally omitted during implementation.

**Builder — Questions:**
1. For each classification level: what encryption is required at rest?
   (Database-level / field-level / no requirement)
2. For L4 data: is Hardware Security Module (HSM) key management required?
3. What encryption algorithm and key length is mandated?
   (Minimum for regulated systems: AES-256)
4. What key rotation schedule is required per classification level?
5. For data in transit: what minimum TLS version is required?
   (Minimum: TLS 1.2; preferred: TLS 1.3)
6. For L4 service-to-service communication: is mutual TLS (mTLS) required?
7. Are there backup encryption requirements?

**Gatekeeper — Interpretation:**
- L3 or L4 data stored without encryption at rest is **RED**: "Data classification requires
  encryption at rest — unencrypted storage of [level] data is prohibited."
- L4 data encrypted without HSM key management is **RED**: "L4 classification requires
  HSM — software-only key management insufficient."
- TLS version below the mandated minimum is **RED**: "TLS version below minimum —
  update to TLS [minimum version]."
- Key rotation schedule not configured or exceeding the mandated interval is **YELLOW**:
  "Key rotation not configured — required interval for [level] data is [interval]."
- Backup encryption at lower strength than primary storage is **YELLOW**: "Backup
  encryption must meet the same standard as primary storage."

**Constraints:**
- Must specify at rest encryption for L3 and L4 separately.
- Must specify the encryption algorithm and key length (not just "encrypted").
- Must specify TLS minimum version for in-transit encryption.
- Key rotation schedule must be defined in concrete time units.

**Example:**

```markdown
### Encryption at Rest

| Level | Requirement | Key Management |
|-------|-------------|----------------|
| L1, L2 | No encryption required | N/A |
| L3 | AES-256, database tablespace encryption | Managed key store (e.g., AWS KMS, Azure Key Vault) |
| L4 | AES-256, field-level encryption per data subject | HSM required (FIPS 140-2 Level 3 or equivalent) |

### Encryption in Transit

- All service communication: TLS 1.2 minimum, TLS 1.3 preferred
- L4 service-to-service communication: mTLS required
- TLS certificates: issued by internal CA or commercially trusted CA (no self-signed in production)

### Key Rotation

| Level | Rotation Interval | Action on Rotation |
|-------|------------------|--------------------|
| L3 | Annual | Re-encrypt all L3 data, revoke old key after 30-day grace period |
| L4 | Every 90 days | Re-encrypt all L4 data, revoke old key immediately after rotation |
```

---

## Section: Data Residency

**Purpose:**
Defines where data may be stored and processed, based on classification level. Data residency
requirements are primarily driven by GDPR Chapter V for personal data (L3/L4): personal data
may only flow to third countries with adequate protection. For German-regulated financial
institutions, BAFIN additionally requires that critical data processing remains within the EEA
or in a jurisdiction with adequate oversight. Without explicit residency rules, cloud deployments
gradually expand to regions that violate these requirements — often without any single decision
point that could have been caught.

**Builder — Questions:**
1. For L3 and L4 data: which geographic regions are permitted?
   (EEA only / EEA + adequacy decision countries / no restriction)
2. Are there specific cloud regions that are explicitly approved?
3. Are there specific cloud regions that are explicitly forbidden?
4. For transfers to non-EEA countries: what legal basis applies?
   (Standard Contractual Clauses, adequacy decision, binding corporate rules)
5. Has a Transfer Impact Assessment (TIA) been conducted for any non-EEA transfers?
6. For BAFIN-regulated entities: which data qualifies as "critical" under BAFIN-BAIT
   and is subject to notification requirements for cloud outsourcing?

**Gatekeeper — Interpretation:**
- Implementation proposing L3/L4 data processing in a non-EEA cloud region without
  documented SCCs or adequacy decision is **RED**: "GDPR Chapter V violation — L3/L4
  data cannot be processed in [region] without adequate legal basis."
- Implementation proposing L3/L4 data processing in a US-based cloud region without
  documented Schrems-II compliant safeguards (SCCs + TIA) is **RED**: "Schrems II
  compliance required — document SCCs and Transfer Impact Assessment."
- Implementation proposing L1/L2 data in unrestricted regions is **GREEN** (no residency
  constraint applies).
- BAFIN-regulated critical data processing in a cloud environment without notification
  is **YELLOW**: "BAFIN-BAIT cloud outsourcing notification may be required — verify
  with compliance team."

**Constraints:**
- Must explicitly state permitted regions for L3 and L4.
- Must address non-EEA transfers (even if to say "prohibited").
- Must reference Schrems II compliance approach for any US cloud services.

**Example:**

```markdown
### Permitted Processing Locations

| Level | Permitted Regions | Conditions |
|-------|------------------|------------|
| L1, L2 | No restriction | — |
| L3, L4 | European Economic Area (EEA) only | Default — see exceptions below |

### Non-EEA Transfer Rules

All L3 and L4 data transfers outside the EEA require:
1. An adequacy decision by the European Commission (GDPR Art. 45), OR
2. Standard Contractual Clauses (SCCs, 2021 version) + Transfer Impact Assessment (TIA)

Currently approved non-EEA transfers: none.
Any new non-EEA transfer of L3/L4 data requires DPO (Data Protection Officer) sign-off.

### Cloud Provider Constraints

- US-based cloud regions: prohibited for L3/L4 data without SCC + TIA documentation
  (ECJ Schrems II ruling, C-311/18)
- Approved cloud providers for L3/L4: [list maintained by information security team]
- BAFIN-BAIT: cloud outsourcing of critical banking functions requires advance notification
  to BAFIN (BAIT §5 Abs. 2). Verify with compliance team before new cloud deployments.
```

---

## Section: Compliance Notes

**Purpose:**
Documents the specific regulatory frameworks that apply to this security contract, with
references to the articles and rules that drive the requirements in the other sections.
Compliance notes serve two purposes: they make the regulatory derivation transparent
(so future maintainers understand why a rule exists) and they provide the Gatekeeper
with regulatory context to include in its RED assessments (so that a violated rule can
be linked to a specific regulatory obligation, not just an internal policy).

**Builder — Questions:**
1. Which regulatory frameworks apply to this system?
   (GDPR, PSD2, BAFIN-MaRisk, BAFIN-BAIT, ISO 27001, SOC 2, PCI-DSS, etc.)
2. For each framework: which specific articles or sections are most relevant?
3. Are there GDPR data subject rights that must be technically implementable?
   (Art. 15 right of access, Art. 17 right to erasure, Art. 20 right to data portability)
4. Are there data retention and deletion schedules mandated by regulation?
5. Are there regular compliance audits or assessments, and what evidence do they require?
6. Are there notification obligations in case of data breach?
   (GDPR Art. 33: 72-hour notification to supervisory authority)

**Gatekeeper — Interpretation:**
- Include relevant regulatory references in all **RED** and **YELLOW** assessments.
  Example: "L3 data processed without consent — GDPR Art. 6 lawful basis required."
- If an implementation makes GDPR Art. 22 automated decisions (credit decisions, risk scoring)
  without a human review path, flag as **RED**: "GDPR Art. 22 — automated decisions must be
  challengeable by data subject."
- If an implementation processes GDPR special category data (Art. 9) without explicit
  consent, flag as **RED**: "GDPR Art. 9 — special category data requires explicit consent."
- Retention period violations (keeping data longer or shorter than mandated) are **YELLOW**:
  "Retention period mismatch — [data type] requires [mandated period] per [regulation]."

**Constraints:**
- Must list all applicable regulatory frameworks.
- GDPR data subject rights (Art. 15, 17, 20) must be mentioned for any system processing
  personal data (L3+).
- Retention periods must be specified for all data categories with regulatory retention
  obligations.

**Example:**

```markdown
### Applicable Regulatory Frameworks

| Framework | Relevance |
|-----------|-----------|
| GDPR (EU 2016/679) | Personal data of EU residents (L3, L4) |
| PSD2 (EU 2015/2366) + RTS (EU 2018/389) | Payment services, SCA, TPP access |
| BAFIN-MaRisk (AT 7.2) | Credit risk data handling (lending domain) |
| BAFIN-BAIT (§4, §5) | IT systems, logging, cloud outsourcing |
| ISO 27001 | Information security management (certification basis) |

### GDPR Data Subject Rights

The following rights must be technically implementable for all L3/L4 data:

| Right | Article | Implementation Requirement |
|-------|---------|---------------------------|
| Right of access | Art. 15 | API endpoint to export all personal data per data subject ID |
| Right to erasure | Art. 17 | Erasure workflow with legal hold check (cannot erase data required for regulatory retention) |
| Right to portability | Art. 20 | Machine-readable export (JSON or CSV) of personal data |
| Right to object to automated decisions | Art. 22 | Human review path for all automated credit decisions |

### Regulatory Data Retention Periods

| Data Category | Minimum Retention | Maximum Retention | Regulation |
|---------------|------------------|------------------|------------|
| Transaction records | 10 years | — | BAFIN §238 HGB |
| Credit application data | 6 years post-closure | — | BAFIN-MaRisk |
| Customer PII (post-relationship) | 5 years | 10 years | BAFIN §257 HGB |
| TPP access logs | 5 years | — | PSD2 Art. 69 |
| Security event logs | 5 years | — | BAFIN-BAIT §4.3 |

### Data Breach Notification

- GDPR Art. 33: Supervisory authority notification within 72 hours of breach discovery
- GDPR Art. 34: Data subject notification without undue delay if high risk to individuals
- BAFIN: Significant IT incidents must be reported under BAFIN-BAIT §4 Abs. 3
- Incident response runbook: [link to internal runbook]
```
