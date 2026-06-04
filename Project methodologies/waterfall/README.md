# Waterfall Model

Comprehensive study guide for understanding the sequential Waterfall model, SDLC phases, gate reviews, and comparison with Agile.

---

## 1. What is the Waterfall Model?

The **Waterfall Model** is a linear, sequential approach to software development, where project phases flow steadily downwards like a physical waterfall. Each phase must be completed 100% before the next phase begins.

### The Six SDLC Phases of Waterfall
1. **Requirements Gathering & Analysis**: All possible requirements are captured upfront in a comprehensive Product Requirements Document (PRD).
2. **System Design**: Architects design the hardware, software, and logical system architecture based on the PRD.
3. **Implementation (Coding)**: Developers write code according to the system designs.
4. **Integration & Testing**: The system is compiled, integrated, and thoroughly tested for bugs and defects.
5. **Deployment (Delivery)**: The completed software is installed and released to the customer.
6. **Maintenance**: Post-release support, bug fixes, and minor upgrades.

---

## 2. Key Characteristics & Gates

- **Gate Reviews (Phase Gates)**: At the end of each phase, a formal audit or gate review occurs. Stakeholders inspect the phase deliverables (e.g., the System Design document) and must formally sign off before the next phase is allowed to start.
- **Upfront Planning**: There is little to no room for changes once a phase ends. Altering a requirement during the Implementation phase is extremely costly because it requires rewriting design, requirements, and contract documents.

---

## 3. Comparing Waterfall vs. Agile

| Dimension | Waterfall | Agile |
| :--- | :--- | :--- |
| **Workflow** | Sequential / Linear | Iterative / Incremental |
| **Scope** | Fixed upfront | Dynamic / Flexible |
| **Customer Feedback**| Very late (at the end of SDLC) | Constant (at the end of every Sprint) |
| **Risk Management** | High risk (flaws discovered late) | Low risk (continuous testing/reviews) |
| **Contracts** | Fixed-scope / Fixed-price contracts | Time-and-materials / Value-based |

---

## 4. High-Impact Interview Questions & Answers

### Q1: When is the Waterfall model superior to Agile? Provide actual industry examples.
**Answer**:
Waterfall is superior in environments where requirements are completely stable, highly regulated, and any failure would be catastrophic:
1. **Safety-Critical and Embedded Systems**: Aerospace software (e.g., flight controller software), medical devices (e.g., pacemakers), and nuclear power grid controls. Upfront mathematical proofs, rigid safety requirements, and extensive verification are mandatory.
2. **Hardware-Software Integrations**: Building a physical smartphone or automobile. You cannot easily change the physical motherboard layout (hardware) in a two-week sprint, so requirements must be locked down early.
3. **Highly Bounded Government/Enterprise Projects**: Where contracts mandate a strict fixed-price, fixed-scope, and fixed-deadline delivery model.

### Q2: What is the "V-Model" in Waterfall, and how does it address quality assurance?
**Answer**:
The **V-Model (Validation and Verification Model)** is an extension of the Waterfall model that maps testing phases directly to corresponding development phases.
- **Structure**: The SDLC bends into a "V" shape. The left side of the V is the **Verification** (design/coding) phase, and the right side is the **Validation** (testing) phase.
- **Mapping**:
  - *Requirements Analysis* maps to *User Acceptance Testing (UAT)*.
  - *High-Level Design* maps to *System Testing*.
  - *Low-Level Design* maps to *Integration Testing*.
  - *Coding* maps to *Unit Testing* (at the tip of the V).
- **Benefit**: Ensures that test plans are designed and written *at the same time* as their corresponding development phase documents, catching flaws much earlier in the SDLC.
