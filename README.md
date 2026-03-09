# Spec-Driven Architecture (SDA) — by Example

> SDD describes how a system is built. SDA describes how systems stay coherent.

This repository contains examples, skills, and reference implementations for **Spec-Driven Architecture (SDA)**.

---

## What is SDA?

Agentic development makes implementation fast, cheap, and scalable. What it doesn't solve: how a portfolio of systems stays coherent across a distributed landscape. Architectural knowledge today lives implicitly — in ADRs nobody reads, in Confluence pages that go stale, in conventions taken for granted until an agent doesn't know them.

**Spec-Driven Architecture (SDA)** applies the core principle of Spec-Driven Development to the architecture level: the architecture is not diagrams, not ADRs — it is **versioned, agent-readable contracts** — enforceable in agentic workflows and in the CI/CD pipeline.

SDA does not replace existing governance. It automates it.

---

## Core Concept: Contracts

A contract in SDA is an explicit, verifiable statement about expectations, guarantees, and boundaries between domains, services, teams, and agents. It is treated like source code: versioned, reviewed, part of the pipeline.

**Important distinction:** SDA contracts must not be confused with API or Consumer-Driven Contracts (such as Pact). While those govern the behavior between two services at the implementation level, SDA contracts operate at the architecture, governance, and domain level. They do not define how two services communicate — they define what is permitted in a system at all. A contract defines both **boundaries** (what is not allowed) and **guarantees** (what consumers can rely on) — obligations and rights at the same time.

### Contract Types

| Type | Describes | Example Enforcement |
|------|-----------|---------------------|
| **Domain Contract** | The domain's external semantic: Ubiquitous Language, invariants, Published Interface, Consumption Rules | Review by Gatekeeper skill |
| **Architecture Contract** | Allowed dependencies, communication patterns, layer boundaries | YAML policy against dependency graph (e.g. ArchUnit, Deptrac) |
| **Ops Contract** | Operational requirements: metrics, tracing, logging standards | Structured checks against service configuration |
| **Security Contract** | Data classifications, auth mechanisms, compliance rules | OPA policies against generated code |
| **Test Contract** | Quality gates and coverage requirements for a domain | CI quality gate |
| **Deployment Contract** | Infrastructure requirements, runtime constraints | Policy-as-code in the deployment process |

This list is not exhaustive — which contracts a system needs depends on its specific requirements.

---

## Getting Started: A Contract in Markdown

The simplest entry point is a Markdown file in the repository. No special format, no tool dependency — just explicit, versioned knowledge that is equally readable for humans and agents.

```markdown
# Domain Contract: Ordering

## Owner
Team Checkout, checkout@example.com

## Ubiquitous Language
- **Order**: A confirmed purchase intent by a customer, containing at least one item.
- **OrderItem**: A single product line with quantity and price fixed at the time of ordering.
- **OrderStatus**: Enum — PENDING, CONFIRMED, SHIPPED, CANCELLED

## Invariants
- An Order always contains at least one OrderItem.
- The total price is frozen at the time of order creation and never changes retroactively.
- A CANCELLED Order cannot be reactivated.

## Published Interface
- REST API: `POST /orders`, `GET /orders/{id}`
- Events: `OrderConfirmed`, `OrderCancelled` (schema see /contracts/events/ordering.json)

## What this domain does NOT own
- Payment processing (→ Payment Domain)
- Inventory management (→ Inventory Domain)

## Consumption Rules
- Direct database access is not permitted.
- Status changes only via the API — never through direct event manipulation.
```

---

## Contracts in Agentic Workflows: The Gatekeeper

A contract in the repository is worthless if the agent doesn't know it. The solution is a **Gatekeeper skill** — a specialized agent that knows the contracts and plays two roles in the workflow:

**1. Advisor (pre-implementation):** Before an implementation agent begins a task, it consults the gatekeeper. The gatekeeper delivers the relevant contract context and assesses the planned approach: green (compliant), yellow (risk, justification required), red (violation).

**2. Reviewer (post-step):** At the end of each workflow step, the gatekeeper validates the result against the contracts. The workflow only continues on a green signal.

```markdown
---
name: contract-gatekeeper
description: Use this skill to validate implementation decisions against active
  domain, architecture, ops, and security contracts. Invoke before any
  non-trivial implementation step and after each workflow step completes.
---

# Contract Gatekeeper

You are the contract gatekeeper for this system.
You know all active Domain Contracts, Architecture Contracts, Ops Contracts,
and Security Contracts.

## When consulted (pre-implementation)

An agent asks whether a planned approach is contract-compliant.

1. Identify which contracts are relevant to the request.
2. Respond with the relevant contract sections.
3. Return a clear assessment:
   - **green**: compliant, proceed.
   - **yellow**: risk identified, document justification before proceeding.
   - **red**: violation, do not proceed without contract change.

## When reviewing (post-step)

An agent submits an implementation result for review.

1. Check the result against all relevant contracts.
2. Return structured feedback:
   - which contracts were checked
   - what is compliant
   - what is not compliant, with specific reference to the violated rule

## Contracts

[Embed relevant contracts here or retrieve via MCP at runtime]
```

The workflow agent is instructed such that it cannot skip the gatekeeper:

```markdown
# Implementation Agent — Workflow Instructions (excerpt)

## Contract Compliance

Before any implementation decision that touches a domain boundary, a dependency,
or an interface: consult the contract-gatekeeper skill.

Provide: what you intend to implement, in which domain, and which resources
you plan to use.

After completing each step: submit the result to the contract-gatekeeper for review.
Do not proceed until the gatekeeper returns green, or a justified yellow
is documented.
```

---

## Contracts in the CI/CD Pipeline

The same logic works without an agentic workflow — as an entry point into existing CI/CD pipelines. A dedicated pipeline step calls an agent that checks every commit against the relevant contracts. Existing CI/CD tools are sufficient for this.

This makes contracts a pragmatic starting point: teams that haven't yet switched to agentic workflows can introduce contracts and immediately benefit from automated checking. Those who start with the CI step have already laid the foundation when the move to SDA workflows follows.

---

## Governance Without Permanent Objection

SDA fundamentally changes the governance role. Those who previously had to enforce governance in reviews gain a tool that kicks in earlier: contracts that apply from the very first commit. The responsibility remains the same — what changes is when it takes effect.

Architecture Review Boards don't disappear. They regain their actual purpose: making decisions, formulating contracts, deliberately setting boundaries. What falls away is the constant overhead of enforcing those decisions anew in every context.

Automated contract checking makes governance a property of the process, not a gatekeeper role.

---

## Maturity Path

```
Markdown in the repo
    ↓
Contract repository (central, versioned, consistency-checked)
    ↓
Contract server via MCP (dynamic delivery to agents at runtime)
```

---

## Further Reading

- Blog post: `my-content/spec-driven-architecture/blogpost.adoc`
- Reference: [Spec-Driven Development (GitHub spec-kit)](https://github.com/github/spec-kit/blob/main/spec-driven.md)
- MCP protocol: [Model Context Protocol](https://spec.modelcontextprotocol.io)
