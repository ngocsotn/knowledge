# Agent Skills Playbook: Reuse, Verify, Grill

A skill is a reusable, bounded procedure. It is not a free-form prompt: it defines when to act, what may be used, how success is proved, and when to stop.

In a developing app, distinguish between three things:

```text
skill      = how to perform a bounded task
capability = existing code, package, library, component, service, or other reusable implementation
tool       = mechanism used to inspect, change, run, or verify work
```

The default workflow is:

```text
request -> understand -> inspect -> reuse -> plan -> execute -> verify -> deliver
                              |
                        consequential action?
                              |
                         grill-me gate
```

## Reuse Existing Capabilities

Prefer existing implementation capabilities before creating new ones.

Search in this order, and stop once a suitable reusable capability has been identified:

```text
existing package or library
    -> existing project code, module, component, or service
    -> existing project utility, script, test helper, or template
    -> approved external tool or service
    -> new implementation
```

Inspect the relevant codebase and dependency context before deciding that something must be created from scratch.

Reuse when the candidate's scope, assumptions, interfaces, dependencies, runtime constraints, and security model are compatible with the task.

An existing capability does not need to be an exact match. Prefer, in order:

```text
reuse directly
    -> compose existing capabilities
    -> extend existing code
    -> adapt existing code with a small boundary layer
    -> create new implementation
```

Do not duplicate functionality that already exists in the project or its dependencies without a clear reason.

When no suitable reusable capability exists, create the new implementation that closes the explicit gap. Create a new skill or procedure only when the resulting workflow itself is expected to be reusable.

### Reuse Decision

```text
Need capability
      |
      v
Is there a suitable package or library?
      | yes
      v
Can it be used safely within current constraints?
      | yes --------------------> reuse
      | no
      v
Is there existing project code that can be composed or extended?
      | yes
      v
Reuse, extend, or adapt existing code
      |
      no
      v
Is there an existing utility, script, test helper, template,
approved tool, or service that provides the capability?
      | yes
      v
Reuse it
      |
      no
      v
Create the new implementation
```

## Grill-Me: A Risk Gate, Not a Ritual

Use `grill-me` before a consequential action or when making a completion claim that depends on material assumptions or incomplete evidence. Skip it for routine, low-impact reads and changes.

```text
proposed action: deploy authentication change
question 1: is rollback tested?        -> no
question 2: is approval in scope?      -> no
question 3: what evidence is missing?  -> rollback test and approval
decision: stop; request the missing evidence and approval
```

Time-box the gate. Each objection must point to a real missing check, assumption, boundary, or authority problem.

For application development, relevant risks include:

```text
security or data impact
breaking API or compatibility changes
production or deployment impact
irreversible migrations
missing or insufficient verification
unverified assumptions about dependencies
out-of-scope authority or approval
```

The grill gate is not a brainstorming exercise. Its purpose is to identify whether the agent has enough evidence and authority to proceed.

## A Good Skill in One Screen

| Field           | Example                                                                                               |
| --------------- | ----------------------------------------------------------------------------------------------------- |
| Trigger         | "Review a pull request."                                                                              |
| Input           | Diff, repository conventions, acceptance criteria, and relevant existing code.                        |
| Reuse path      | Existing package/library, project module/component, test utility, then approved tool.                 |
| Allowed actions | Read code, inspect dependencies, modify scoped code, and run relevant checks.                         |
| Verification    | Evidence appropriate to the change confirms the acceptance criteria, and unresolved risks are stated. |
| Stop condition  | Evidence is sufficient, a blocker is found, or approval is required.                                  |

## Verification

Verification should match the type and risk of the change. Tests are one form of evidence, not the only form.

Use the evidence appropriate to the task, such as:

```text
unit or integration tests
type checking or static analysis
build or packaging checks
linting or formatting checks
runtime or smoke checks
API or contract validation
database migration validation
security checks
manual acceptance checks
```

A completion claim should be based on observed evidence rather than on the fact that an implementation was written.

## The Standard Loop

```text
understand
   -> inspect existing capabilities
   -> identify reuse path
   -> plan
   -> validate assumptions
   -> act
   -> observe
   -> verify
   -> deliver
```

Use progressive disclosure: load only the instructions, references, dependencies, and code relevant to the current task and selected reuse path. Keep unrelated rules and implementation details from obscuring the work.

## Core Principles

```text
reuse before recreate
compose before duplicate
extend before fork
verify before claim
grill before consequential action
stop when evidence or authority is insufficient
```

---

## Interview Questions & High-Impact Answers

### Q1: What is the difference between a skill, a capability, and a tool?
* **Answer:** A **skill** is a bounded procedure. It says when to act, what may be used, how success is proved, and when to stop. A **capability** is an existing implementation the skill draws on, such as a package, library, module, component, or service. A **tool** is the mechanism used to inspect, change, run, or verify work, such as file reads, shell access, test runners, or code-graph queries. They fail in different ways. A bad skill produces the wrong workflow, a missing capability produces duplicated code, and a missing tool produces claims nobody can check. Mixing all three into one prompt is the usual cause of prompt sprawl.

