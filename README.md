# Tech-Diver MUX (TDM)

## Multi-Diver Technical Dive Planning & Gas Management

**Tech-Diver MUX (TDM)** is a technical-diving planning application designed to calculate a coordinated dive plan for a team of divers who may have different breathing rates, swimming speeds, cylinder configurations, starting pressures, and gas capacities.

The central concept is the **MUX Engine**:

> Instead of planning gas for an "average diver," TDM calculates what **this specific team**, using **this specific equipment**, can safely accomplish on **this specific dive**.

The limiting member of the team determines the allowable dive.

---

## Core Workflow

**Diver → Dive Kit → Dive Profile → MUX Calculation**

### 1. Select Divers

Each diver maintains a reusable profile containing information such as:

- Name
- Certifications and technical-diving qualifications
- SAC/RMV
- Average fin/swimming speed
- Preferred units
- Other performance information used by the planning engine

Each diver is calculated independently. TDM never assumes every team member consumes gas or travels at the same rate.

### 2. Select a Dive Kit

Each diver selects a saved equipment configuration for the planned dive.

Examples:

- Single AL80
- 2× AL80 Sidemount
- 2× LP85 Sidemount
- Double HP100
- AL80 + Stage

A Dive Kit may define:

- Configuration
- Number of cylinders
- Cylinder type
- Rated gas capacity
- Working pressure
- Actual starting pressure
- Gas mixture
- Nitrox percentage
- Helium percentage
- Stage/decompression cylinders
- Reserve parameters

TDM converts cylinder pressure into **actual available gas volume** before comparing divers.

PSI alone is never treated as equivalent between unlike cylinders.

### 3. Create the Dive Profile

The user defines the proposed dive.

Parameters may include:

- Maximum depth
- Expected average depth
- Planned penetration
- Travel speed
- Descent rate
- Ascent rate
- Gas-management strategy
- Environmental assumptions
- Decompression assumptions
- Gas switches
- Emergency assumptions

The dive profile represents what the team intends to accomplish.

The MUX Engine determines whether the team has sufficient resources to accomplish it.

---

## The MUX Engine

The MUX Engine combines:

```text
Diver Profile
      +
Dive Kit
      +
Dive Profile
      +
Gas Management Strategy
      +
Emergency Scenario
      =
TEAM DIVE PLAN
```

Consider two divers:

```text
DIVER 1
RMV: 0.80 cu ft/min
Cylinders: 2× AL80
Starting pressure: 3000 PSI

DIVER 2
RMV: 1.10 cu ft/min
Cylinders: 2× LP85
Starting pressure: 2640 PSI
```

TDM does not simply assign both divers the same turn pressure.

Instead, it determines each diver's:

- Actual gas volume
- Consumption at depth
- Travel consumption
- Required exit gas
- Emergency gas requirement
- Required reserve
- Individual turn pressure

The resulting pressures may be different.

Example:

```text
Maximum Recommended Penetration: 1,420 ft

Team Turn:
Distance: 1,420 ft
Time: 24 min

Diver 1 Turn Pressure: 2,080 PSI
Diver 2 Turn Pressure: 2,340 PSI

Limiting Diver: Diver 2
```

The team has **one coordinated turn point**, even though individual divers may have different SPG pressures at that point.

---

## Gas Management

Initial development focuses on **Rule of Thirds**.

The architecture should allow additional gas-management strategies to be added later, including:

- Rule of Thirds
- Modified Thirds
- Sixth-based penetration rules
- Minimum Gas / Rock Bottom
- Stage gas planning
- Decompression gas planning
- CCR bailout planning
- User-defined reserve strategies

Gas-management strategies should remain modular rather than being embedded throughout the application.

---

## Emergency Gas Planning

A primary purpose of TDM is determining whether the team can manage a catastrophic gas-loss event at the planned maximum penetration.

The MUX Engine evaluates scenarios such as:

```text
Diver loses usable breathing gas
            ↓
Another diver becomes donor
            ↓
Both divers exit using donor's gas
            ↓
Gas consumption calculated through exit
            ↓
Required reserve preserved
```

