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