### Q2: Why prefer reuse over new implementation, and when is reuse unsafe?
* **Answer:** Existing code already carries tested behavior, known interfaces, and an accepted security posture. Rewriting it duplicates maintenance work and brings back bugs that were already fixed. The search order is: existing package or library, then existing project code, then an existing utility, script, test helper, or template, then an approved external tool or service, and only then a new implementation. Reuse becomes unsafe when the candidate's **scope, assumptions, interfaces, dependencies, runtime constraints, or security model** do not fit the task. A common example is a library that assumes a single-tenant synchronous context being pulled into a multi-tenant async request path. When that happens, fall back through compose, extend, adapt behind a small boundary layer, and finally create new.

### Q3: A library covers 80% of a requirement. What should the agent do?
* **Answer:** An exact match is not required. Prefer, in order: reuse directly, compose existing capabilities, extend existing code, adapt with a small boundary layer, then create new. For an 80% match, adapt it behind a thin wrapper so the remaining 20% sits in one place that is easy to test and easy to delete once upstream catches up. Forking or reimplementing is the worst option because you quietly take on the maintenance cost of the 80% that already worked.

### Q4: What is a grill-me gate, and when should it not run?
* **Answer:** `grill-me` is a risk gate. Before a consequential action, or before claiming completion on incomplete evidence, the agent questions its own position: is rollback tested, is approval in scope, what evidence is missing. Every objection has to point at a real missing check, assumption, boundary, or authority problem. It should not run on routine, low-impact reads and changes. Running it on everything turns it into a ritual that adds latency and teaches reviewers to skip it. It is also not a brainstorming step. Its only job is to decide whether there is enough evidence and authority to proceed.

### Q5: Which risks should trigger the gate in application development?
* **Answer:** Security or data impact, breaking API or compatibility changes, production or deployment impact, irreversible migrations, missing or insufficient verification, unverified assumptions about dependencies, and actions outside the agent's approval scope. What these share is blast radius. Anything the agent cannot cheaply undo, or that commits someone else's system, deserves the gate. A simple test: if the recovery step would be "ask a human", ask before acting instead of after.

### Q6: Why is "the tests pass" not the same as verification?
* **Answer:** Verification should match the type and risk of the change. Tests are one form of evidence, not the definition of it. Depending on the change, the right evidence may be unit or integration tests, type checking or static analysis, build and packaging checks, lint or format checks, runtime or smoke checks, API or contract validation, migration validation, security checks, or manual acceptance. A config change with a green unit suite is still unverified if nothing exercised the config path, and a docs change needs no test run at all. The rule is that a completion claim rests on evidence you observed, not on the fact that code was written.

### Q7: What belongs in a good skill definition?
* **Answer:** Six fields that fit on one screen. **Trigger**, the situation that invokes it. **Input**, the context it needs, such as the diff, conventions, acceptance criteria, and relevant existing code. **Reuse path**, the ordered search for existing capabilities. **Allowed actions**, the explicit permission boundary. **Verification**, the evidence that proves the acceptance criteria and how unresolved risks get reported. **Stop condition**, meaning evidence is sufficient, a blocker was found, or approval is required. Each missing field has a predictable failure: no trigger and it fires at the wrong time, no allowed actions and it creeps out of scope, no stop condition and it runs until the context is gone.

### Q8: What is progressive disclosure and what problem does it solve?
* **Answer:** Load only the instructions, references, dependencies, and code that the current task and the chosen reuse path actually need. It fixes two problems at once. The first is context dilution, where the rules that matter get buried under rules that do not and end up ignored. The second is cost and latency, where every request pays for instructions it never uses. In practice the skill's entry point stays small and pulls in detailed references on demand instead of shipping every branch up front.

### Q9: How does the standard loop differ from "plan, then code"?
* **Answer:** The loop is understand, inspect existing capabilities, identify the reuse path, plan, validate assumptions, act, observe, verify, deliver. Two steps make the difference. Inspecting before planning means the plan is written against what the codebase actually contains, so it does not propose building something that already exists. Validating assumptions before acting means shaky premises get checked while they are still cheap to fix, instead of surfacing as failures after the code is written. Observe and verify are also separate: observing is reading what happened, verifying is confirming it meets the acceptance criteria.

### Q10: What are the core principles, and which one gets violated most?
* **Answer:** Reuse before recreate, compose before duplicate, extend before fork, verify before claim, grill before a consequential action, and stop when evidence or authority is insufficient. The one most often broken is **verify before claim**. An agent that has just written plausible code is strongly inclined to report success, because producing the implementation feels like finishing the task. The runner-up is **stop when evidence or authority is insufficient**, because stopping looks like failure unless the workflow explicitly treats "blocked, and here is what is missing" as a good outcome.

### Q11: How do you keep a skill from turning into a bloated general-purpose prompt?
* **Answer:** Three constraints. Keep it bounded, with a stated stop condition, so it cannot drift into open-ended work. Create a new skill only when the workflow itself is reusable, since one-off procedures belong in the request and otherwise the library fills with near-duplicates nobody can choose between. Use progressive disclosure, keeping detailed references in linked documents so the skill body stays a procedure rather than a manual. When a skill starts collecting conditional branches for unrelated situations, split it.

### Q12: A skill reports "done, implemented and working." What questions expose a weak claim?
* **Answer:** Ask what evidence was observed rather than what was written: which checks ran, against which code path, and what the output said. Ask what was assumed instead of verified, since dependency behavior, environment config, and data shape are the usual unchecked premises. Ask what was not covered and which risks are still open, because a solid completion states them. Finally, ask whether the change touched anything on the risk list, such as security, contracts, production, or migrations, and if so what the grill gate concluded. A claim that cannot answer "what did you observe?" is an implementation report, not verification.
