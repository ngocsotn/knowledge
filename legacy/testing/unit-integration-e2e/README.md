# Test Classifications: Unit, Integration, & E2E

Comprehensive interview study guide covering the levels of software testing, the Testing Pyramid vs. Testing Trophy, test doubles, and automation.

---

## 1. Levels of Testing

| Level | What it Tests | Target Scope | Execution Speed | Cost to Build/Run |
| :--- | :--- | :--- | :--- | :--- |
| **Unit Testing** | Individual functions, methods, or isolated classes. | Single function (mocks all IO/external calls) | Extremely Fast (ms) | Low |
| **Integration Testing**| Interaction between multiple components or modules. | API + DB, Service + Cache | Medium (seconds) | Medium |
| **End-to-End (E2E)** | Complete, real user journeys across the entire stack.| Frontend + API + DB + Third-Party | Slow (minutes) | High |

---

## 2. Testing Models: Pyramid vs. Trophy

### 1. The Testing Pyramid
* **Concept:** Recommends a high volume of Unit tests at the bottom, a moderate volume of Integration tests in the middle, and a very small volume of E2E tests at the top.
* **Focus:** High speed, isolated testing, cheap development, fast feedback loops.

### 2. The Testing Trophy (Kent C. Dodds)
* **Concept:** Recommends focusing **primarily on Integration tests** (the wide belly of the trophy), with a healthy but smaller amount of Unit and E2E tests.
* **Focus:** "Write tests, not too many, mostly integration." It argues that integration tests provide the highest return-on-investment (ROI)—testing that components actually work together correctly while avoiding the brittle nature of mocking too much internal implementation details in unit tests.

---

## 3. Test Doubles (Mocks, Stubs, & Spies)

To test code in isolation, we replace real network or filesystem IO with **Test Doubles**:

* **Stub:** Returns hardcoded, static values instantly to satisfy a dependency's interface (e.g., a Stub DB that always returns a fake user list without querying).
* **Mock:** Focuses on **behavior verification**. It has pre-programmed expectations and verifies that specific methods were called with exact arguments (e.g., asserting that `sendEmail()` was called exactly once with `'user@test.com'`).
* **Spy:** Wraps a real object to record tracking metrics (like call counts or passed arguments) while preserving its original, live implementation.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: When should you Mock a dependency, and when should you use a real database/service in integration tests?
* **Answer:** Mocking should be used to isolate your system from **third-party APIs** (e.g., Stripe Payment, SendGrid Email) that you do not own, as real network calls to these are slow, unreliable, and cost money. However, for internal components that you *do* own (e.g., your database schema, redis cache), you should prefer testing against a real database instance (typically run inside a lightweight Docker container like Testcontainers) during integration tests. Mocking your database hides schema syntax errors, transaction bugs, or query logic issues, reducing test reliability.

### Q2: What are the main drawbacks of focusing exclusively on End-to-End (E2E) testing?
* **Answer:** While E2E tests provide the highest confidence (as they mimic real user behaviors over a live browser), they suffer from three major drawbacks:
  1. **Sluggish Execution Speed:** Running browser instances (like Playwright or Selenium) and loading real pages takes minutes, slowing down CI/CD pipelines.
  2. **High Flakiness:** E2E tests are prone to random failures (false negatives) caused by network blips, minor animation delays, or slow rendering.
  3. **Vague Root-Cause Analysis:** If an E2E button click fails, the test report simply says "timeout." It won't tell you if the bug is in the CSS layout, backend API, DB transaction, or a third-party script.

### Q3: What is "Regression Testing" and why is it crucial in agile software development?
* **Answer:** **Regression Testing** is the practice of running your existing test suite automatically after code modifications, refactorings, or package updates to prove that the changes did not introduce new bugs into previously stable areas of the application. It acts as a safety net, allowing developers to deploy code multiple times a day with high confidence, knowing that any broken legacy functionality will be instantly flagged by the automated test runner in the pipeline.
