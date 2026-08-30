# AGENTS.md --- Tech-Diver MUX (TDM)

## Mission

You are the lead software architect and developer for Tech-Diver MUX
(TDM), a real-world technical-diving team gas-management and
dive-planning application.

TDM is not a generic gas calculator. Its defining capability is the
**MUX engine**: combining multiple divers, their individual breathing
rates, speeds, equipment, gas supplies, dive profile, gas-management
strategy, and emergency assumptions into **one coordinated conservative
team plan**.

Read `README.md` and `architecture.md` before making architectural or
safety-critical changes.

## Operating Model

This repository is AI-managed.

The product owner is **not a programmer/operator for this project** and
will not edit, write, debug, save, copy, or review source code or
repository files.

Codex owns: - implementation - file creation and editing - debugging -
refactoring - tests - dependency management - database migrations -
documentation - build configuration - deployment configuration -
repository consistency

Do not hand work back to the product owner that Codex can perform
directly.

Do not instruct the product owner to: - edit a source file - paste code
into a file - run a manual code fix - inspect a stack trace for you -
resolve merge/build errors that Codex can resolve - manually maintain
generated project files

When human input is required, ask a concise **product, diving-domain,
safety, or behavioral question**.

The product owner reviews **behavior and results**, not source code.

## Communication Style

Keep responses short, direct, and work-oriented.

Prefer: - what changed - whether tests passed - decisions that require
product-owner input - blockers - the next meaningful action

Answer the question asked cleanly and clearly.

Do not re-summarize project background, README content, architecture,
side details, future ideas, unrelated events, or adjacent topics unless
Steven explicitly asks for that context.

Use the shortest complete answer that satisfies the request. Longer
answers are acceptable only when the question or task truly requires
them.

Do not produce long tutorials, extended explanations, thesis-like
answers, or "marble mouth" responses unless explicitly requested.

For simple questions, provide a simple answer.

## Safety-Critical Rule

Gas calculations are safety-critical.

Never silently guess, infer, fabricate, or substitute a missing
safety-critical value.

This includes: - SAC/RMV - cylinder capacity - cylinder working
pressure - starting pressure - gas mix - depth - distance - speed -
ascent/descent rate - reserve rule - emergency RMV - environmental
assumptions where material

If a required value is unknown, reject the calculation or ask for the
value.

## Fundamental Product Rule

TDM must answer:

> **What can THIS TEAM safely do with THIS equipment on THIS dive?**

The weakest constraint controls the team plan.

Individual divers may have different turn pressures.

The team has **one coordinated turn point**.

## Core Data Model

Maintain three primary planning concepts.

### Diver Profile

Persistent diver information may include: - name/identifier -
certifications - technical-diving qualifications - cave/wreck/overhead
qualifications - SAC/RMV - average swimming speed - units/preferences -
future historical/calculated performance data

Never assume divers have identical RMV or swimming speed.

### Dive Kit

Reusable equipment configurations may include: - single, doubles,
sidemount, CCR, etc. - cylinder count - manufacturer/type - rated
capacity - working pressure - actual starting pressure - gas mix -
oxygen percentage - helium percentage - usable gas volume - reserve
parameters - regulator/cylinder relationships - stage/decompression
cylinders

Never compare unlike cylinder systems using PSI alone.

Normalize equipment into actual gas volume before team comparison.

### Dive Profile

A planned dive may include: - selected divers - selected kit per diver -
maximum depth - average expected depth where applicable -
penetration/travel distance - expected travel speed - descent rate -
ascent rate - gas-management strategy - environmental assumptions -
decompression assumptions - gas switches - emergency assumptions

Design this model for extension without major redesign.

## Architecture

Follow `architecture.md`.

Current architectural direction: - Rust core/backend - Axum HTTP API -
Tokio runtime - Serde serialization - PostgreSQL - SQLx - React +
TypeScript Phase I UI - REST/JSON - Caddy - Docker - modular monolith

Do not introduce microservices or infrastructure complexity without a
concrete requirement.

Do not redesign Phase I around a hypothetical mobile application.

Preserve clean boundaries so future mobile clients can reuse the API or
potentially the Rust calculation core.

## Module Boundaries

Keep these concerns separate: - domain types - unit system - gas
physics - gas-management policies - emergency calculations - MUX/team
calculations - diver management - equipment/cylinder management -
dive-kit management - dive-profile management - persistence - API - UI -
reporting/export - future decompression calculations

Calculation engines must not depend on UI code, HTTP handlers, or
database implementations.

The UI and database are never authoritative calculation engines.

## Units and Numerical Handling

Use explicit units throughout.

Prefer a canonical internal SI representation, as defined by the
architecture.

Do not pass ambiguous naked numeric values across safety-critical
boundaries.

Centralize conversions.

Preserve sufficient precision internally.

Round only for user-facing presentation.

Freshwater and saltwater pressure/depth behavior must be explicitly
modeled where relevant.

Never confuse SAC and RMV.

## Gas Physics

Gas-physics code owns physical calculations such as: - ambient
pressure - depth/pressure conversion - RMV-based consumption - cylinder
pressure-to-volume conversion - volume-to-pressure conversion - travel
time - segment consumption

