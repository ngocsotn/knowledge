# GraphQL API Design

Comprehensive study guide for understanding, designing, and optimizing production-grade GraphQL APIs, along with architectural comparisons and performance strategies.

---

## 1. What is GraphQL?

**GraphQL** is an open-source data query and manipulation language for APIs, and a runtime for fulfilling those queries with your existing data. Created by Facebook in 2012 and released publicly in 2015, it addresses the inflexibility and inefficiencies of RESTful APIs.

### Core Philosophy & Advantages
1. **Single Endpoint**: All operations go to a single endpoint (typically `POST /graphql`), reducing connection overhead and route management.
2. **Client-Driven Queries**: The client specifies *exactly* which fields it needs. This eliminates:
   - **Over-fetching**: Getting more data than needed, wasting network bandwidth.
   - **Under-fetching**: Getting insufficient data, requiring additional round-trip requests to other endpoints (N+1 HTTP requests).
3. **Strongly Typed Schema**: The API contract is explicitly defined using the **Schema Definition Language (SDL)**. This serves as a self-documenting contract between frontend and backend.
4. **Real-time Updates**: Built-in support for WebSockets-driven **Subscriptions** alongside standard Queries (reads) and Mutations (writes).

---

## 2. Core Concepts & Operations

### Schema Definition Language (SDL)
The schema defines types, relationships, and entry points:

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String
  author: User!
}

# Entry points for operations
type Query {
  getUser(id: ID!): User
  listPosts(limit: Int): [Post!]!
}

type Mutation {
  createPost(title: String!, content: String, authorId: ID!): Post!
}
```

- **`!` (Non-Null)**: Indicates that a field or argument cannot be null.
- **Resolvers**: Functions on the server that fetch the actual data for each field in the schema.

---

## 3. High-Impact Technical Challenges & Solutions

### The N+1 Database Query Problem
- **Problem**: When a user queries a list of resources and nested relationships, the server may query the database once for the list, and then make a separate query *for each item* in the list to load its relation.
  ```graphql
  query {
    listPosts(limit: 10) {
      title
      author { # Resolves 10 times individually! (1 + 10 = 11 queries)
        name
      }
    }
  }
  ```
- **Solution**: **DataLoader Pattern**. DataLoader uses batching and caching. It collects all individual requests for `author` during a single tick of the event loop, coalesces their IDs, and executes a single batched database query:
  `SELECT * FROM users WHERE id IN (1, 2, 3, ...)`

### Deep/Circular Query Attacks (Denial of Service)
- **Problem**: A malicious client can submit a deeply nested circular query to exhaust server CPU and database resources:
  ```graphql
  query Attack {
    getUser(id: "1") {
      posts {
        author {
          posts {
            author { # Infinitely nested
              name
            }
          }
        }
      }
    }
  }
  ```
- **Solutions**:
  1. **Query Depth Limiting**: Analyze the AST (Abstract Syntax Tree) of the incoming query before execution and reject any queries that exceed a configured depth (e.g., maximum depth of 5).
  2. **Query Complexity Analysis**: Assign cost values to fields (e.g., scalar fields = 1, lists = 10). Calculate the total complexity score of the query AST and reject if it exceeds a threshold.
  3. **Persisted Queries (Best Practice)**: Instead of sending full query strings over the wire, clients upload their query strings during build-time. The server stores them, assigns a SHA-256 hash to each, and at runtime, the client only sends the hash:
     `GET /graphql?hash=abcdef123456...`. The server executes only these pre-approved hashes, fully blocking ad-hoc query attacks.

---

## 4. Hard Interview Questions & Deep Answers

### Q1: How do you handle file uploads in GraphQL? What are the standard approaches and their trade-offs?
**Answer**:
There are three main approaches to file uploads in GraphQL:
1. **Base64 Encoding**:
   - *How*: File is encoded as a Base64 string and sent as a standard String input argument in a mutation.
   - *Trade-offs*: Simple to implement. However, Base64 encoding increases file payload size by ~33%, increases server CPU utilization to decode, and bypasses streaming.
2. **GraphQL Multipart Request Spec (RFC-compliant Multipart Uploads)**:
   - *How*: Uses standard multipart form-data. The request contains the GraphQL operations JSON, a map matching files to variables, and the binary files. Supported by libraries like `apollo-server` and `graphql-upload`.
   - *Trade-offs*: Allows file upload and metadata changes in a single GraphQL mutation. However, it violates the standard JSON transport of GraphQL, is harder to implement behind some API Gateways, and increases CPU overhead on the GraphQL server itself.
3. **Presigned URLs (Recommended for Scale)**:
   - *How*: Client requests a presigned S3/GCS upload URL via a GraphQL query, uploads the file directly from the browser to the object storage bucket (e.g., S3), and then submits a GraphQL mutation containing only the resulting file URL.
   - *Trade-offs*: Highly scalable. Offloads binary upload bandwidth and CPU processing entirely from the GraphQL server. Ideal for microservices architectures.

### Q2: Compare Schema Stitching vs. Apollo Federation for combining multiple microservices into a single GraphQL graph.
**Answer**:
- **Schema Stitching**:
  - *Mechanism*: A gateway microservice explicitly loads schemas from downstream GraphQL services and merges (stitches) them. The gateway configures custom resolvers to handle cross-service linkages.
  - *Trade-offs*: Simple for small setups. However, it centralizes integration logic within the gateway, making it a bottleneck and violating service autonomy.
- **Apollo Federation (Recommended for Enterprise)**:
  - *Mechanism*: Declarative subgraph composition. Downstream services (subgraphs) declare their own types and extend types owned by other services using decorators (e.g., `@key`, `@extends`, `@external`). The federation gateway (Supergraph) parses these annotations and generates a query execution plan automatically.
  - *Example*: User service owns the `User` type. Post service extends the `User` type to attach a nested list of `posts` based on the user's `@key(fields: "id")`.
  - *Trade-offs*: True service decoupling. Downstream teams own their extensions, and the gateway is completely stateless and thin. The trade-off is higher initial architectural complexity and reliance on federation-specific spec tooling.

### Q3: How do you implement schema versioning and handle breaking changes in GraphQL?
**Answer**:
- **GraphQL does not use version numbers (no `/v1` or `/v2`)**. Instead, it uses **Continuous Schema Evolution**:
  1. **Add non-breaking fields**: Add new fields alongside old ones. Clients that don't query the new fields are unaffected.
  2. **Deprecation**: Mark old fields with the `@deprecated(reason: "Use newField instead")` directive. Modern GraphQL clients and documentation tools automatically highlight this to developers.
  3. **Telemetry Analysis**: Use tracing/schema registry telemetry (e.g., Apollo Studio, Hive) to monitor if any active clients are still querying the deprecated fields.
  4. **Removal**: Once telemetry confirms 0 clients are querying the deprecated fields, safely remove the fields from the schema.
  5. **Breaking Change Protection**: Implement Schema Check steps in CI/CD pipelines (using tools like `graphql-schema-linter` or `spectaql`) to block commits that introduce breaking changes (e.g., deleting a field, changing field nullability, or changing argument types) without deprecation grace periods.
