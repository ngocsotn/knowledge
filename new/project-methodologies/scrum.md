# Scrum Framework

Comprehensive study guide for understanding Scrum roles, ceremonies, artifacts, and empiricism in agile software delivery.

---

## 1. What is Scrum?

**Scrum** is a lightweight framework designed to help teams collaborate and solve complex adaptive problems, while productively and creatively delivering products of the highest possible value.

### Three Pillars of Empiricism
Scrum is founded on empirical process control, which relies on:
1. **Transparency**: The process and work must be visible to those performing the work as well as those receiving it.
2. **Inspection**: Scrum artifacts and progress toward goals must be inspected frequently to detect undesirable variances.
3. **Adaptation**: If aspects of a process or product deviate outside acceptable limits, the process or materials must be adjusted immediately.

---

## 2. Scrum Roles, Ceremonies, & Artifacts

### A. The Three Scrum Roles
- **Product Owner (PO)**: Maximizes the value of the product. Owns the Product Backlog, prioritizing stories based on customer feedback and business value.
- **Scrum Master (SM)**: Facilitates the Scrum process, coaches the team on Scrum theory, and removes impediments that block developers.
- **Developers**: The cross-functional team of professionals (engineers, designers, testers) who build the working Product Increment.

### B. The Five Scrum Ceremonies

| Ceremony | Goal | Frequency / Duration |
| :--- | :--- | :--- |
| **Sprint Planning** | Define the Sprint Goal and select backlog items to commit to for the upcoming Sprint. | Once per Sprint (typically 2-4 hours) |
| **Daily Standup** | Inspect progress toward the Sprint Goal and synchronize daily plans. (What did I do yesterday? What will I do today? Am I blocked?) | Daily (Strictly 15 minutes) |
| **Sprint Review** | Demo the working product increment to stakeholders and gather feedback. | End of Sprint (typically 1-2 hours) |
| **Sprint Retrospective**| Inspect the team's processes, collaboration, and tools, and create a concrete action plan for improvement. | End of Sprint (typically 1-1.5 hours) |
| **The Sprint** | The core container event. A time-boxed period during which a usable, inspectable increment is built. | 1 to 4 weeks (Fixed duration) |

### C. The Three Scrum Artifacts
- **Product Backlog**: The dynamic, ordered list of everything needed in the product.
- **Sprint Backlog**: The set of product backlog items selected for the Sprint, plus a plan for delivering the Increment and achieving the Sprint Goal.
- **Increment**: The concrete step toward the Product Goal. Must meet the **Definition of Done (DoD)** to be considered usable.

---

## 3. High-Impact Interview Questions & Answers

### Q1: What is the difference between a "Definition of Done" (DoD) and "Acceptance Criteria"?
**Answer**:
- **Acceptance Criteria**:
  - Unique to each specific User Story. It defines the functional requirements and business rules that a particular feature must satisfy to be accepted by the Product Owner.
  - *Example*: For a login page, AC might be: *"Must show an error for invalid passwords and lock the account after 5 failed attempts."*
- **Definition of Done (DoD)**:
  - A global, standardized checklist applied to **every single backlog item** in the sprint. It guarantees quality consistency across the entire product.
  - *Example*: DoD might be: *"Unit test coverage is $\ge$ 80%, code compiles without warnings, security scans pass, code is reviewed by a peer, and deployment succeeds in the staging environment."*

### Q2: What is the role of a Scrum Master? Are they a Project Manager?
**Answer**:
No, a Scrum Master is **not** a Project Manager.
- **Project Manager**: Typically commands and controls, assigns tasks, manages budgets, owns deadlines, and is accountable for the project plan.
- **Scrum Master**: A **Servant Leader** and facilitator. They:
  1. Do not assign tasks (the team is self-organizing).
  2. Coach the team and organization on Agile and Scrum.
  3. Facilitate Scrum ceremonies to keep them within time-boxes.
  4. Actively remove organizational and technical road-blocks (impediments) that stop developers from working.

### Q3: How do you handle a "Carry-over" (unfinished user stories) at the end of a Sprint?
**Answer**:
1. **Analyze the Cause (Inspection)**: In the Sprint Retrospective, inspect *why* the story was not completed. Was it due to poor estimation, external dependency blockages, or scope creep?
2. **Re-evaluate and Split**: Split the completed portion of the work from the remaining work.
3. **Move to Product Backlog**: The unfinished portion automatically moves back to the Product Backlog. **Never automatically carry it over to the next Sprint**.
4. **PO Prioritization**: The Product Owner must re-prioritize the remaining ticket. If it is still high-priority, it is estimated and planned for the next sprint. Only count completed story points in the current sprint velocity; unfinished points count as 0.
