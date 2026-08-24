# LLM Fundamentals & Mechanics

Comprehensive study guide covering Large Language Model (LLM) mechanics, Transformer self-attention, tokenization constraints, sampling hyperparameters (temperature, top_p, top_k), system vs. user prompts, and structured output parsing.

---

## 1. How LLMs Work Under the Hood

Large Language Models are probabilistic auto-regressive systems built on the **Transformer Decoder architecture**.

```
Input Text ──► Tokenizer ──► Embedding ──► Self-Attention ──► Softmax Logits ──► Next Token (Output)
   ▲                                                                                  │
   └────────────────────────────────── Auto-Regressive Loop ──────────────────────────┘
```

* **Auto-Regressive Loop:** LLMs do not write complete paragraphs in a single computational pass. They generate text **one token at a time**. On each iteration, the previously generated token is appended to the input sequence, and the entire updated sequence is re-evaluated to predict the next token.
* **The Self-Attention Mechanism:** The core of the Transformer. Instead of processing text sequentially (like legacy RNNs), self-attention calculates a dynamic mathematical weight (attention score) between every word (token) in the input sequence simultaneously. This allows the model to capture deep semantic dependencies and contextual relationships across massive distances in the text.
* **Tokenization (Byte-Pair Encoding):**
  * LLMs cannot read raw characters or strings. Text is converted into numerical IDs using algorithms like **Byte-Pair Encoding (BPE)** (e.g., `tiktoken` for OpenAI, `SentencePiece` for Llama).
  * **The Token Rule:** On average, 1 token $\approx$ 4 characters, or 0.75 words in English.
  * **The Non-English Penalty:** BPE tokenizers are heavily optimized for English. A single complex Vietnamese character or word might be split into 3 or 4 separate tokens, dramatically consuming the context window and increasing execution costs.
  * **The Tokenizer Boundary Bug:** Because tokenizers split words based on statistical frequencies, they can fail at basic character/math reasoning (e.g., counting letters in a word) because they see "apples" as a single token ID rather than individual letters `a-p-p-l-e-s`.

---

## 2. Decoding & Sampling Hyperparameters

Before outputting a token, the model calculates a vector of raw scores (called **Logits**) representing the mathematical likelihood of every token in its vocabulary. Sampling hyperparameters scale these logits to shape the output behavior:

```
Logits (Raw Scores) ──► Logit Scaling (Temperature) ──► Truncation (Top_P / Top_K) ──► Softmax Probability ──► Sampled Token
```

### A. Temperature
Scales the raw logit scores before applying the Softmax function to convert them into probabilities:
* **Low Temperature ($T \to 0$):** Forces the model to pick only the single most probable token on every step (deterministic). Best for code generation, mathematical calculation, and factual lookups.
* **High Temperature ($T > 1$):** flattens the probability distribution, making unlikely tokens much more likely to be selected (creative/random).
* **The Math:** Logits are divided by $T$. When $T$ is small, the gap between the highest logit and others is amplified, causing Softmax to yield near-100% probability for the top token.

### B. Top-P (Nucleus Sampling)
Instead of looking at the entire vocabulary, Top-P restricts candidate tokens to the smallest subset whose **cumulative probability** exceeds the threshold $P$ (e.g., $P = 0.90$).
* If $P = 0.90$, the model ignores the bottom 10% of least likely tokens entirely, avoiding gibberish.

### C. Top-K
Limits the selection pool to exactly the $K$ most likely tokens (e.g., $K = 50$).

### D. Production Best Practice
**Never change both Temperature and Top-P simultaneously.** Tweak one or the other to maintain predictable output boundaries.

---

## 3. Prompt Architecture: System vs. User Prompts

```
┌────────────────────────────────────────────────────────┐
│ System Prompt (Guarding & Behavior)                    │
│ "You are a secure SQL compiler. Only output raw SQL."  │
├────────────────────────────────────────────────────────┤
│ User Prompt (Dynamic Task Context)                     │
│ "Get users whose age is greater than 20"               │
└────────────────────────────────────────────────────────┘
```

