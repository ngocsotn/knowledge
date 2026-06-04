# TDD & BDD Methodologies

Comprehensive interview study guide covering Test-Driven Development (TDD), Behavior-Driven Development (BDD), red-green-refactor cycles, and domain alignment.

---

## 1. TDD (Test-Driven Development)

**TDD** is an iterative software development process where you write tests *before* writing the actual production code.

### The Red-Green-Refactor Cycle
```
              ┌──────────────┐
              │     RED:     │
              │  Write test  │◄────────┐
              │  that fails  │         │
              └──────┬───────┘         │
                     │                 │
                     ▼                 │
              ┌──────────────┐         │
              │    GREEN:    │         │
              │  Write bare  │         │
              │  min code to │         │
              │  make it pass│         │
              └──────┬───────┘         │
                     │                 │
                     ▼                 │
              ┌──────────────┐         │
              │  REFACTOR:   │         │
              │ Clean up and │─────────┘
              │ optimize code│
              └──────────────┘
```

* **Pros:** Guarantees 100% test coverage, prevents over-engineering (you write strictly the bare minimum code to make the test pass), forces modular class designs with high testability.
* **Cons:** Requires strong discipline, can slow down early prototyping, and may lead to brittle tests if tied too closely to implementation details rather than business behavior.

---

## 2. BDD (Behavior-Driven Development)

**BDD** is an extension of TDD that focuses on testing **business behavior and user requirements** from the user's perspective, using a ubiquitous, human-readable language.

### Given-When-Then syntax (Gherkin/Cucumber)
BDD scenarios are written in structured, non-technical plain text so that developers, QA, and business stakeholders can collaborate and agree on requirements:

```gherkin
Feature: User Login
  Scenario: Successful login with valid credentials
    Given the user is on the login page
    When the user enters a valid username and password
    And clicks the submit button
    Then the user should be redirected to the dashboard
    And see a welcoming message "Welcome back!"
```

---

## 3. Comparing TDD vs. BDD

| Attribute | TDD | BDD |
| :--- | :--- | :--- |
| **Focus** | How code is implemented correctly (Developer-focused). | How the system behaves for the user (Business-focused). |
| **Audience** | Developers | Developers, QA, Product Managers, Stakeholders. |
| **Tools** | Jest, JUnit, Vitest, Go `testing` | Cucumber, Behave, Cypress, Playwright. |
| **Output** | Technical function tests, mock assertions. | Human-readable user stories and specs. |

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: Why do some developers complain that TDD slows them down, and how do you address this critique?
* **Answer:** TDD can feel slow initially because it forces developers to design interfaces, write assertions, and think through edge cases before writing a single line of production code. However, TDD **saves time in the long run** by eliminating the expensive "debug-and-fix" cycle. In non-TDD workflows, developers often write complex code, run it, find bugs manually, and spend hours debugging. With TDD, bugs are detected in seconds at the terminal. It also serves as living documentation, reducing onboarding time for new engineers.

### Q2: What is the main trap of TDD, and how do you write tests that do not break during refactoring?
* **Answer:** The main trap of TDD is **testing implementation details rather than business outcomes**. For example, if you write unit tests that assert exactly how many times an internal helper function was called, or mock internal class variables, your tests are brittle. The moment you refactor or clean up code, these tests will fail, even if the final business output remains completely correct. To prevent this, always test the **public API contract** (inputs and outputs) of your modules, treating the internal implementation as a black box.

### Q3: How does BDD bridge the communication gap between technical and non-technical team members?
* **Answer:** BDD uses Gherkin's structured **Given-When-Then syntax**, which is written in standard plain English. This allows non-technical Product Managers, Designers, and Business Analysts to write and review acceptance criteria directly. Since BDD frameworks (like Cucumber) parse these plain-text files and map them to physical test scripts, the business requirements *become* the actual automated test cases. This guarantees that developers build exactly what the business specified, preventing feature drift and requirement misunderstandings.
