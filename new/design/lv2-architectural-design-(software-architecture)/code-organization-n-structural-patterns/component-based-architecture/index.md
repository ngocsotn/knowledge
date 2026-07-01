# Component-Based Architecture

Component-Based Architecture partitions complex user interfaces or codebases into self-contained, reusable modules (Components).
- **Frontend Standard:** Every modern framework (React, Svelte, Vue) enforces this, structuring code around cohesive files containing local HTML, CSS, and JS logic.
- **Encapsulation:** Components hide their internal state, exposing only clear props/interfaces for inputs and event callbacks for outputs.

## Interview Questions & Answers

### Q1: What is the difference between Smart (Container) and Dumb (Presentational) Components?
- **Answer:** **Smart Components** manage state, handle API fetching, and coordinate business logic. **Dumb Components** are stateless; they receive data via props, render UI elements static-only, and trigger events for user interactions, maximizing UI reuse and ease of testing.
