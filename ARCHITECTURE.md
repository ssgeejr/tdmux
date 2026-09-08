# Tech-Diver MUX (TDM) — Architecture

## 1. Purpose

This document defines the governing architectural decisions and software design for Tech-Diver MUX (TDM).

TDM is safety-critical dive-planning software. The MUX calculation engine is the core product. User interfaces, persistence, APIs, and deployment infrastructure exist around that engine and must not contain authoritative gas-planning logic.

The architecture must support the product defined in `README.md` and the engineering instructions in `AGENTS.md`.

## 2. Architectural Rules

### Architectural Rule 01: Docker-Only Runtime

TDM runtime services must run under Docker.

This rule applies to:
- PostgreSQL and any future persistent database
- Rust API/backend services
- React/web frontend services
- Reverse proxy/web server services
- Migration runners
- Background workers, if introduced
- Supporting infrastructure services, if introduced

The project must not depend on a developer or server manually installing and running PostgreSQL, Caddy, the API server, the UI server, or other runtime services directly on the host operating system.

Allowed host-level dependencies are limited to development and orchestration tools needed to build, test, or operate the Dockerized system, such as:
- Docker / Docker Compose
- Git
- Repository tooling used by AI engineering agents
- Local editors or inspection tools

All normal application execution must be reproducible through Docker configuration committed to the repository.

Architectural consequences:
- Database configuration belongs in Docker-managed service definitions.
- Web/API service configuration belongs in Docker-managed service definitions.
- Migrations must be runnable from Docker-managed tooling.
- Deployment documentation must describe Docker-based operation.
- Local development must prefer Docker Compose or equivalent Docker orchestration.
- Native host execution may exist only as an optional AI/developer convenience and must not be the authoritative runtime path.

This rule is non-negotiable unless the product owner explicitly changes it in this document.

## 3. Development Model

TDM is an AI-managed software project.

The human product owner defines product intent, technical-diving rules, safety requirements, expected behavior, and acceptance decisions.

AI engineering agents own:
- Architecture and implementation
- Source files and repository maintenance
- Database schemas and migrations
- Dependency management
- Builds and configuration
- Automated tests and regression tests
- Debugging and refactoring
- Documentation
- Deployment configuration
- Versioning and release preparation

Do not create workflows that require the product owner to edit, write, debug, save, copy, or review source code.

When human input is required, ask domain, safety, product, or behavioral questions rather than coding questions.

## 4. Architectural Style

Use a **modular monolith**.

Do not introduce microservices unless a future requirement clearly justifies them.

Primary boundaries:
- Domain model
- Gas physics
- Gas-management strategies
- Emergency calculations
- MUX/team calculations
- Diver management
- Cylinder/equipment management
- Dive-kit management
- Dive-profile management
- Persistence
- API
- User interface
- Reporting/export
- Future decompression calculations

Calculation engines must not depend on UI, HTTP, or database components.

## 5. Technology Stack

### Core / Backend
- Language: **Rust**
- HTTP framework: **Axum**
- Async runtime: **Tokio**
- Serialization: **Serde**
- Database access: **SQLx**

### Database
- **PostgreSQL**

### Phase I User Interface
- **TypeScript**
- **React**

### API
- **REST/JSON**
- Version API routes from the beginning, e.g. `/api/v1/...`
- Calculation requests should be able to operate on complete request payloads without requiring every intermediate plan edit to be persisted.

### Deployment
- **Linux**
- **Docker-only runtime**
- **Docker Compose** for local and initial server orchestration unless replaced by another Docker-native orchestrator
- **Caddy** as TLS termination and reverse proxy
- PostgreSQL as a separate Docker-managed persistent service

Avoid unnecessary infrastructure such as Kubernetes, Kafka, service meshes, distributed caches, or serverless calculation functions during initial development.

Any service required for the application to run must have a repository-owned Docker definition before it is considered part of the supported architecture.

## 6. Repository Structure

Initial target structure:

```text
tdmux/
├── README.md
├── AGENTS.md
├── ARCHITECTURE.md
├── docs/
├── crates/
│   ├── tdm-domain/
│   ├── tdm-physics/
│   ├── tdm-gas-management/
│   ├── tdm-emergency/
│   ├── tdm-mux/
│   ├── tdm-persistence/
│   └── tdm-api/
├── web/
│   └── tdm-ui/
├── migrations/
├── tests/
├── docker/
└── compose.yaml
```

The exact structure may evolve, but module boundaries must remain explicit.

Docker configuration is part of the architecture, not an afterthought. It must stay source-controlled and aligned with the application, database, migration, and reverse-proxy design.

## 7. Dependency Direction

Dependencies flow inward toward the domain and calculation engines.

```text
React UI
   |
REST API
   |
Application / MUX orchestration
   |
+-----------------------------+
| MUX Engine                  |
| Emergency Engine            |
| Gas-Management Strategies   |
+-----------------------------+
   |
Gas Physics
   |
Domain Types
```

Persistence is accessed through explicit repository interfaces.

