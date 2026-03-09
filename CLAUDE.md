# CLAUDE.md — SDA by Example

This repository is a reference implementation for **Spec-Driven Architecture (SDA)**.
It demonstrates how versioned, agent-readable contracts keep a distributed system coherent —
and how two Claude skills (Builder and Gatekeeper) integrate contracts into agentic workflows.

The example domain is a **Banking/Fintech platform** with three domain services:
Accounts, Transactions (PSD2), and Lending (BAFIN-MaRisk).

---

## Repository Structure

```
contract-templates/    ← Shared templates: Builder uses them to ask questions,
│                        Gatekeeper uses them to interpret contract sections.
│  domain.md           ← Template for Domain Contracts
│  architecture.md     ← Template for Architecture Contracts
│  ops.md              ← Template for Ops Contracts
│  security.md         ← Template for Security Contracts
│
skills/                ← Claude Code skills (copy to .claude/skills/<name>/SKILL.md to activate)
│  sda-contract-builder.md      ← Guided interview → writes a new contract file
│  sda-contract-gatekeeper.md   ← Validates implementation decisions → GREEN/YELLOW/RED
│
contracts/             ← Active SDA contracts for the banking example system
│  domains/
│    accounts.md       ← Accounts domain (balance, holds, IBAN)
│    transactions.md   ← Transactions domain (SEPA, PSD2 SCA)
│    lending.md        ← Lending domain (credit decisions, MaRisk BA 7)
│  architecture/
│    service-dependencies.md    ← Allowed/forbidden service dependencies
│  ops/
│    observability-standards.md ← Logging, metrics, tracing, SLOs
│  security/
│    data-classification.md     ← L1–L4 classification, GDPR, PSD2, BAFIN-BAIT
```

---

## Contract Format

All contracts use **YAML frontmatter + Markdown body**.

Key frontmatter fields:
- `contract-type`: `domain` | `architecture` | `ops` | `security`
- `id`: `<type>/<slug>` (e.g., `domain/accounts`)
- `enforcement.agent-check: true` — signals the Gatekeeper to load this contract
- `enforcement.ci-check: false` — future: static pipeline check (not yet active)
- `regulatory` — optional list of applicable frameworks (GDPR, PSD2, BAFIN-MaRisk, etc.)

---

## Skills

### Gatekeeper — `skills/sda-contract-gatekeeper.md`

Invoke before touching a domain boundary, service dependency, or interface.
Invoke after completing an implementation step.

```
/sda-contract-gatekeeper pre: I plan to call accounts-service directly from lending-service
/sda-contract-gatekeeper post: I implemented the TransactionCompleted event consumer
```

The Gatekeeper:
1. Globs `contracts/**/*.md`, reads frontmatter to find relevant contracts
2. Loads the corresponding `contract-templates/<type>.md` for interpretation hints
3. Returns **GREEN** (proceed) / **YELLOW** (proceed with documented justification) / **RED** (stop)

**The Gatekeeper is read-only.** It never writes files. Unknown territory defaults to YELLOW,
never GREEN.

### Builder — `skills/sda-contract-builder.md`

Invoke to create a new contract via guided interview.

```
/sda-contract-builder domain
/sda-contract-builder architecture
```

The Builder:
1. Reads the corresponding template from `contract-templates/<type>.md`
2. Conducts a section-by-section interview using the template's Builder questions
3. Writes the completed contract to `contracts/<type>/<slug>.md`

---

## Working with This Repository

### Before implementing anything that touches a domain boundary:
Consult the Gatekeeper. Relevant contracts are in `contracts/`. The most critical rules are:
- Forbidden dependencies: `contracts/architecture/service-dependencies.md §Forbidden Dependencies`
- Data classification: `contracts/security/data-classification.md` — L3/L4 data requires
  specific auth, encryption, and access logging
- Domain write authority: each domain contract's `§Invariants` defines who may write what

### When adding a new contract:
Use the Builder skill. Do not write contracts from scratch — the templates in
`contract-templates/` define the required sections, constraints, and format.

### When modifying a contract:
Increment the `version` field (semver). If the change restricts what was previously allowed,
treat it as a breaking change (major version bump). Add a note in the commit message
referencing why the contract changed.

### Contract language:
Contracts, templates, and skills are in **English**.