* **System Prompts (Instruction / Guardrails):**
  * Injected at the highest priority tier of the chat context.
  * Used to define the model's identity, behavior limits, permitted tools, and output formatting rules (e.g., "Always return JSON", "Do not write code explanations").
  * **Resilience:** System prompts are mathematically more resistant to user prompt injection attacks than instructions passed in user messages.
* **User Prompts (Task Context):**
  * Dynamic, runtime messages containing the exact inputs and instructions provided by the end-user.

---

## 4. Structured JSON Outputs: Mechanisms & Pitfalls

Modern software systems require structured JSON data from LLMs to execute downstream APIs, database inserts, and program logic.

### A. Core Output Paradigms
1. **Plain Prompting + Post-Processing (Legacy/Fragile):**
   * *Mechanism:* Instructing the model: "Return a JSON object with keys `name` and `age`."
   * *Pitfall:* Highly fragile. The model can easily include markdown wrapper ticks (````json ... ````), leading whitespace, trailing commas, or conversational intro text (e.g., "Here is your JSON:"), causing standard JSON parsers to crash.
2. **JSON Mode:**
   * *Mechanism:* The API runtime forces the model to generate valid JSON syntactically (supported by OpenAI/Anthropic/Gemini).
   * *Pros:* Guarantees the output parses as JSON.
   * *Cons:* Does not guarantee the JSON matches your specific required schema structure (e.g., keys can still be missing or of wrong types).
3. **Structured Outputs (Constrained Decoding / Grammar-Based):**
   * *Mechanism:* Providing a strict JSON schema (or Zod/Pydantic model) to the API. The inference engine uses **Grammar-Based Constrained Decoding**.
   * *How it works:* During token generation, the engine actively masks out invalid tokens at the mathematical logit level. If the schema specifies a number, the engine physically forbids the model from outputting alphabetical character tokens next.
   * *Pros:* **100% schema compliance guarantee**.

---

## 5. Local LLM vs. Cloud LLM & Local Inference Engines

With massive model sizing optimizations, software engineers can choose between deploying cloud-hosted frontier models or running compact models locally on consumer-grade developer machines.

### A. Local LLM vs. Cloud LLM

| Dimension | Cloud-Hosted LLMs (e.g., GPT-4o, Claude 3.5 Sonnet) | Local-Hosted LLMs (e.g., Llama 3.1 8B, Qwen 2.5) |
| :--- | :--- | :--- |
| **Raw Intelligence** | **State-of-the-Art (SOTA)**. Massive parameters (>1T) yield superior multi-step reasoning. | Highly competent on targeted tasks (coding, summarization) but limited on broad general logic. |
| **Data Privacy & Security**| Low. Payloads travel over public networks, posing enterprise compliance risks. | **Absolute Privacy**. Payload never leaves physical machine RAM; compliant with strict HIPAA/GDPR limits. |
| **Network Dependency**| Requires high-bandwidth, stable internet connections. | **100% Offline Capable**. Works seamlessly on airplanes or remote data-vaults. |
| **Operating Costs** | Pay-per-token API pricing. Can scale up expensively under high-volume production loads. | **Free & Infinite**. Limited only by physical hardware power and electricity. |
| **Inference Speed** | Bounded by network transit latency and global queue queues. | Bounded strictly by local CPU/GPU **Tokens Per Second (t/s)** write rates. |

---

### B. Inference Engines: Ollama vs. Llama.cpp
To run models locally, developer environments rely on low-level tensor runners or high-level daemon coordinators:

* **Llama.cpp (Low-Level Core):**
  * Written in pure C/C++ by Georgi Gerganov. It compiles directly to the host machine's hardware instruction set without bloated runtime containers.
  * It provides raw, close-to-the-metal inference performance, leveraging Apple Silicon **Metal** shaders, NVIDIA **CUDA**, or AMD **ROCm** directly.
