# Project basic information

This is a markdown project for storing knowledge, mainly focus for interview in software engineering. Organize folders according to knowledge categories, each category will have many sub-categories and topics, each one will have content to read as a e-book and below always have Q&A for interview or core knowledge. For details, please check the `Repository layout` below.

## Getting started

- No specific scripts included in this project.

## Delivery

- Do not commit. Do not pull. Do not fetch. Do not push. Do not stash. Do not stage..., human will check and dot it.

## Repository layout

- `ai-driven` contain knowledge about AI Agent, AI driven related content in this AI industry. Not related to algorithm training AI or content related to training AI. Only about usage AI that help improve performance, the usage of AI in application like chat agent, auto flow...etc
- `auth` contain knowledge about Authentication, Authorization and related content of it.
- `backend` contain knowledge about backend development, API design, concurrency and race conditions, diagnostics, file handling, implementation challenges, and backend security.
- `browsers` contain knowledge about browser internals, layout and rendering engines, storage mechanisms (cookies, web storage, cache), and content compression.
- `caching` contain knowledge about caching fundamentals, local vs. distributed caching strategies, and Redis data structures and patterns.
- `culture-fit` contain knowledge about behavioral and culture-fit interview questions, professional mindset, communication, and working alongside AI.
- `database` contain knowledge about relational databases (SQL), non-relational databases (NoSQL), vector databases, indexing, query optimization, transactions, and scaling.
- `design` contain knowledge about multi-tier system and software design, spanning infrastructure/topology (Lv1), software architecture (Lv2), design patterns (Lv3), language-specific idioms (Lv4), and interview scenarios.
- `devops-cloud` contain knowledge about DevOps and cloud computing, including CI/CD pipelines, Docker, Kubernetes, virtualization, cloud architecture, and security.
- `dsa` contain knowledge about data structures and algorithms, time/space complexity, sorting, searching, and algorithm design paradigms.
- `frontend` contain knowledge about frontend engineering, modern frameworks (React, Next.js, Angular), DOM/V-DOM, rendering patterns, state management, performance, and frontend architecture.
- `git` contain knowledge about Git version control, internal architecture, branching workflows, and merge/rebase mechanics.
- `interview-scenarios` contain practical system design and technical interview scenarios, case studies, and structured problem-solving walkthroughs.
- `network` contain knowledge about computer networks, network protocols (HTTP, TCP/UDP, DNS, WebSocket, SSH, etc.), and network security.
- `observation` contain knowledge about system observability and telemetry, focusing on logging, metrics, and distributed tracing.
- `operating-system` contain knowledge about operating systems, process/thread management, CPU scheduling, memory, and processor architectures.
- `project-methodologies` contain knowledge about software delivery and project management frameworks, including Agile, Scrum, Kanban, and Waterfall.
- `specific-language` contain in-depth knowledge of specific programming languages, runtime environments, and syntax paradigms (e.g., JavaScript, TypeScript, Node.js).
- `test` contain knowledge about software quality assurance, testing pyramid (unit, integration, E2E), and methodologies like TDD and BDD.

## Markdown convention

- Each folder will have an index.md for common knowledge of those sub-folders belong to that main folder category, will have comparison of them, relation between them, common concept and different concept between them. always re-update this file when modify, add or delete other sub-categories.
- Each file will have many content sections like a blog to read, have graphical by `mermaid` if needed to display of flow/chart/...
- Each file also need to be easy to understand and readable for human. Having example for each section will be good.
- Each file will explain detail about the topic: why we need it? what it's solve? what it's problems bring with the things it's solve? When we should use it? when we should not use it? the pros, the cons? and maybe the cost if could explain.
- Below each file, will be list of questions for interview, core knowledge by format:

```
### Q1: question 1
* Answer: answers

### Q2: question 2
* Answer: answers

```

## Others convention if not mention

- Always ask user before doing anything.

## Security

- Masked all tokens or secrets or credentials with `***`, example: `password: 123***89`