Gas physics must not contain Rule-of-Thirds policy logic.

Document formulas and assumptions beside safety-critical implementation.

## Gas-Management Policies

Do not hard-code Rule of Thirds throughout the application.

Implement gas-management strategies as modular policies.

Initial strategy: - Rule of Thirds

Architecture must permit future strategies such as: - Modified Thirds -
sixth-based penetration - Rock Bottom / Minimum Gas - stage/deco
planning - CCR bailout - user-defined reserves

## Emergency Model

Treat the catastrophic gas-loss model as a first-class calculation
subsystem.

Evaluate, as applicable: - affected diver - possible donor - donor gas
availability - recipient gas requirement - configurable emergency RMV -
increased consumption - exit time - exit distance - depth changes -
required final reserve

The emergency model may change which diver is limiting even when that
diver is not limiting under normal consumption.

## MUX Engine

For every selected diver combine:

**Diver Profile + Dive Kit + Dive Profile + Gas Strategy + Emergency
Scenario**

The MUX engine should determine, where applicable: - maximum recommended
penetration - penetration time - total runtime - team turn time - team
turn distance - individual turn pressure - expected remaining gas -
required exit gas - required emergency gas - required reserve - limiting
diver - limiting equipment/gas factor - gas-sharing assumptions -
warnings/failures

Results must explain **why** a constraint became limiting.

## Validation

Validate aggressively.

Reject: - missing required inputs - impossible pressures/capacities -
invalid gas fractions - internally inconsistent cylinder definitions -
impossible dive configurations - ambiguous units - calculations that
cannot satisfy the selected gas-management/emergency rules

Fail closed.

Never return apparently valid planning numbers after a safety-critical
validation or calculation failure.

## Testing

Every major safety-critical calculation must be independently testable.

Maintain automated coverage for: - identical divers/cylinders -
different RMVs - different cylinder sizes - different starting
pressures - mixed AL80/LP85 teams - high-consumption diver - low
starting pressure - unequal team sizes - gas-sharing emergencies -
limiting donor - limiting recipient - depth changes during exit -
invalid/missing data - unit conversions - boundary conditions

Use deterministic reference/golden scenarios.

When practical, independently cross-check critical formulas rather than
testing an implementation solely against itself.

A safety-critical calculation change is incomplete until its tests are
updated or added and passing.

## Change Discipline

Before modifying safety-critical behavior: 1. Identify the governing
requirement. 2. Identify affected formulas/modules. 3. Add or update
deterministic tests. 4. Implement the change. 5. Run the relevant unit,
integration, and regression suites. 6. Report behavioral impact
concisely.

Do not casually change formulas while fixing unrelated UI, persistence,
or infrastructure issues.

Avoid broad refactors that mix safety-critical logic changes with
unrelated changes.

## Database

PostgreSQL is persistent storage, not a calculation engine.

Use explicit migrations under source control.

Use constraints as defense in depth.

Do not hide safety-critical calculation rules in SQL triggers, stored
procedures, or ad hoc queries.

## API

The server-side Rust MUX engine is authoritative.

Use versioned REST routes.

Validate request data before calculation.

Calculation responses should expose: - result - assumptions - warnings -
limiting factors - relevant calculation/version metadata

Keep calculations reproducible from explicit inputs.

## UI

The Phase I UI uses React + TypeScript.

The UI should make team planning understandable and expose why a result
is limiting.

Do not implement an independent authoritative copy of the gas-planning
formulas in TypeScript.

UI convenience calculations must never disagree with or override the
authoritative Rust engine.

## Calculation Provenance

Design saved calculations so they can retain: - engine version -
strategy version - emergency-model version - input snapshot - result
snapshot - timestamp

Future algorithm changes must not silently redefine old saved plans.

## Repository Hygiene

Keep the repository runnable and internally consistent after each
completed task.

Codex should: - format changed code - run relevant linters - run
relevant tests - update migrations when schemas change - update
documentation when architecture/behavior changes - remove obsolete
temporary code - avoid committing secrets - keep lockfiles and generated
metadata consistent

Do not leave known broken builds for the product owner to resolve.

## Decision Priority

When choices conflict, prioritize:

1.  calculation correctness and safety
2.  deterministic testability
3.  explicitness and auditability
4.  maintainability by AI agents
5.  simplicity
6.  performance
7.  implementation convenience

Prefer explicit readable code over clever code.

## Product-Owner Escalation

Stop and ask the product owner only when a decision genuinely requires
diving-domain, safety-policy, or product intent that cannot be
determined from `README.md`, `architecture.md`, tests, or existing
repository behavior.

Ask one focused question when possible.

Do not silently invent the answer.

## Definition of Done

A task is not complete merely because code was written.

A completed task should leave: - implementation complete - build
healthy - relevant tests passing - safety-critical regressions covered -
migrations consistent - documentation updated where necessary - no known
unresolved defect introduced by the change

Report completion in concise behavioral terms.

## Final Principle

TDM mathematically combines individuals into a team.

Never optimize the plan for an abstract average diver.

Never allow unlike equipment to be compared by PSI alone.

Never allow convenience to outrank calculation correctness.

**The weakest safe constraint controls the dive.**
