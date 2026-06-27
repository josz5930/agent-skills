---
name: architecture-planner
description: >-
  This skill should be used when the user wants to design, plan, or evaluate how
  to build or deploy a system or workflow: "design a secure setup for...",
  "architecture for...", "how should I structure / deploy / spin up...", "plan
  the infrastructure for...", agentic or Claude Code / CI workflow design,
  choosing deployment options (serverless vs VM, managed vs self-hosted), IAM /
  auth / zero-trust design, cloud provisioning (AWS / GCP / Azure, Lambda, EC2,
  Cloud Run, Kubernetes), or IaC (OpenTofu / Terraform) layout — even when the
  user never says "architecture". It produces a detailed, decision-justified
  infrastructure and software architecture plan in which every choice is
  resolved against a fixed priority ladder — security (least privilege + strong
  auth) first, then budget control, then low operational complexity, then
  readability — used as the explicit tie-breaker. Prefer this skill over an
  ad-hoc answer whenever the request implies trade-offs between security, cost,
  complexity, and maintainability.
---

# Architecture Planner

Turn a system description into a single, comprehensive architecture plan whose
every meaningful decision is justified — and whose conflicts are resolved by one
fixed ranking.

## The Priority Ladder (the spine of every decision)

When two viable options conflict, choose strictly in this order:

1. **Security** — minimized privileges, strong authentication. *First because
   security failures are often irreversible and uncapped: a leaked long-lived
   credential or an over-scoped role can cost far more than the system ever
   saved. Prefer controls that are **enforced/provable** (cryptographic
   identity, policy engines, IAM conditions) over controls that are merely
   promised in docs or convention.*
2. **Budget control** — *second because runaway cost quietly kills projects.
   Spend should be bounded and observable by design, never discovered on a
   bill.*
3. **Low operational (use) complexity** — *third because systems people can't
   operate confidently degrade and become insecure over time. Fewer moving
   parts, managed where sensible, one-command workflows. Actively resist
   over-engineering.*
4. **Code / config readability** — *last because it genuinely matters for
   maintenance, but never at the expense of the three above.*

State the ladder near the top of every plan, and whenever you resolve a real
trade-off, name which rung decided it. The goal is the **cheapest secure
option, operated as simply as possible, written as cleanly as that allows** —
in that order.

> Equal-security → pick cheaper. Equal-cost → pick simpler to operate.
> Equal-simplicity → pick more readable.

## Before you assert anything: verify live

Architecture plans fail when they rest on stale numbers or deprecated patterns.
Before stating any of the following as fact, **search the web (Tavily or web
search) and cite the source**:

- **Pricing** — compute, storage, egress, per-request/token, managed-service
  fees, model/API costs.
- **Quotas & limits** — default service limits (e.g. Lambda concurrency, vCPU
  caps, API rate limits), model context windows, and **plan quotas** (e.g. what
  a given Claude / SaaS tier actually includes, and current availability of
  features like Claude Code on the web).
- **Current security / IAM best practice** — recommended auth flows, deprecated
  mechanisms, default-secure settings, recently disclosed pitfalls.
- **Service capabilities** — whether a service still exists, region
  availability, and whether a claimed integration is real.

Flagging convention, used inline throughout the plan:

- `[VERIFIED ✓ — <source>]` after a fact you confirmed this session.
- `[VERIFY]` after a number or claim you could not confirm. Never launder an
  unverified figure as if it were checked. If pricing/limits couldn't be
  confirmed, give a clearly-labelled estimate **and** add it to the
  verification TODO list (final section).

This is non-negotiable: a confident wrong number is worse than an honest gap.

## Workflow

1. **Read the request for the four inputs that shape everything**: what's being
   built (functional goal), the environment/stack (named tools, cloud,
   existing quotas like "use my Pro plan by default"), hard constraints (budget
   ceiling, region, compliance), and who/what operates it. If a *must-have* is
   missing and would change the design branch, ask a tight clarifying question
   first — otherwise state a labelled assumption and proceed.
