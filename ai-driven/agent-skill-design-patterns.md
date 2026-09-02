# Agent Skill Design Patterns

A skill is a reusable, bounded procedure. It is not a free-form prompt. It says when to act, what may be used, how success is proved, and when to stop.

Most agent failures are not model failures. They are missing structure. The agent rebuilt something the project already had, ran an action nobody approved, or reported success it never checked. The patterns below are the recurring fixes for those failures.

## Vocabulary

```text
skill      = how to perform a bounded task
capability = existing code, package, library, component, service, or other reusable implementation
tool       = mechanism used to inspect, change, run, or verify work
```

Keep the three separate. A bad skill produces the wrong workflow, a missing capability produces duplicated code, and a missing tool produces claims nobody can check. A prompt that tries to be all three at once is the usual cause of sprawl.

## Anatomy of a Skill

Every pattern in this document is a way of filling in one of these fields.

| Field           | Purpose                                                      | Example: "review a pull request"                                                    |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------ |
| Trigger         | When the skill fires, and when it does not                     | A PR is opened or updated on a non-draft branch                                        |
| Input           | The context it needs before acting                             | Diff, repository conventions, acceptance criteria, related existing code               |
| Reuse path      | Ordered search for existing capabilities                       | Existing package, then project module, then test utility, then approved tool           |
| Allowed actions | The explicit permission boundary                               | Read code, inspect dependencies, modify scoped files, run checks. No pushes, no deploys |
| Verification    | Evidence that proves the acceptance criteria                   | Type check and test suite pass on the changed paths, unresolved risks stated           |
| Stop condition  | When to hand back                                              | Evidence is sufficient, a blocker is found, or approval is required                    |

A missing field has a predictable failure. No trigger and it fires at the wrong time. No allowed actions and it creeps out of scope. No stop condition and it runs until the context is gone.

## Pattern Catalog

| Pattern                     | Problem it solves                                        |
| --------------------------- | -------------------------------------------------------- |
| Reuse-First                 | The agent rebuilds something the project already has      |
| Grill-Me                    | The agent builds the wrong thing from a vague request     |
| Evidence-Based Verification | The agent claims done without checking                    |
| Progressive Disclosure      | Relevant rules get buried under irrelevant ones           |
| Bounded Scope               | The task expands until the context runs out               |
| Allowed-Actions Boundary    | The agent touches systems outside its remit               |
| Validate Before Execute     | Malformed or invented tool calls reach real systems       |
| Idempotent Action           | Retries duplicate side effects                            |
| Escalation Gate             | An irreversible action happens without approval           |

---

## Pattern: Reuse-First

**Problem.** A model with no view of the repository defaults to writing new code, because generating is easier than searching. The result is a third date formatter, a second retry helper, and a parallel validation layer nobody knows about.

**Pattern.** Put an explicit inspect step before the plan step, and search in a fixed order. Stop at the first suitable match.

```text
existing package or library
    -> existing project code, module, component, or service
    -> existing project utility, script, test helper, or template
    -> approved external tool or service
    -> new implementation
```

An exact match is not required. Prefer, in order:

```text
reuse directly
    -> compose existing capabilities
    -> extend existing code
    -> adapt existing code with a small boundary layer
    -> create new implementation
```

Reuse when the candidate's scope, assumptions, interfaces, dependencies, runtime constraints, and security model fit the task.

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

**Examples.**

* Asked to "add retry with backoff to the payment client", the agent greps for `retry` and finds `lib/http/withRetry.ts` already wired to the same telemetry. It wraps the existing helper instead of writing a new loop.
* Asked to "parse and format the invoice dates", it checks `package.json`, sees `date-fns` is already a dependency used in twelve files, and uses it rather than hand-rolling padding logic.
* Asked to "add rate limiting to the upload endpoint", it finds the gateway already enforces per-IP limits, so the correct change is a config value, not middleware.
* A counterexample: an internal session library assumes a single-tenant synchronous request. The new endpoint is multi-tenant and async. Here reuse is wrong, and the right move is to adapt it behind a thin boundary or build new.

**Fails when.** The agent has no cheap way to search. If listing dependencies and grepping the codebase costs more than writing the function, it will write the function.

---

## Pattern: Grill-Me

