# Model Context Protocol (MCP)

Comprehensive interview study guide covering the Model Context Protocol (MCP), its architecture, host/client/server communication models, core primitives, and why it is a game-changer for AI integrations.

---

## 1. What is Model Context Protocol (MCP)?

The **Model Context Protocol (MCP)** is an open-source standard created by Anthropic that establishes a secure, structured protocol for sharing data and capabilities between AI models (acting as clients/hosts) and external data sources or execution environments (acting as servers).

```
Traditional Integration (Fragile & Bloated)
Clients (Cursor, Claude, Copilot) ──► Custom APIs (M x N connectors) ──► Sources (DB, GitHub, Search)

MCP Integration (Clean & Standardized)
Clients (Cursor, Claude, Copilot) ────┐
                                      ├──► [MCP Standard] ──► MCP Servers (DB, GitHub, Search)
Local / Remote Tools ─────────────────┘
```

### The $M \times N$ Integration Problem
Previously, connecting $M$ different AI clients (e.g., Cursor, Claude Desktop, VS Code, ChatGPT) to $N$ different tools and data sources (e.g., PostgreSQL, GitHub, Slack, local CLI) required writing $M \times N$ separate custom API integrations. 
* **The MCP Solution:** Standardizes communication. A client implements the MCP client specification once, and any tool/database implements the MCP server specification once. Any MCP client can now instantly connect and communicate with any MCP server, reducing integration complexity to $M + N$.

---

## 2. MCP Core Architecture & Communication Flow

MCP operates on a bidirectional **Client-Server-Host** topology over standard transport layers:

```
┌────────────────────────────────────────────────────────────────────────┐
│                              AI Host (IDE / App UI)                    │
│  ┌──────────────────────┐                     ┌─────────────────────┐  │
│  │      LLM Engine      │                     │     MCP Client      │  │
│  └──────────┬───────────┘                     └──────────▲──────────┘  │
└─────────────┼────────────────────────────────────────────┼─────────────┘
              │ (1) Tool Spec                              │ (4) Session Response
              ▼                                            │
   ┌───────────────────────────────────────────────────────┼─────────────┐
   │                                                     JSON-RPC        │
   │                                                       Stdio / SSE   │
   └───────────────────────────────────────────────────────▲─────────────┘
              │ (2) Execute Tool Request                   │ (3) Execution Result
              ▼                                            │
┌─────────────┴────────────────────────────────────────────┴─────────────┐
│                              MCP Server                                │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │   Exposed Capabilities: Resources (Read) | Tools (Write/Run)     │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

### A. Core Architectural Roles
1. **The Host (AI Platform/IDE):** The parent software environment (like Claude Desktop, Cursor, or a custom developer CLI) that manages the user session, displays the UI, orchestrates client-server bindings, and communicates directly with the foundational LLM.
2. **The Client:** A protocol layer running inside the Host that initiates connections, manages the session lifecycle, negotiates capabilities, translates model intents into structured RPC requests, and routes server outputs back to the model.
3. **The Server:** A lightweight, independent process (running locally or remotely) that implements the MCP specification. It declares its available resources, prompts, and tools to the client, and safely executes those actions when requested.

### B. Transport Layer Protocols
MCP is transport-agnostic, supporting two primary bidirectional transport mechanisms:
* **Standard Input/Output (Stdio):** Best for local servers (running on the same machine as the Host). The client spawns the server as a child process and communicates directly via `stdin` and `stdout`.
* **Server-Sent Events (SSE) / HTTP:** Best for remote servers. The client sends write operations via standard HTTP POST requests, and the server streams real-time updates and notifications back to the client using a persistent Server-Sent Events (SSE) connection.

---

## 3. The Three Core Primitives: Resources, Prompts, and Tools

An MCP server exposes three fundamental interfaces to the AI model, known as the core primitives:

```
┌──────────────────────────────────────────────────────────────┐
│ MCP Primitives                                               │
├───────────────────┬───────────────────┬──────────────────────┤
│ Primitive         │ Model Permission  │ Real-World Example   │
├───────────────────┼───────────────────┼──────────────────────┤
│ Resources         │ Read-Only         │ Database Schemas     │
│ Prompts           │ Template Load     │ Code Review Slashing │
│ Tools             │ Executable/Write  │ Run Bash / Edit File │
└───────────────────┴───────────────────┴──────────────────────┘
```

### A. Resources (Read-Only Context)
Resources are structured, read-only data sources that the server makes available to the AI.
* **How it works:** The server exposes a catalog of items with unique custom URI patterns (e.g., `postgres://prod-db/tables/users/schema`). The model can inspect, query, and read the contents of these resources to dynamically inject raw facts into its context window.
* **Examples:** Log files, local file contents, database table schemas, Git commit histories.

