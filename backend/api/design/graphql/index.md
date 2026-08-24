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

## 4. Enterprise GraphQL Optimization & Orchestration

Operating GraphQL APIs in massive, distributed enterprise architectures introduces specialized design choices around streaming binaries, cross-service schema unification, and safety boundaries.

### A. Architectural Strategies for Binary File Handling
Handling file uploads natively in a JSON-centric GraphQL environment requires selecting between base64 wrapping, custom multipart requests, or presigned decoupling.

1. **Base64 String Encoding:**
   * *Mechanism:* The file is converted into a base64-encoded string and passed inside standard variables in a mutation.
   * *Trade-off:* Inefficient. Base64 encoding increases total payload transfer sizes by ~33%, causes high CPU parsing spikes during server decoding, and bypasses file streaming capabilities. Suitable only for micro-images (e.g., small avatars).
2. **Multipart Request Spec:**
   * *Mechanism:* Forces the GraphQL engine to accept multi-part form data containing standard operation JSON mappings alongside raw binary files in a single HTTP request.
   * *Trade-off:* Violates the strict standard JSON transit model of GraphQL and adds parsing load directly onto the core GraphQL application gateway node.
3. **Presigned Storage Redirects (Recommended for Scale):**
   * *Mechanism:* Offloads binary transmission from the GraphQL server completely.
     1. The client issues a GraphQL Query requesting an upload URL: `query { getPresignedUrl(fileName: "video.mp4") }`.
     2. The server responds with a secure, temporary, presigned S3/GCS URL.
     3. The client uploads the binary directly to the bucket via a PUT request.
     4. The client issues a final GraphQL Mutation passing the generated object storage identifier.

---

### B. Microservice Aggregation: Schema Stitching vs. Apollo Federation
When multiple backend microservices own portions of the organizational data model, you must choose between gateway-centric stitching or subgraph-driven federation to present a unified API.

1. **Schema Stitching:**
   * *Mechanism:* A central gateway service fetches the schemas of all downstream services at startup, merges (stitches) them into a single graph, and maintains custom resolver logic within the gateway to bridge cross-service relationships.
   * *Trade-offs:* Simple to implement initially, but centralizes business logic inside the gateway. The gateway team becomes a bottleneck for every downstream modification.
2. **Apollo Declarative Federation (Recommended):**
   * *Mechanism:* Decentralized ownership. Each downstream microservice operates as an independent **Subgraph**. Services declare their own types and extend types owned by other subgraphs using declarative decorators (e.g., `@key`, `@extends`, `@external`).

```graphql
# Downstream User Subgraph:
type User @key(fields: "id") {
  id: ID!
  name: String!
}

# Downstream Post Subgraph (extends User type seamlessly):
type User @key(fields: "id") @extends {
  id: ID! @external
  posts: [Post!]! # Relational link resolved by the Post service!
}
```

   * *Trade-offs:* Downstream teams possess total ownership of their extensions. The gateway remains a thin, stateless coordinator compiling query execution plans dynamically. The trade-off is higher initially designed infrastructure complexity.

---

### C. Continuous Schema Evolution and Breaking Change Mitigation
GraphQL rejects version numbers (`/v1`, `/v2`) in favor of a single, continuously evolving schema.

1. **Adding Fields:**
   Fields are strictly additive. Adding a non-nullable or nullable field does not affect legacy clients since they only query the fields they declare.
2. **Deprecation Pipelines:**
   When retiring elements, apply the `@deprecated` directive:
   `type User { email: String! @deprecated(reason: "Use primaryEmail instead") }`
   Modern IDE client consoles and generated document suites will instantly flag this to developers.
3. **Trace Telemetry Analysis:**
   Utilize schema monitoring registries (e.g., Hive, Apollo Studio) to trace exactly which clients are executing calls to deprecated fields.
4. **Surgical Removal:**
   Once trace telemetry metrics confirm active queries for the deprecated fields have dropped to exactly `0`, the fields can be safely expunged.
5. **CI/CD Schema Checks:**
   Integrate automated schema check tools (e.g., `graphql-inspector`) in build pipelines. Any developer commit that introduces a breaking change (such as deleting an active field, changing a type from nullable to non-null, or altering an argument schema) without a formal deprecation period will automatically block the deployment.

---

## 5. Interview Masterclass: High-Impact Q&As

### Q1: What is the GraphQL N+1 Query Problem under the hood, and how does the DataLoader pattern resolve it?
* **Answer:**
  * **The Problem:** When resolving queries with nested array relations (e.g., fetching 10 posts and their authors), a standard GraphQL resolver executes `SELECT * FROM posts LIMIT 10` (1 query), and then triggers the `author` nested field resolver once *for each* of the 10 posts, executing 10 individual SQL queries: `SELECT * FROM users WHERE id = X` (N queries). This results in $1+N=11$ separate database calls.
  * **The DataLoader Solution:** DataLoader uses **batching** and **caching** inside the JS event loop. Instead of resolving authors instantly, DataLoader queues the requests during a single tick. Once the call stack clears, it executes a single batched SQL query using collected IDs: `SELECT * FROM users WHERE id IN (1, 2, 3...)`. It then distributes the retrieved author records back to the waiting resolvers.

### Q2: How do you defend a production GraphQL server from malicious, deeply nested query attacks?
* **Answer:** Since GraphQL queries are client-defined, an attacker can submit deeply nested circular queries (e.g., `user -> posts -> author -> posts -> author...`) to crash the server's CPU and database memory. Three primary mitigations are used:
  1. **Query Depth Limiting:** The server parses the incoming query string into an **Abstract Syntax Tree (AST)** before execution, calculates the nested depth, and automatically rejects queries exceeding a safe threshold (e.g., maximum depth of 5).
  2. **Query Complexity Analysis:** Assign numerical cost scores to schema fields (e.g., scalar field = 1, list fields = 10). The server calculates the cumulative complexity score of the query AST and blocks execution if it exceeds a limit.
  3. **Persisted Queries (Best Practice):** Disable ad-hoc queries entirely in production. During build-time, client queries are registered, hashed (SHA-256), and saved on the server. At runtime, clients only transmit the hash: `GET /graphql?hash=abcdef123...`. The server only executes these pre-approved queries.

### Q3: Contrast Schema Stitching and Apollo Federation for microservices aggregation.
* **Answer:**
  * **Schema Stitching:** A central Gateway microservice actively fetches schemas from downstream APIs and stitches them together. The Gateway team is responsible for writing custom integration resolvers to connect types between services.
  * **Apollo Federation:** A decentralized subgraph architecture. Each downstream microservice (subgraph) declares its own types and extends types owned by other subgraphs using declarative metadata decorators (e.g., `@key`, `@extends`, `@external`). The Federation Gateway compiles these annotations at boot, generating a query execution plan automatically. This decouples teams and removes the Gateway as a business-logic bottleneck.

### Q4: How do you manage file uploads in a scalable, high-volume GraphQL application?
* **Answer:**
  * Avoid raw **Base64 string encoding** (which inflates files by 33% and spikes CPU decoding overhead) and **GraphQL Multipart Spec requests** (which violate standard JSON transit specs and consume Gateway file-parsing streams).
  * **Presigned Redirects (Recommended for Scale):** Decouple binary processing completely from the GraphQL stack. The client executes a GraphQL query to request an upload URL: `query { getPresignedUrl(fileName: "image.png") }`. The server returns a temporary, secure, presigned S3/GCS URL. The client uploads the binary file directly from the browser to S3 via a native `PUT` request. Once uploaded, the client triggers a standard GraphQL mutation, passing the file's final object storage URL.