Forbidden:
- UI code implementing authoritative gas formulas
- SQL implementing authoritative dive-planning calculations
- HTTP handlers containing core gas-management logic
- Calculation modules importing UI components

## 8. Domain Model

The domain layer defines the vocabulary and safety-critical types used throughout TDM.

Initial entities include:
- Diver
- Certification / qualification data
- Cylinder
- Gas mix
- Dive kit
- Dive profile
- Dive team
- Team participant
- Environment
- Gas-management strategy
- Emergency scenario
- Dive-plan request
- Dive-plan result

Use explicit value types for safety-critical quantities rather than untyped numeric primitives wherever practical.

Examples:
- Pressure
- Gas volume
- RMV
- Depth
- Distance
- Velocity
- Duration
- Ambient pressure
- Gas fraction

Invalid states should be difficult to construct.

## 9. Units

Units must always be explicit.

Never store or pass ambiguous values such as `depth = 100` or `pressure = 3000` without a defined unit.

Use a single canonical internal unit system. The preferred architecture is **SI internally**, with imperial/metric conversion at system boundaries and presentation layers.

Examples:
- Pressure: Pa or another explicitly defined canonical pressure unit
- Gas volume: liters
- Distance/depth: meters
- Duration: seconds
- Velocity: meters/second
- Gas fractions: normalized decimal values

User-facing values may be displayed in PSI, cubic feet, feet, feet/minute, bar, liters, meters, etc.

RMV is the canonical breathing-rate concept for calculations. SAC may be accepted as a user-facing label, import term, or onboarding aid, but it must be normalized into the explicit RMV value type before safety-critical planning logic runs. If SAC is supplied as a cylinder-pressure rate, the cylinder context must be explicit before conversion.

Conversions must be centralized, deterministic, and tested.

No rounding inside safety-critical calculations. Round only for presentation.

## 10. Cylinder and Gas Modeling

TDM must compare unlike cylinder systems using gas volume, never PSI alone.

Cylinder models must explicitly represent relevant properties such as:
- Rated free-gas capacity
- Rated/working pressure
- Actual starting pressure
- Cylinder count
- Gas mix
- Configuration

Do not silently infer missing cylinder specifications.

Invalid, incomplete, or internally inconsistent cylinder configurations must be rejected.

## 11. Gas Physics Engine

`tdm-physics` owns physical gas calculations and unit-safe conversions.

Responsibilities include:
- Depth to ambient pressure
- RMV and ambient pressure to gas consumption
- Cylinder pressure to available gas volume
- Gas volume to cylinder pressure
- Distance and velocity to travel time
- Depth-segment gas consumption
- Freshwater/saltwater handling where applicable

The physics engine must not know Rule of Thirds or other gas-management policies.

Functions should be deterministic and independently testable.

## 12. Gas-Management Strategies

Gas-management rules must use a strategy/policy abstraction.

Initial implementation:
- Rule of Thirds

Future strategies may include:
- Modified Thirds
- Sixth-based rules
- Minimum Gas / Rock Bottom
- Stage/decompression gas policies
- CCR bailout
- User-defined conservative reserves

Do not scatter Rule-of-Thirds conditionals throughout the application.

A strategy receives validated planning context and returns explicit gas constraints/results.

## 13. Emergency Engine

Emergency calculations are a separate module.

The engine must support the defined catastrophic gas-loss scenario and determine whether available team gas can support the affected diver during exit while preserving required reserves.

Inputs may include:
- Affected diver
- Candidate donor
- Recipient RMV
- Donor RMV
- Configurable emergency/stress RMV
- Failure depth/location
- Exit profile
- Exit distance/time
- Depth changes
- Donor gas availability
- Required final reserve

The engine should be designed so multiple failure/donor combinations can be evaluated.

Emergency logic must not be hidden inside UI or generic MUX orchestration.

## 14. MUX Engine

`tdm-mux` is the authoritative team-planning engine.

Conceptual interface:

```text
TeamDivePlanRequest
        |
        v
     MUX Engine
        |
        v
TeamDivePlan
```

The MUX engine combines:
- Diver profiles
- Selected dive kits
- Dive profile
- Gas-management strategy
- Emergency scenario

It must determine:
- Maximum recommended penetration
- Team turn time
- Team turn distance
- Estimated runtime
- Individual diver turn pressure
- Individual expected gas remaining
- Required exit gas
- Required emergency gas
- Required reserve
- Limiting diver
- Limiting gas/equipment constraint

The team has one coordinated turn point even when individual turn pressures differ.

Results must include explanations of why the limiting constraint was reached.

## 15. Determinism and Calculation Provenance

Given identical validated inputs and the same calculation-engine version, TDM must produce identical results.

Saved calculations should eventually retain provenance such as:
- Engine version
- Algorithm/strategy version
- Input snapshot
- Emergency-model version
- Calculation timestamp
- Result snapshot

A future engine change must not silently redefine previously saved calculations.

## 16. Persistence

PostgreSQL is the authoritative persistent datastore.

Use relational modeling for structured entities and relationships.