* **Ollama (High-Level Wrapper):**
  * A developer-friendly daemon/wrapper built directly on top of Llama.cpp.
  * It simplifies local LLM management: handles automatic background downloading, registries (`ollama run llama3`), local memory context allocation, and automatically serves an **OpenAI-compatible HTTP REST API** (`localhost:11434`) out of the box, making it highly simple to hook up to existing software pipelines.

#### The Magic of Quantization (GGUF Format)
Frontier models are trained in FP16 or FP32 (16-bit or 32-bit floating-point numbers) representing neural connection weights. Storing an 8-billion parameter model in FP16 requires $8 \text{B} \times 2 \text{ bytes} = 16\text{GB}$ of physical VRAM.
* **Quantization:** Compresses these weights down to lower bit representations (such as **INT4** or **INT8**: 4-bit or 8-bit integers) using the **GGUF** file format.
* **Impact:** Compressing weights via 4-bit quantization reduces memory requirements to ~4.5GB while preserving >95% of the original model's reasoning capabilities, allowing high-performance models to run smoothly on standard consumer laptops.

---

### C. Local Model Selection Guide

Depending on your local developer workstation hardware profile, select the appropriate model size:

#### 1. Low-End Workstations (8GB unified RAM, standard CPU / Integrated GPU)
* **Llama 3.2 (1B or 3B):** (Meta) Incredibly fast generation speeds (>40 t/s) with outstanding contextual reading comprehension for its tiny footprint.
* **Qwen 2.5 (1.5B or 3B):** (Alibaba) Highly optimized for multilingual translation, formatting, and structured coding assistance.
* **Phi-3.5 Mini (3.8B):** (Microsoft) Exceptionally strong logical reasoning and mathematical problem-solving benchmarks.

#### 2. Mid-End Workstations (16GB-32GB RAM, dedicated GPU with 8GB-12GB VRAM / Mac M1/M2/M3)
* **Llama 3.1 (8B):** The standard champion for local development. Possesses highly robust system instructions execution, general reasoning, and structured JSON generation.
* **Mistral (7B) / Codestral (22B Q4):** Optimized specifically for multi-step software engineering logic, code compilation, and tool-calling execution loops.
* **Qwen 2.5 (7B or 14B):** Outstanding code-generation performance, matching many larger 70B models on active benchmarks.

---

## 6. Popular Interview Questions & High-Impact Answers

### Q1: What is the "auto-regressive" nature of LLMs, and how does it impact computational latency and billing?
* **Answer:** LLMs are auto-regressive, meaning they generate text sequentially, predicting exactly one token at a time. Every generated token is fed back into the model's input context to calculate the next one. This creates two distinct latency phases:
  1. **Time to First Token (TTFT) / Prefill:** Fast. The model processes the entire input prompt in a single parallel step.
  2. **Inter-Token Latency (ITL) / Generation:** Slow. Each output token requires a complete, sequential forward pass through the entire network, leading to linear latency scaling ($O(N)$) based on output length.
  Billing is charged per input token (cheap, processed once) and per output token (expensive, due to sequential forward-pass compute overhead).

### Q2: How does the Temperature hyperparameter alter the randomness of LLM outputs under the hood?
* **Answer:** Temperature scales the raw scores (logits) generated by the final layer of the network before applying the Softmax function. Mathematically, logits are divided by the temperature value $T$. When $T$ is very low (e.g., $0.1$), the mathematical difference between the highest logit score and all others is drastically magnified, causing the Softmax probability of the top token to approach $1.0$ (highly deterministic). When $T > 1$, the logit distribution is flattened, reducing the probability gap and allowing the model to randomly sample less likely tokens (creative/random).

### Q3: What is the difference between JSON Mode and Structured Outputs (Constrained Decoding) in production APIs?
* **Answer:** **JSON Mode** only guarantees that the generated string is syntactically valid JSON. It does not enforce any specific schema, so keys can be named incorrectly, be missing, or contain wrong data types. **Structured Outputs** (via Constrained Decoding) injects the required JSON schema directly into the token sampling layer. The inference engine physically blocks (masks out) any tokens that would violate the schema at runtime, guaranteeing 100% schema-compliant JSON structure and data types.