Emergency calculations may account for:

- Recipient RMV
- Donor RMV
- Configurable emergency/stress RMV
- Ambient pressure
- Exit distance
- Exit speed
- Depth changes
- Available donor gas
- Remaining team gas
- Required final reserve

The objective is determining whether the **team can manage the defined failure at the worst planned point of the dive.**

---

## Calculation Output

Pressing **CALCULATE** should provide a team plan containing information such as:

### Team Plan

- Maximum recommended penetration
- Estimated penetration time
- Estimated total runtime
- Team turn time
- Team turn distance

### Diver Plan

- Individual turn pressure
- Expected gas consumption
- Expected gas remaining
- Required reserve

### Emergency Plan

- Required emergency gas
- Gas donor
- Gas recipient
- Emergency exit requirement

### Limiting Factor

- Limiting diver
- Limiting gas supply
- Limiting equipment configuration
- Reason the limit was reached

TDM should explain **why** a particular diver became the limiting member of the team.

---

## Gas Physics

All calculations must use explicit units and actual gas volume.

Gas consumption at depth must account for ambient pressure.

Conceptually:

```text
Gas Consumption = RMV × Ambient Pressure × Time
```

Cylinder calculations must account for:

- Rated capacity
- Working pressure
- Actual starting pressure
- Cylinder count

Internal calculations should maintain adequate precision.

Rounding should occur only when values are presented to the user.

---

## Safety-Critical Software

Technical-diving gas calculations can affect real-world dive planning.

TDM therefore treats its calculation engine as safety-critical software.

Development principles include:

- Never silently guess missing values.
- Never silently assume cylinder specifications.
- Never compare unlike cylinders using PSI alone.
- Never silently substitute SAC for RMV.
- Validate inputs aggressively.
- Reject impossible configurations.
- Make assumptions visible.
- Keep gas calculations independent from UI code.
- Document formulas.
- Test calculations independently.
- Maintain deterministic regression tests.

TDM is a **planning tool**, not a substitute for diver training, judgment, established procedures, environmental assessment, or direct verification of a dive plan.

Divers remain responsible for independently validating their gas plan before entering the water.

---

## Architecture

TDM should maintain separation between major functional areas:

```text
Diver Management
Equipment / Cylinder Database
Dive Kit Management
Dive Profile Management
Gas Physics Engine
Gas Management Strategies
MUX Team Engine
Emergency Gas Engine
Decompression Engine
Persistence / Database
User Interface
Reporting / Export
```

Calculation modules should not depend upon UI components.

This allows calculations to be independently tested and eventually reused by different interfaces.

---

## Testing Philosophy

Every safety-critical calculation should have automated tests.

Important test scenarios include:

- Identical divers with identical cylinders
- Different RMV values
- Different cylinder capacities
- Different starting pressures
- AL80 and LP85 combinations
- High-consumption diver
- Low starting pressure
- Multiple team sizes
- Gas-sharing failures
- Limiting donor
- Limiting recipient
- Depth changes during exit
- Unit conversions
- Invalid inputs
- Missing data
- Boundary conditions

Known dive scenarios should produce deterministic expected results.

A change to the calculation engine must not silently change previously validated results.

---

## Future Capabilities

The architecture should permit future development of features such as:

- Stage cylinders
- Decompression gases
- Trimix
- Multi-level profiles
- Variable-depth cave profiles
- CCR
- CCR bailout
- DPV travel
- Dive-site profiles
- Saved cave/wreck routes
- Actual-vs-planned dive analysis
- Dive computer data import
- Team optimization
- Dive reports
- Mobile interfaces

These capabilities should extend the existing calculation model rather than require replacement of the MUX architecture.

---

## Design Principle

TDM revolves around one fundamental question:

> **What can THIS TEAM safely do with THIS equipment on THIS dive?**

Every diver is modeled individually.

Every gas supply is modeled individually.

Every relevant constraint is calculated.

The constraints are then multiplexed into a single coordinated team plan.

**The weakest constraint controls the dive.**

That is the **MUX** in **Tech-Diver MUX**.