**Problem.** Vague requests produce confident, wrong work. "Add caching to the API" has at least six reasonable readings, and the agent will pick one silently.

**Pattern.** Before building, the agent interviews the requester. One question at a time, walking the decision tree, answering from the codebase where it can rather than asking. It keeps going until the fuzzy parts are explicit, then hands back a plan with real acceptance criteria and waits for confirmation.

```text
request: "add caching to the API"
question 1: which endpoints?          -> the three read-heavy report endpoints
question 2: shared or per-user?       -> per-user, responses include private rows
question 3: invalidation trigger?     -> on write to the underlying tables
question 4: acceptable staleness?     -> 60 seconds
plan: per-user cache, 60s TTL, write-through invalidation on 3 tables
```

The same pattern runs in reverse as a pre-action risk gate, where the agent interrogates its own position instead of yours:

```text
proposed action: deploy authentication change
question 1: is rollback tested?        -> no
question 2: is approval in scope?      -> no
question 3: what evidence is missing?  -> rollback test and approval
decision: stop; request the missing evidence and approval
```

**Examples.**

* "Make the dashboard faster" turns into: which page, which percentile, measured how, and is the bottleneck the query or the render. Two of those are answerable from existing traces, so the agent answers them itself and asks only the other two.
* "Clean up the old users table" turns into: drop, archive, or soft-delete, which is a one-way door if answered wrong.
* Skipped correctly: "fix the typo in the README heading" needs no interview.

**Fails when.** It runs on everything. Time-box it, skip it for routine low-impact work, and require every question to point at a real missing decision, assumption, boundary, or authority problem. Unconditional gates become rituals people click through.

---

## Pattern: Evidence-Based Verification

**Problem.** Writing an implementation feels like completing the task, so the agent reports success on the strength of having produced code.

**Pattern.** Match the evidence to the type and risk of the change, then state what was observed. Tests are one form of evidence, not the definition of it.

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

**Examples.**

* A migration is not verified by "the file was written". It is verified by running it against a copy of the staging schema, running the down migration, and comparing row counts.
* A config change with a green unit suite is unverified if no test loads that config path. The evidence is a boot with the new value.
* A dependency bump is verified by the build, the type check, and the changelog for breaking entries, not by the test suite alone.
* A docs change needs no test run. Over-verifying is its own waste.

Keep the layers separate. Whether the agent retrieved the right context and whether it made the right decision are different questions, and merging them hides failures.

**Fails when.** The checks run but nobody reads the output. "Tests passed" with no named test and no touched path is a claim, not evidence.

---

## Pattern: Progressive Disclosure

**Problem.** Every rule loaded up front competes with every other rule. Past a certain size the instructions that matter get ignored, and every request pays for guidance it never uses.

**Pattern.** Keep the entry point small. Load instructions, references, dependencies, and code only when the current task and the chosen path need them.

**Examples.**

* A code-review skill loads the accessibility checklist only when the diff touches components, and the migration checklist only when it touches `db/`.
* A deploy skill keeps the rollback runbook in a linked document and pulls it in only after a failure.
* The same idea applies to tools. Twenty overlapping tools route worse than six with clear boundaries.

**Fails when.** The split is by topic rather than by trigger. If the agent cannot tell which document to load, it loads all of them and you are back where you started.

---

## Pattern: Bounded Scope

**Problem.** "Fix the failing test" becomes a refactor of the module, then a rename across the package, then a stalled run with no output.

**Pattern.** State the stop condition in the skill itself: evidence is sufficient, a blocker is found, or approval is required. Treat "blocked, and here is what is missing" as a successful outcome, not a failure.

**Examples.**

* A bug-fix skill that stops after the failing test goes green and lists adjacent problems it noticed instead of fixing them.
* A dependency-upgrade skill that upgrades one package per run, so the diff stays reviewable.
* A one-off procedure stays in the request. Create a new skill only when the workflow itself is reusable, or the library fills with near-duplicates nobody can choose between.

**Fails when.** Stopping is treated as failure. Then the agent keeps going to avoid looking unhelpful.

---

## Pattern: Allowed-Actions Boundary

**Problem.** An agent with unscoped permissions eventually uses them.

**Pattern.** Enumerate what the skill may do, and split read-only tools from write tools with different permission levels.

**Examples.**

