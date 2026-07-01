# Agile Methodology

Comprehensive study guide for understanding the core principles, practices, and estimations of Agile project management in modern software development.

---

## 1. What is Agile?

**Agile** is a mindset and an iterative approach to software development and project management that helps teams deliver value to their customers faster and with fewer headaches. Instead of betting everything on a single "big-bang" launch, an Agile team delivers work in small, consumable increments.

### The Agile Manifesto (4 Core Values)
Created in 2001 by 17 software developers, the manifesto states:
1. **Individuals and interactions** over processes and tools.
2. **Working software** over comprehensive documentation.
3. **Customer collaboration** over contract negotiation.
4. **Responding to change** over following a plan.

---

## 2. Agile Estimations & Metrics

Estimating complexity and tracking team speed are central to Agile delivery.

### A. Story Points & Planning Poker
- **Story Points**: A relative unit of measure used to express the overall effort, complexity, and risk associated with implementing a product backlog item. Points are not hours; they represent **complexity relative to a baseline task**.
- **Fibonacci Sequence (`1, 2, 3, 5, 8, 13, 21`)**: Teams use Fibonacci numbers for estimations because:
  - Larger numbers represent higher uncertainty.
  - It forces discrimination (it is easy to argue 7 vs 8 hours, but choosing 5 vs 8 story points clearly highlights a jump in complexity).
- **Planning Poker**: A collaborative estimation technique. Team members hold estimation cards, discuss a user story, and simultaneously reveal their cards to avoid anchoring bias. If estimates differ widely, the outliers discuss their reasoning before re-voting.

### B. Velocity vs. Capacity
- **Velocity**: The average amount of work (expressed in story points) a team completes during a single iteration (Sprint).
  - *Use*: Used for long-term forecasting (e.g., if a team's average velocity is 40 points and there are 120 points left in the backlog, the release will take ~3 sprints).
- **Capacity**: The maximum amount of work hours or points a team can realistically commit to in a specific upcoming Sprint, accounting for vacations, holidays, and meetings.

---

## 3. High-Impact Interview Questions & Answers

### Q1: Why do we estimate in Story Points instead of Hours?
**Answer**:
1. **Focuses on Complexity, Not Time**: Hours do not account for relative experience. A senior engineer might finish a task in 2 hours, whereas a junior might need 12 hours. However, the complexity of the task remains identical. Agreeing on a relative Story Point (e.g., 3 points) normalizes expectations.
2. **Accounts for Non-Coding Work**: Hour-based estimates often fail to account for communication, reviews, testing, deployment setup, and meetings. Story points capture the *complete end-to-end effort* (complexity + risk + uncertainty).
3. **Reduces Micromanagement and Anxiety**: Hour-based estimates turn into hard deadlines, causing developers to rush code, accumulate technical debt, or inflate estimates defensively. Story points focus the team on relative sizing and predictable sprint capacity.

### Q2: How do you handle and pay down Technical Debt in an Agile team?
**Answer**:
Technical debt is inevitable as features are shipped quickly. If left ignored, velocity slows down to a crawl.
- **Handling Strategy**:
  1. **Visibility**: Always track technical debt as concrete refactoring tickets in the product backlog (e.g., in Jira), categorized and prioritized based on impact.
  2. **The 20% Allocation Rule (Recommended)**: Agree with the Product Owner (PO) to allocate 15% to 20% of every sprint's capacity strictly for technical debt, bug fixes, and infrastructure maintenance.
  3. **Build-Quality Integration**: Implement automated linting, test coverage gates, and static analysis (SonarQube) in the CI/CD pipeline to prevent new debt from slipping in unnoticed.

### Q3: What do you do when a Product Owner repeatedly asks to add "urgent" tasks in the middle of an active Sprint?
**Answer**:
In Scrum, the Sprint backlog is locked during the sprint to guarantee focus. However, real-world emergencies happen.
- **Action Plan**:
  1. **Impact Analysis**: Ask the PO to clarify the business urgency. Evaluate the story points of the new task.
  2. **The "Trade-off" Negotiation**: Explain that team capacity is fixed. If the new task is added, a task of equal story points **must be removed** from the current Sprint back to the Product Backlog to preserve the team's focus and velocity.
  3. **Aborting the Sprint**: If the change is so massive that it renders the current Sprint Goal completely obsolete, recommend aborting the Sprint entirely and starting a new Sprint planning session (high organizational cost).