2. **Pick the stack.** Honor any tools the user named. Where they leave a gap,
   default to sensible, widely-used choices, leaning on this known-good
   fallback set unless the context points elsewhere: AWS or GCP; OpenTofu for
   IaC; Cloudflare Zero Trust / Tunnel for ingress and access; Keycloak (OIDC) +
   SPIRE/SPIFFE for human and workload identity; OPA/Rego for authorization;
   FastAPI for service code; Docker Compose for local. Name *why* a default was
   chosen, and note the cheaper/simpler alternative you rejected.
3. **Verify** all pricing/limits/best-practice facts (section above).
4. **Write the plan** using the template below. Write the numbered sections
   first, then distill the **build manifest** at the top from them — it must be a
   faithful summary, never a source of new facts. Scale depth to the problem —
   a two-component design doesn't need ten pages — but never drop the security,
   budget, and simplicity sections; they are the point.

## Output template

Produce one markdown document with these sections. Keep prose tight and
decision-dense; prefer tables for grants and costs.

````markdown
# Architecture Plan: <System Name>

> **Build brief.** <One plain sentence: what a codegen agent is about to build.>
> The manifest below is the agent-ingestible distillation of this plan; the
> numbered sections after it are the authoritative detail and rationale.

```yaml
# BUILD MANIFEST — a downstream codegen agent can parse this and start planning
# files/tasks without reading the full document. Sections 1–10 below are the
# source of truth; this is a faithful summary, not new information.
objective: <one line>
target_stack:           # languages, frameworks, cloud, IaC, key services
  - <e.g. OpenTofu>
  - <e.g. AWS Lambda (ARM)>
components:             # what gets built; enough for the agent to scaffold
  - name: <component>
    responsibility: <one line>
    interface: <API/contract/CLI it exposes or consumes, if any>
constraints:           # HARD rules. Each tagged with the ladder rung it serves.
  # rung ∈ {security, budget, simplicity, readability}. Security rungs are
  # non-negotiable: an agent reading ONLY this block must still inherit them.
  - rule: MUST_NOT <...>
    rung: security
  - rule: MUST <...>
    rung: budget
acceptance_criteria:   # what "done / testable" means, concretely
  - <e.g. debug APK builds and unit tests pass in-sandbox>
build_order:           # RECOMMENDED, not binding — the agent may resequence
  recommended: true
  steps:
    - step: <e.g. bootstrap OIDC identity + budget alarm>
      checkpoint: <how to know it worked>
open_verifications:    # unconfirmed items the agent must not assume away
  - <[VERIFY] item, mirrors section 10>
```

## 1. Summary & Governing Priorities
<2–4 sentences on what's being built.> Decisions in this plan are ranked:
**security → budget → low operational complexity → readability.** The single
biggest trade-off resolved by that ranking here is <...>.

## 2. Requirements, Constraints & Assumptions
- **Functional:** what it must do.
- **Constraints:** budget ceiling, region, compliance, existing quotas to reuse
  (e.g. an existing subscription/plan), latency/throughput targets.
- **Assumptions:** each tagged [VERIFIED ✓ — src] or [VERIFY].