* A review skill can read anything and run checks, but cannot push, deploy, or edit files outside the diff.
* A triage skill can read logs and comment on issues, but cannot close them.
* A local agent can run the test suite but reaches production only through a separate, explicitly approved skill.

**Fails when.** The boundary is written as advice rather than enforced by the tools. If the shell tool exists, "please do not deploy" is a suggestion.

---

## Pattern: Validate Before Execute

**Problem.** Models invent plausible tool calls: table names that do not exist, resource IDs never seen, arguments in the wrong shape.

**Pattern.** Put a validation stage between reasoning and execution. Check the tool exists, validate arguments against the schema, and reject malformed calls instead of coercing them. Constrain arguments with enums and identifiers the agent must have seen. Return a specific error the model can act on.

**Examples.**

* A SQL tool that accepts only table names from the live schema, so a hallucinated `user_accounts_v2` fails at validation rather than at runtime.
* A ticket tool that accepts an issue ID only if it appeared in an earlier tool result.
* A loop bounded by step, time, and token budgets, with repeat detection, since the same failing call three times is drift, not progress.

**Fails when.** Errors are generic. "Invalid input" gives the model nothing to correct, so it retries the same call.

---

## Pattern: Idempotent Action

**Problem.** Tool calls get retried, by the harness or by the model. Anything with a side effect can happen twice.

**Pattern.** Assume every call may run more than once. Use a caller-supplied idempotency key, prefer upsert over blind insert, and return the current state rather than a bare success so a repeat is detectable.

**Examples.**

* A refund tool keyed by transaction ID, so a retried call returns the existing refund instead of issuing a second one.
* A "create branch" tool that returns the existing branch when it already exists.
* A destructive tool that offers a dry-run mode and refuses to run without one on production targets.

**Fails when.** The tool returns only `ok`. The agent cannot tell a fresh success from a replay, and neither can you reading the trace.

---

## Pattern: Escalation Gate

**Problem.** Some actions cannot be undone by the agent, or commit systems the agent does not own.

**Pattern.** Route those through a human before execution, not after. The risk list is the trigger:

```text
security or data impact
breaking API or compatibility changes
production or deployment impact
irreversible migrations
missing or insufficient verification
unverified assumptions about dependencies
out-of-scope authority or approval
```

**Examples.**

* Dropping a production index, rotating a credential, or sending anything to customers.
* A schema migration where the down path has never been run.
* A change to auth or permissions logic, regardless of how small the diff looks.

The test: if the recovery step would be "ask a human", ask before acting.

**Fails when.** Everything is gated. Approval fatigue produces reflexive yes, which is worse than no gate because it looks like oversight.

---

## Composing Patterns Into a Loop

Individually the patterns are checklists. Together they are a workflow.

```text
request -> understand -> inspect -> reuse -> plan -> execute -> verify -> deliver
                              |                          |
                        grill-me if vague        escalation gate if consequential
```

Expanded, with the guard patterns in place:

```text
understand
   -> inspect existing capabilities        (reuse-first)
   -> identify reuse path
   -> clarify what is still vague          (grill-me)
   -> plan
   -> validate assumptions
   -> act                                  (validate before execute, idempotent action)
   -> observe
   -> verify                               (evidence-based verification)
   -> deliver                              (or stop, with what is missing)
```

Observe and verify are separate steps. Observing is reading what happened. Verifying is confirming it meets the acceptance criteria.

## Core Principles

```text
reuse before recreate
compose before duplicate
extend before fork
clarify before build
validate before execute
verify before claim
escalate before irreversible action
stop when evidence or authority is insufficient
```

---

## Interview Questions & High-Impact Answers

### Q1: How do you use AI in your day-to-day coding?
* **Answer:** Describe what you actually do and where you stop. Three uses are worth naming. **Exploration**: asking about an unfamiliar codebase, drafting a first version, comparing two approaches before committing. **Mechanical work**: refactors, test scaffolding, migrations, and repetitive edits where the shape is known and the blast radius is small. **Review**: pointing the model at a diff and asking what it cannot verify. Then state the boundaries: you do not merge code you have not read, you keep the agent scoped to a known set of files, and you require a check that actually ran before calling anything done. The interviewer is listening for whether you treat model output as a draft or as an authority. "I don't use it" and "it writes everything" both land badly.

