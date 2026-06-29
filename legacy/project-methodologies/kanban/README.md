# Kanban Framework

Comprehensive study guide for understanding Kanban principles, pull systems, Work In Progress (WIP) limits, flow metrics, and continuous optimization in modern software delivery.

---

## 1. What is Kanban?

**Kanban** (Japanese for "signboard" or "billboard") is a highly popular lean, visual framework used to implement Agile and Lean software development. It emphasizes visual management, continuous flow, and maximizing efficiency by limiting work in progress. Originating from the Toyota Production System in the 1940s, it has been widely adopted by software engineering teams.

### Five Core Principles of Kanban
1. **Visualize the Workflow**: Create a physical or digital board (columns representing steps like: *Backlog*, *In Progress*, *Peer Review*, *Testing*, *Done*) to make the flow of work visible.
2. **Limit Work in Progress (WIP)**: Set maximum card limits on active columns to prevent overloading team capacity.
3. **Manage and Optimize Flow**: Track progress, analyze cycle time, and resolve blockages to keep work moving smoothly.
4. **Make Process Policies Explicit**: Clearly define rules for when an item can move to the next stage (e.g., Definition of Ready, Definition of Done).
5. **Continuous Improvement (Kaizen)**: Use empirical data and metrics to continuously optimize team processes collaboratively.

---

## 2. Work In Progress (WIP) Limits & Key Metrics

### A. WIP Limits
A WIP limit is a hardcap on the maximum number of task cards allowed in a column concurrently:
- *Example*: If the `In Progress` column has a WIP limit of `3`, and there are already 3 cards in it, **no new card can be pulled into In Progress** until one of the existing cards is moved to the next column (`Peer Review`).
- *Goal*: Eliminates multi-tasking and context switching, forcing the team to focus on *finishing* open tasks rather than *starting* new ones ("Stop starting, start finishing").

### B. Core Flow Metrics

```
[Task Added to Backlog] ──(Lead Time Starts)──> [In Progress] ──(Cycle Time Starts)──> [Completed]
           |                                          |                                   |
           |<───────────────────────── Lead Time ────>|<─────────── Cycle Time ──────────>|
```

- **Lead Time**: The total time elapsed from when a task is first created/requested by a customer until it is fully delivered.
- **Cycle Time**: The time elapsed from when active work actually begins on the task until it is completed. **Minimizing cycle time is the primary goal of Kanban**.
- **Throughput**: The number of completed tasks delivered per unit of time (e.g., 15 cards per week).
- **Little's Law**: In queueing theory, Little's Law states:
  $$\text{Cycle Time} = \frac{\text{Work in Progress}}{\text{Throughput}}$$
  Reducing WIP mathematically decreases the average Cycle Time, meaning tasks get delivered to customers faster.

---

## 3. Advanced Flow Visualization: CFD

A **Cumulative Flow Diagram (CFD)** is a graphical chart showing the cumulative number of work items in each stage of the workflow over time.

- **Identifying Bottlenecks**: If a column's band on the CFD is widening vertically over time, it indicates that input to that stage is faster than output, marking a growing bottleneck.
- **Predicting Lead Time**: The horizontal distance between the top-most line (backlog) and the bottom-most line (completed) indicates the average Lead Time of the team.

---

## 4. High-Impact Interview Questions & Answers

### Q1: What is "Backpressure" on a Kanban board, and how do WIP limits solve it?
**Answer**:
- **The Problem (Bottlenecks)**: Without WIP limits, if the `Testing` phase slows down, developers will keep pulling tasks into `In Progress` and pushing them into `Ready for Testing`. The testing column overflows, creating high stress, context switching, and a bottleneck that halts deliveries.
- **How WIP Limits Solve It**:
  1. If `Testing` reaches its WIP limit, testers cannot accept new items.
  2. Developers cannot move completed code from `In Progress` to `Testing` because the next column is blocked.
  3. Developers are forced to **stop coding new features**, walk over, and assist the testers in clearing the testing bottleneck (a practice called **Swarming**). This balances the pipeline, optimizes global flow, and maintains high quality.

### Q2: Compare Scrum vs. Kanban across major operational dimensions. When is Kanban the superior choice?
**Answer**:

| Dimension | Scrum | Kanban |
| :--- | :--- | :--- |
| **Cadence** | Time-boxed Sprints (1-4 weeks) | Continuous flow, no fixed cycles |
| **Roles** | Product Owner, Scrum Master, Developers | No standard predefined roles (optional) |
| **Estimation** | Story Points (Velocity) | Optional (focuses on Cycle Time/Throughput) |
| **Changes** | Locked during Sprint (no mid-sprint changes) | Dynamic (can change backlog priorities anytime) |
| **Primary Metric** | Velocity (points per sprint) | Cycle Time, Lead Time, Throughput |

- **When Kanban is Superior**:
  - **Support and Operations Teams**: Teams handling customer support tickets, bug triage, or sysadmin tasks where priorities change hourly and time-boxed sprints are impossible.
  - **Mature Product Teams**: Highly cohesive teams that can deliver value continuously without needing the overhead of Scrum ceremonies and sprint planning.

