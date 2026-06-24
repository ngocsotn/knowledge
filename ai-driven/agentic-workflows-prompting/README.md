# Agentic Workflows & Prompt Engineering Patterns

Comprehensive study guide covering advanced prompt engineering patterns (Zero-shot, Few-shot, Chain-of-Thought), Agentic architectures (ReAct, Planning, Reflection), Tool Calling execution loops, and AI system evaluation strategies.

---

## 1. Advanced Prompt Engineering Patterns

Prompt engineering is the practice of crafting instructions to optimize LLM reasoning and structure output behavior without updating neural weights.

```
Zero-shot ────────► Few-shot (In-context) ────────► Chain-of-Thought (CoT)
"Write SQL"         "Here is an example... Write SQL"   "Let's think step-by-step. First..."
```

### A. Zero-shot Prompting
* **Concept:** Directly asking the model to perform a task with zero context or examples.
* **When to use:** Simple, standard tasks (e.g., "Translate this sentence to French").

### B. Few-shot Prompting (In-Context Learning)
* **Concept:** Providing one or more structured examples of the input-output mapping inside the prompt before asking the model to process the actual target input.
* **Why it works:** Harnesses the Transformer's ability to recognize patterns in-context, aligning the output format and style. Ideal for complex classification or structured extraction tasks.

### C. Chain-of-Thought (CoT) Prompting
* **Concept:** Instructing the model to generate its step-by-step reasoning process explicitly before outputting the final answer (e.g., adding "Let's think step-by-step").
* **The Cognitive Mechanism:** By generating reasoning tokens first, the model allocates computation steps (and attention vectors) to resolve sub-problems, significantly reducing logical errors and mathematical hallucinations.

---

## 2. Agentic Architectures: From Static to Dynamic loops

While static prompting runs in a single forward pass, **Agentic Workflows** execute in interactive, stateful loops, allowing the model to make dynamic decisions, correct its own errors, and run tasks autonomously.

```
Static Pipeline (Single Pass)
[Input Prompt] ──► [LLM Engine] ──► [Output]

Agentic Loop (Interactive & Stateful)
                ┌──────────────────────────────────┐
                ▼                                  │
[User Task] ──► [Reasoning/Plan] ──► [Call Tool] ──┴─► [Observe Result] ──► [Verify/Reflect] ──► [Final Success]
```

### A. ReAct (Reason + Act)
* **How it works:** Interleaves step-by-step reasoning thoughts with action execution.
* **The Cycle:**
  1. **Thought:** Model analyzes state: "I need to find the current price of stock X."
  2. **Action:** Model chooses a tool and formats arguments: `google_search("stock X price")`.
  3. **Observation:** The runtime executes the search and feeds the raw results back to the model context.
  4. The loop repeats until the model has sufficient facts to output the final answer.

### B. Plan-and-Solve (Planning)
* **How it works:** Instead of jumping straight to tool execution, the agent generates a complete multi-step execution plan first. It executes tasks sequentially, checking off progress, and replans dynamically if a step fails.

### C. Reflection & Self-Correction
* **How it works:** A multi-agent or dual-step pattern where the output of Generator Agent A is passed to a Critic Agent B. Agent B evaluates the output against strict rubrics (or compilers) and returns feedback. Agent A reads the critique, corrects its mistakes, and regenerates. This drastically increases code syntax correctness and logical accuracy.

---

## 3. Tool Calling (Function Calling) Execution Loop

Tool calling is the standardized mechanism allowing closed-system LLMs to interact safely with external APIs, filesystems, and databases:

```
1. Client sends Query + Tool Schema (JSON Schema)
2. LLM yields Tool Call Intention (Name + JSON Arguments)
3. Client intercepts Tool Call, runs action locally, and gets output
4. Client sends output back to LLM as Tool Observation Message
5. LLM reads Observation and synthesizes final natural language answer
```

* **Safety Boundary:** **The LLM never executes the tool directly.** The model only generates the *intention* to call a tool as structured JSON arguments. The client application running on your server parses this JSON, verifies permissions, runs the function, and injects the text output back into the chat history.

---

## 4. Evaluating AI Systems (Systematic Grading)

In software engineering, we write unit tests. In AI engineering, because LLM outputs are non-deterministic, we rely on systematic **Evaluations (Evals)**:

### A. Traditional Lexical Heuristics
* **BLEU (Bilingual Evaluation Understudy):** Measures n-gram overlap between generated text and reference text. Best for translation.
* **ROUGE (Recall-Oriented Understudy for Gisting Evaluation):** Measures recall overlap. Best for summarization.
* *Drawback:* Poor semantic awareness; penalizes valid synonyms.

### B. LLM-as-a-Judge (Modern Standard)
* **Concept:** Using a highly powerful, frontier model (like GPT-4 or Claude 3.5 Sonnet) to grade evaluation outputs using a set of structured scoring rubrics (**G-Eval**).
* **Metrics Tracked:**
  1. **Faithfulness:** Does the answer contain facts NOT supported by the retrieved context? (Measures hallucination).
  2. **Answer Relevance:** Does the response actually address the user's query?
  3. **Context Precision:** Did the retrieval system fetch relevant chunks at high ranks?

---

## 5. Popular Interview Questions & High-Impact Answers

### Q1: What is Chain-of-Thought (CoT) prompting, and why does it mathematically reduce hallucinations in LLM reasoning?
* **Answer:** LLMs are auto-regressive; they predict the next token based on all preceding tokens in the sequence. If you ask a complex question directly, the model must output the final answer in its first few tokens, relying on a single forward pass without computing intermediate states. **CoT** forces the model to write out its step-by-step reasoning first. Mathematically, this populates the context window with correct intermediate logical steps (tokens). When the model finally predicts the terminal answer, its self-attention layers can compute attention weights over these verified reasoning tokens, drastically reducing logical jumps and hallucinations.

### Q2: Explain the execution flow and safety boundaries of LLM Tool Calling. Does the model run the code?
* **Answer:** No, the model never runs any code. The application client provides the model with a catalog of available tools defined as **JSON Schemas**. The model parses the schemas, and if it decides to use a tool, it suspends text generation and outputs a structured JSON block specifying the tool name and arguments. The **client application** running on the host server intercepts this JSON, safely executes the function (e.g., querying a database or file), and sends the raw text output back to the model as a new `tool` message type. The model reads this observation and synthesizes a final response.

### Q3: What is "LLM-as-a-Judge" in AI evaluation, and how do you design a pipeline to measure RAG hallucinations?
* **Answer:** "LLM-as-a-Judge" is the process of using a frontier model (like Claude 3.5 Sonnet) to systematically grade application outputs at scale. To measure RAG hallucinations, we track **Faithfulness**:
  1. We feed the retrieved context chunks, the user query, and the final generated response into the Judge model.
  2. We instruct the Judge to break the generated response down into individual factual claims.
  3. The Judge verifies each claim against the retrieved context, assigning a binary `1` (supported) or `0` (not supported) score.
  4. The final Faithfulness score is the fraction of supported claims over total claims. A score of $1.0$ guarantees zero hallucination.