### Q2: Walk me through the loop of an agent that edits code.
* **Answer:** Understand the request, inspect existing capabilities, pick the reuse path, plan, validate assumptions, act, observe, verify, deliver. A production framing adds the surrounding machinery: intake, context assembly, model reasoning, action validation, sandboxed execution, result processing, state update, then loop or terminate. Two steps carry most of the value. Inspecting before planning means the plan is written against what the codebase actually contains, so it does not propose building something that already exists. Validating assumptions before acting catches shaky premises while they are still cheap to fix. Observe and verify are separate: observing is reading what happened, verifying is confirming it meets the acceptance criteria.

### Q3: How do you know an agent's change is actually done?
* **Answer:** Verification has to match the type and risk of the change, and tests are only one form of evidence. Depending on the change the right evidence may be unit or integration tests, type checking, static analysis, a build, lint, a smoke check, contract validation, migration validation, a security check, or manual acceptance. A config change with a green unit suite is still unverified if nothing exercised the config path, and a docs change needs no test run at all. Keep the checks separated too: whether the agent retrieved the right context is a different question from whether it made the right decision, and merging them hides failures. The rule is that a completion claim rests on evidence you observed, not on the fact that code was written.

### Q4: What guardrails do you put in place before an agent takes an irreversible action?
* **Answer:** Start from the risk list: security or data impact, breaking API changes, production or deployment impact, irreversible migrations, missing verification, unverified assumptions about dependencies, and anything outside the agent's approval scope. What these share is blast radius. For those cases, require human approval before execution rather than after, run the action in a sandbox or dry-run mode first, confirm a rollback path exists and has been exercised, and set a confidence threshold below which the agent refuses instead of guessing. A simple test: if the recovery step would be "ask a human", ask before acting.

### Q5: How do you detect and handle hallucinated tool calls or agentic drift?
* **Answer:** Put a validation stage between reasoning and execution. Check the tool name exists, validate arguments against the schema, and reject anything malformed instead of coercing it. Constrain arguments with enums and identifiers the agent must have seen, so it cannot invent a resource ID. Bound the loop with step, time, and token budgets, and detect repetition, since the same failing call three times in a row is drift, not progress. Require the agent to cite the observation supporting each claim, which makes a fabricated result visible in the trace. When validation fails, return a specific error the model can act on rather than a generic failure.

### Q6: How do you control token cost and latency in an agent?
* **Answer:** Make the budget explicit rather than hoping the loop is short. Cache the stable prefix of the prompt so repeated turns do not pay for it again. Keep retrieval tight with a small top-k instead of dumping whole files into context. Route by difficulty, sending cheap and mechanical steps to a smaller model and reserving the large one for real reasoning. Set per-request limits on tokens, tool calls, and wall-clock time so a stuck run fails fast. Fire independent tool calls in parallel. Above all, load only what the current task needs, which is where progressive disclosure pays for itself.

### Q7: What do you log to debug an agent in production?
* **Answer:** One trace per run, correlated by a run ID, containing the prompt or skill version, the model, every tool call with its arguments and result, token counts, latency per step, retries, and the final decision with the citations that support it. That is enough to answer the three questions that actually come up: what did it see, what did it do, and why did it stop. Log the model's stated reasoning separately from the observed tool results so you can tell a wrong conclusion apart from a bad observation. Without per-step traces you cannot distinguish a retrieval failure from a reasoning failure, and you end up rewriting the prompt to fix a search bug.

### Q8: Tool calls get retried and are not deterministic. How do you keep writes safe?
* **Answer:** Assume every tool call may run more than once. Make writes idempotent with a caller-supplied key, prefer upsert over blind insert, and have tools return the current state rather than a bare success, so a repeat is detectable instead of silently doubling an effect. Split read-only tools from write tools and give them different permission levels, so retries on the read path cost nothing. Offer a dry-run mode for anything destructive, and refuse to build a destructive tool that cannot be replayed safely.