## 3. Threat Model & Trust Boundaries  — *priority 1*
- **Assets** worth protecting (data, credentials, compute, the agent's powers).
- **Actors / adversaries** — include semi-trusted insiders and, where an AI
  agent can execute commands or IaC, **the agent itself** as an actor whose
  blast radius must be bounded.
- **Attack surface & trust boundaries.**
- **Diagram** (Mermaid): components, trust boundaries, and identity/data flows.
  Make boundaries visually explicit (e.g. subgraphs per trust zone).

## 4. Identity, Access & Authentication  — *priority 1*
- **Identities:** human and workload identities, and how each is established
  (prefer short-lived, attested identity — OIDC, mTLS, SPIFFE SVIDs — over
  long-lived keys).
- **Least-privilege grants table:**

  | Principal | Resource | Minimal permission | Why this is the floor |
  |-----------|----------|--------------------|------------------------|

- **Authentication:** the strong mechanism per boundary (MFA, OIDC, mTLS,
  token exchange). No shared or long-lived secrets unless justified.
- **Authorization / enforcement:** where policy is *enforced* (IAM conditions,
  OPA/Rego, service controls) — not just documented.
- **Secrets & credential lifecycle:** issuance, rotation, storage, revocation.
- **Agent-safety controls** (when an AI/automation can act): scoped/short-lived
  credentials for the agent, command/action allowlists, network egress limits,
  a sandbox or non-prod blast radius, and **human-in-the-loop confirmation for
  irreversible or spend-incurring actions.**

## 5. Component & Infrastructure Design
For each component: its responsibility, the chosen implementation, and a
one-line *why this and not the obvious alternative* (citing the rung that
decided it). Include data flow and where state lives.

## 6. Budget & Cost Controls  — *priority 2*
- **Cost model** with verified unit prices [VERIFIED ✓ — src].
- **Estimated monthly cost** (low / expected / high), with the main drivers.
- **Guardrails:** budgets + alerts, quotas/limits, prefer serverless or
  spot/auto-stop for bursty or idle workloads, right-sizing, and reuse of
  existing quotas/free tiers before paid ones.
- **Cost-vs-security trade-offs** and how the ladder resolved each (security
  wins, but pick the *cheapest secure* option).

## 7. Operational Simplicity (Day-2)  — *priority 3*
- **Fewest-steps lifecycle:** ideally one command each to provision, deploy,
  test, and tear down.
- **What the operator must think about** routinely — keep it short.
- **Where security added complexity** and how you kept it manageable.
- **Deliberately omitted:** list things you chose *not* to build to avoid
  over-engineering, and why that's safe.

## 8. Implementation Sketch  — *priority 4 (readability)*
- **Directory layout** (tree).
- **Illustrative skeletons** (not full files): key IaC modules / service code,
  short and idiomatic, showing structure and the security controls in place.
- **Testability:** how to stand up the "testable version" (local emulation,
  ephemeral env, smoke tests).

## 9. Runbook
Ordered steps: provision → deploy → test → operate → **tear down**. Include a
rollback / kill-switch and how to revoke the agent's or service's access fast.

## 10. Open Questions & Verification TODOs
Every [VERIFY] item, plus decisions that hinge on unconfirmed pricing/limits and
any clarifications the user still owes you.
````

## Conventions & quality bar

- **The build manifest is the agent's entry point.** Treat it as a contract a
  downstream codegen agent can act on with no further reading. Keep it a faithful
  distillation — every fact in it must trace to a numbered section. Because an
  agent may act on the manifest *alone*, every security and budget guardrail
  must appear there as an explicit `MUST` / `MUST_NOT` constraint tagged with the
  rung it serves; never leave a rung-1 control implicit or "see below". The
  `build_order` is advisory (`recommended: true`) — a sensible default sequence,
  not a rule the agent must obey.
- **Diagrams:** use Mermaid (`graph` / `flowchart`), with `subgraph` blocks to
  draw trust zones. Show identity flow, not just network arrows. Keep it
  readable — a dozen nodes, not fifty.
- **Least privilege is concrete, not a slogan.** Don't write "scoped IAM role";
  write the specific actions/resources and why nothing less works. If you can't
  name the minimal permission, that's a [VERIFY], not a hand-wave.
- **Reuse what the user already pays for.** If they say "use my X plan/quota by
  default," make that the primary path and treat paid alternatives as
  fallbacks with a cost note.
- **Resist over-engineering.** Prefer one managed service to three self-hosted
  ones unless security or cost clearly demands otherwise. Every added component
  is operational surface area — justify it on a rung of the ladder or drop it.
- **One voice.** The plan is a single clean document. No meta-commentary about
  these instructions, no agent/workflow scaffolding language in the output.
- **Cite as you assert.** Pricing, limits, and best-practice claims carry their
  source inline; unverified ones carry `[VERIFY]`.

## Worked snippet (illustrative tone, not a full plan)

> **Decision — agent execution surface.** Claude Code may run shell commands
> *or* generate-and-apply OpenTofu. Granting it standing cloud admin would
> minimize friction (simplicity, rung 3) but hands an automated actor an
> unbounded blast radius — **rung 1 overrides.** Floor: the agent authenticates
> via short-lived OIDC-federated credentials [VERIFY: provider's web-identity
> federation setup] to a role scoped to a dedicated project/account, with
> `apply` gated behind human confirmation and a hard monthly budget alarm
> (rung 2). Net: slightly more setup, dramatically smaller worst case.

When in doubt, re-read the ladder and ask: *what's the cheapest option that
doesn't weaken security, that a tired operator could still run at 2am?*