### B. Prompts (Predefined Workflows)
Prompts are reusable, parameterized templates and slash commands defined on the server side.
* **How it works:** The server provides a list of prompt templates to the client. The user can select these prompts in the UI, and the server generates the exact system/user instructions with placeholders resolved (e.g., `/explain-error [log_file]`).
* **Examples:** Code generation templates, security auditing workflows, bug analysis scripts.

### C. Tools (Executable Actions / Write Operations)
Tools are active, executable functions that the AI model can trigger to perform side-effects, edit files, or execute operations.
* **How it works:** The server exposes a tool schema using standard JSON Schema definitions (detailing arguments, descriptions, and types). The model reads the schema and generates a JSON request specifying which tool to run and with what arguments. The client executes the tool on the server and feeds the raw console output/result back to the model.
* **Examples:** Writing to a local file, executing a bash script, querying Google Search, creating a Jira ticket.

---

## 4. Why MCP is a Game-Changer

1. **Enterprise Security Boundaries:**
   * Traditional plugins run code *inside* the AI provider's cloud, demanding deep access credentials.
   * **MCP isolation:** Servers run locally inside the developer's private VPC or secure local system. The host (e.g., Cursor) can only request tool executions via JSON-RPC, keeping sensitive source code, databases, and private credentials completely isolated from third-party AI provider clouds.
2. **Stateless JSON-RPC Protocol:**
   * Uses standard JSON-RPC 2.0. This eliminates the need for maintaining heavy socket connections or complex sessions, making it highly robust, easy to debug, and simple to implement in any language (Go, Python, TypeScript, Rust).
3. **Decoupled Engine Scaling:**
   * Because capabilities are declared dynamically via a simple handshake (`initialize` request), upgrading, replacing, or patching an MCP server does not require redeploying or altering the parent AI platform/client.

---

## 5. Popular Interview Questions & High-Impact Answers

### Q1: What is the $M \times N$ integration problem in AI engineering, and how does the Model Context Protocol resolve it?
* **Answer:** The $M \times N$ problem states that connecting $M$ different AI clients (e.g., Claude Desktop, VS Code, Cursor, Copilot CLI) to $N$ separate tools or data sources (e.g., Postgres, GitHub, Slack, local terminal) requires writing unique, custom connectors for every single pairing, leading to fragile and bloated code. **MCP** solves this by establishing a standardized JSON-RPC communication bridge. Clients implement the MCP specification once, and data sources implement the server specification once, reducing the required connector overhead to a scalable $M + N$ model.

### Q2: Explain the security benefits of using local MCP servers over cloud-hosted AI plugins.
* **Answer:** Cloud-hosted plugins require developers to upload sensitive database credentials, SSH keys, or private codebases to third-party AI provider servers to enable tool execution. An **MCP server** can run locally inside the developer's machine or private network behind the firewall. The AI Client (host) communicates with the local MCP server over standard stdio pipes. The server performs all execution locally, returning only the text results back to the LLM context. No raw credentials, source code, or private databases are ever exposed to the external AI cloud, preserving security boundaries.

### Q3: What are the three core primitives of the MCP protocol, and how do they differ in model permissions?
* **Answer:** The three core primitives of MCP are **Resources**, **Prompts**, and **Tools**:
  1. **Resources** are read-only data sources (like file contents or DB schemas) that the model can request to inspect.
  2. **Prompts** are pre-configured, parameterized user workflows and templates (like `/refactor-code`) that the user can load.
  3. **Tools** are executable, write-capable actions (like running bash commands, hitting external APIs, or writing to disk) that the model can dynamically invoke to modify state.

### Q4: How does communication work under the hood between a local MCP Host and an MCP Server?
* **Answer:** Local MCP communication relies on standard input/output (**stdio**). When the Host starts up, it spawns the MCP Server as a child process. The Host's MCP Client layer communicates with the Server by writing standard **JSON-RPC 2.0** message envelopes to the child process's `stdin`, and reading responses from `stdout`. If a tool needs to execute, the client writes a request, the server executes the action locally, and returns the standard result to the client's standard output stream, which is then parsed and injected into the LLM context.