### Q9: How do you stop context from bloating as you add tools and instructions?
* **Answer:** Use progressive disclosure. Load only the instructions, references, dependencies, and code that the current task and the chosen path need, and keep the detail in linked documents pulled in on demand. This fixes two problems at once: context dilution, where the rules that matter get buried under rules that do not and end up ignored, and cost, where every request pays for instructions it never uses. The same applies to tools. A large set of overlapping tools makes routing worse, not better, so merge near-duplicates and give each one a clear boundary. When a skill starts collecting conditional branches for unrelated situations, split it.

### Q10: How do you decide between reusing existing code and writing new code, and how do you get an agent to do the same?
* **Answer:** Existing code already carries tested behavior, known interfaces, and an accepted security posture, so rewriting it duplicates maintenance work and brings back solved bugs. The search order is: existing package or library, then existing project code, then an existing utility, script, test helper, or template, then an approved external tool or service, and only then something new. Reuse is unsafe when the candidate's scope, assumptions, interfaces, dependencies, runtime constraints, or security model do not fit, such as a library assuming a single-tenant synchronous context being pulled into a multi-tenant async path. Getting an agent to follow this takes an explicit inspect step before planning, because a model with no view of the repository will always default to writing new code.

### Q11: A library covers 80% of the requirement. What do you do?
* **Answer:** An exact match is not required. Prefer, in order: reuse directly, compose existing pieces, extend, adapt behind a small boundary layer, then build new. For an 80% match, adapt it behind a thin wrapper so the remaining 20% sits in one place that is easy to test and easy to delete once upstream catches up. Forking or reimplementing is the worst option because you quietly take on the maintenance cost of the 80% that already worked.

### Q12: How do you write a tool or skill description the model actually routes to correctly?
* **Answer:** Treat the description as the routing signal, not documentation. Give it a specific trigger stating the situation that invokes it, and an explicit "do not use this when" clause, since most misrouting comes from two tools whose descriptions overlap. Name the inputs and outputs concretely instead of describing them abstractly. Write error messages that tell the model what to do next rather than just reporting failure. Then test it: run ambiguous prompts through and check which tool gets picked. If two tools compete on the same prompt, the fix is usually to merge them or sharpen the boundary, not to add more words.

---

## Lower-Frequency Questions and Follow-Ups

These come up less often on their own, and more as follow-ups once you have already brought up skills, guardrails, or verification.

### Q13: What is the difference between a skill, a capability, and a tool?
* **Answer:** A **skill** is a bounded procedure that says when to act, what may be used, how success is proved, and when to stop. A **capability** is an existing implementation the skill draws on, such as a package, library, module, or service. A **tool** is the mechanism used to inspect, change, run, or verify work, such as file reads, shell access, or a test runner. They fail differently: a bad skill produces the wrong workflow, a missing capability produces duplicated code, and a missing tool produces claims nobody can check. The vocabulary is not standardized across the industry, so define your terms before leaning on them.

### Q14: What is the `grill-me` pattern?
* **Answer:** In its common form, `grill-me` makes the agent interview **you** before it builds anything. It walks the decision tree one question at a time, explores the codebase when a question can be answered from the code, and keeps going until the vague parts of the request are explicit. The output is a sharper plan with real acceptance criteria, and the agent still needs confirmation before acting. Some teams also run the pattern in reverse as a pre-action risk gate, where the agent interrogates its own position before a consequential step: is rollback tested, is approval in scope, what evidence is missing. Either way it should be time-boxed and skipped for routine low-impact work, since running it on everything turns it into a ritual people learn to click through.

### Q15: Which good practices do agents break most often?
* **Answer:** The two that break most are **verify before claim** and **stop when evidence or authority is insufficient**. An agent that has just written plausible code is strongly inclined to report success, because producing the implementation feels like finishing the task. Stopping is rarer still, because it reads as failure unless the workflow explicitly treats "blocked, and here is what is missing" as a good outcome. The others in the set, reuse before recreate, compose before duplicate, and extend before fork, get broken mostly when the agent never inspected the codebase in the first place.

### Q16: An agent reports "done, implemented and working." What do you ask next?
* **Answer:** Ask what evidence was observed rather than what was written: which checks ran, against which code path, and what the output said. Ask what was assumed instead of verified, since dependency behavior, environment config, and data shape are the usual unchecked premises. Ask what was not covered and which risks are still open. Then ask whether the change touched anything on the risk list, such as security, contracts, production, or migrations. A claim that cannot answer "what did you observe?" is an implementation report, not verification.
