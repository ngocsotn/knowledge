# Micro-Frontends Architecture

**Microfrontends** apply the architectural ideas of microservices to the frontend. An application is split into individual, autonomous, domain-focused frontend modules that are composed into a single, cohesive user experience.

### Integration Strategies:
1. **Build-Time Integration (npm packages)**:
   - *Mechanics*: Each microfrontend is compiled and published as a private npm package. The parent container installs them and builds a single monolithic bundle.
   - *Pros*: Simple, compile-time type-safety, robust tooling.
   - *Cons*: Releasing a change to one microfrontend requires recompiling and redeploying the *entire parent container* (coupled deployment).
2. **Runtime Integration (Module Federation - Recommended)**:
   - *Mechanics*: Each microfrontend is compiled and deployed independently as a separate host endpoint. The container loads these dynamic entries at runtime over the network using Webpack Module Federation or Vite Federation.
   - *Pros*: Complete independent deployments. Releasing Microfrontend A instantly updates the live application without touching other modules.
   - *Cons*: Complex routing orchestration; risk of runtime script failures; complex dependency sharing (avoiding loading React 18 multiple times).

### Critical Operational Struggles:
- **CSS Leakage**: Microfrontend A's CSS styles can pollute Microfrontend B's elements.
  - *Fix*: Use strict CSS Modules, BEM (Block-Element-Modifier) naming conventions, or wrap each microfrontend inside a **Shadow DOM** to guarantee style encapsulation.
- **State Sharing**: Sharing state across microfrontends should be avoided to prevent tight coupling.
  - *Fix*: Use native browser **Custom Events** (`window.dispatchEvent()`) for decoupled, asynchronous pub/sub messaging.

---

## Interview Questions & Answers

### Q1: How does Module Federation operate in micro-frontends?
- **Answer:** Module Federation allows a Svelte or React application to dynamically import compiled JS components from a completely separate, independently deployed host build at runtime over the network, resolving singletons and dependencies on-the-fly.