Use database constraints where appropriate as defense in depth, while retaining primary validation in the domain/application layers.

Examples:
- Positive pressures/capacities
- Valid gas fractions
- O2 + He <= 1.0
- Required foreign-key relationships
- Uniqueness constraints where appropriate

Database migrations are source-controlled and AI-managed.

Do not use the database as the calculation engine.

## 17. API

Axum exposes the application through a versioned REST API.

Initial resource areas may include:

```text
/api/v1/divers
/api/v1/cylinders
/api/v1/kits
/api/v1/dive-plans
/api/v1/calculations/mux
```

API validation must reject missing, ambiguous, impossible, or inconsistent safety-critical input.

Calculation responses must include:
- Results
- Assumptions
- Warnings
- Limiting factors
- Relevant calculation metadata

The server-side Rust MUX engine is authoritative.

## 18. User Interface

Phase I uses React + TypeScript.

The UI is responsible for:
- Diver selection and management
- Kit selection and management
- Dive-profile entry
- Calculation submission
- Results presentation
- Warnings and validation feedback
- Explanation of limiting factors
- Reporting/export interfaces

The UI must not become an independent source of safety-critical formulas.

Client-side convenience calculations are permitted only when they cannot diverge from the authoritative server calculation and are clearly non-authoritative.

## 19. Future Mobile Support

Do not design Phase I around a hypothetical mobile implementation.

However, preserve clean calculation boundaries so a future mobile client can:
1. Call the same REST API, or
2. Reuse/compile the Rust calculation core for offline native execution if later justified.

There must remain one authoritative calculation model.

## 20. Testing

Every safety-critical calculation requires automated tests.

Required test layers:
- Unit tests for formulas and conversions
- Domain validation tests
- Gas-strategy tests
- Emergency-engine tests
- MUX scenario tests
- API integration tests
- Database integration/migration tests
- Regression/golden-scenario tests

Required scenario coverage includes the cases defined in `README.md`, including:
- Identical divers/cylinders
- Different RMVs
- Different cylinder sizes
- Different starting pressures
- Mixed AL80/LP85 teams
- High-consumption diver
- Low starting pressure
- Unequal team sizes
- Gas-sharing emergencies
- Limiting donor
- Limiting recipient
- Depth changes during exit
- Unit conversions
- Missing/invalid input
- Boundary conditions

Known reference scenarios must have deterministic expected results.

Where practical, critical formulas and scenarios should be independently cross-checked rather than testing an implementation solely against itself.

## 21. Safety Rules

The following are architectural invariants:

- Never silently guess missing values.
- Never silently assume cylinder capacity or working pressure.
- Never silently mix SAC and RMV without explicit conversion/normalization.
- Never compare unlike cylinders using PSI alone.
- Maintain explicit units.
- Preserve numerical precision internally.
- Round only for presentation.
- Reject physically impossible configurations.
- Display calculation assumptions.
- Keep calculation logic independent of UI.
- Document safety-critical formulas beside implementation.
- Every major calculation must be independently testable.
- A proposed dive that cannot satisfy the selected rules must produce a clear failure/warning, not a fabricated plan.

## 22. Observability and Failure Behavior

Errors must be explicit and diagnosable.

The application should distinguish:
- User input/validation errors
- Impossible dive-plan conditions
- Calculation failures
- Database failures
- Internal software defects

Safety-critical failures must fail closed: do not produce apparently valid planning numbers when required inputs or calculations are invalid.

Logging must not expose secrets or sensitive authentication material.

## 23. Security Baseline

Apply standard secure-development practices from the beginning:
- TLS in deployment
- Secrets outside source control
- Parameterized SQL
- Dependency auditing
- Input validation
- Least-privilege database/service accounts
- Authentication/authorization boundaries when user accounts are introduced
- Secure HTTP defaults and headers
- Reproducible builds and dependency lockfiles

Security controls must not be mixed into calculation formulas.

## 24. Engineering Priorities

When architectural choices conflict, use this priority order:

1. Calculation correctness and safety
2. Deterministic testability
3. Explicitness and auditability
4. Maintainability by AI engineering agents
5. Simplicity
6. Performance
7. Developer convenience

Prefer explicit, readable implementations over clever abstractions.

## 25. Initial Build Order

Recommended implementation sequence:

1. Rust workspace and project scaffolding
2. Domain types and unit system
3. Gas physics
4. Cylinder/gas-volume calculations
5. Rule-of-Thirds strategy
6. Emergency model
7. MUX engine
8. Deterministic reference scenarios and regression suite
9. PostgreSQL schema/persistence
10. Axum REST API
11. React UI
12. Deployment packaging
13. Reporting/export

Do not begin by building a visually complete UI around unverified placeholder calculations.

## 26. Governing Principle

TDM must answer:

> **What can THIS TEAM safely do with THIS equipment on THIS dive?**

Every diver and gas supply is modeled individually.

The MUX engine combines those individual constraints into one conservative coordinated team plan.

**The weakest constraint controls the dive.**
