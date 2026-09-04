# LLM & AI Interview Questions & Answers

Interview questions related to Large Language Models, AI systems, and knowledge management encountered during frontend engineering interviews.

## Table of Contents

- [Q1: How to Handle LLM Knowledge Base Storage Inefficiency](#q1-how-to-handle-llm-knowledge-base-storage-inefficiency)

---

## Q1: How to Handle LLM Knowledge Base Storage Inefficiency

### The Problem

You encounter requirements during development that are valuable enough to persist as institutional knowledge — business rules, domain constraints, edge cases, architectural decisions. The naive approach is to store every such requirement verbatim into the LLM's knowledge base (e.g., via Retrieval-Augmented Generation context, fine-tuning data, or persistent memory stores).

However, indiscriminately storing every requirement of this caliber causes **storage inefficiency**:
- **Token bloat:** RAG retrieval windows are limited (context windows range from 4K to 200K tokens). Filling them with verbose, overlapping, or low-signal entries degrades retrieval quality.
- **Retrieval noise:** More entries increase the probability of retrieving irrelevant context, diluting the signal-to-noise ratio.
- **Embedding collision:** Semantically similar but subtly different entries produce nearly identical vector embeddings, making it harder for the retrieval system to distinguish between them.
- **Maintenance overhead:** Knowledge bases without pruning or organization become stale, contradictory, and increasingly unreliable over time.

### Solution Architecture: Hierarchical Knowledge Compression

Instead of storing every requirement at the same granularity, implement a **tiered knowledge architecture** that balances recall with storage efficiency:

```
┌──────────────────────────────────────────────────────────────┐
│                  KNOWLEDGE TIER ARCHITECTURE                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  TIER 1: Core Principles (Always in Context)                 │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Compressed, high-level rules and invariants.          │  │
│  │  Example: "All monetary calculations use BigDecimal."  │  │
│  │  Storage: System prompt / pinned context               │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  TIER 2: Domain Summaries (Retrieved On-Demand)              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Condensed summaries grouped by domain/feature.        │  │
│  │  Example: "Billing domain: 14 rules (see details)"     │  │
│  │  Storage: RAG vector store, chunked by domain          │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  TIER 3: Granular Requirements (Drill-Down Access)           │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Full verbatim requirements with examples and edge     │  │
│  │  cases. Only loaded when specifically queried.         │  │
│  │  Storage: Document store / file system / database      │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Strategy 1: Semantic Deduplication

Before inserting a new requirement into the knowledge base, compute its embedding and compare it against existing entries using cosine similarity. If the similarity exceeds a threshold (e.g., 0.92), **merge** the new requirement into the existing entry rather than creating a duplicate.

```
New requirement: "Prices must always be rounded to 2 decimal places."
Existing entry:  "All monetary values displayed to users must be rounded to two decimals."

Cosine similarity: 0.96 → MERGE (update existing entry to include both contexts)
```

### Strategy 2: Abstractive Compression

Use the LLM itself to compress verbose requirements into concise rules. Store the compressed version in Tier 1/2 and keep the original in Tier 3 for drill-down.

```
ORIGINAL (142 tokens):
"When a user submits a refund request for an order that was placed more than 30 days ago,
the system must check whether the product category is eligible for extended refund windows.
Electronics have a 14-day window, clothing has a 45-day window, and digital goods are
non-refundable after 7 days. If the window has passed, display error code REFUND_EXPIRED."

COMPRESSED (38 tokens):
"Refund eligibility: Electronics 14d, Clothing 45d, Digital 7d. Past window → REFUND_EXPIRED."
```

### Strategy 3: Structured Knowledge Schemas

Instead of storing free-text blobs, structure each knowledge entry with metadata that enables efficient filtering and retrieval:

```json
{
  "id": "REQ-BILLING-042",
  "domain": "billing",
  "type": "business_rule",
  "priority": "critical",
  "summary": "Refund windows are category-specific",
  "rule": "Electronics: 14d, Clothing: 45d, Digital: 7d",
  "detail_ref": "docs/billing/refund-policy.md",
  "created": "2026-08-15",
  "supersedes": ["REQ-BILLING-038"],
  "tags": ["refund", "eligibility", "time-window"]
}
```

The RAG system retrieves based on `domain` + `tags` + `type` filtering before performing semantic search, dramatically reducing the search space.

### Strategy 4: Knowledge Decay & Active Pruning

Implement a **decay function** on knowledge entries. Requirements that haven't been accessed or validated in N months are flagged for review. A periodic curation pass (manual or LLM-assisted) prunes outdated entries, merges duplicates, and promotes frequently-accessed entries to higher tiers.

```
Access frequency: High → Promote to Tier 1 (always in context)
Access frequency: Medium → Keep in Tier 2 (RAG retrieval)
Access frequency: Low (6+ months) → Archive to Tier 3 or deprecate
Contradicted by newer entry → Supersede and archive
```

### Strategy 5: Retrieval-Augmented Generation (RAG) Optimization

Rather than expanding the knowledge base, improve **retrieval precision**:
- **Chunking strategy:** Split knowledge by semantic boundaries (per-rule, per-domain), not by fixed character counts.
- **Hybrid retrieval:** Combine vector similarity search with keyword BM25 search for higher recall.
- **Re-ranking:** After initial retrieval, use a cross-encoder model to re-rank results by relevance to the current query, filtering out low-signal entries before injecting into the LLM context window.
- **Contextual compression:** After retrieval, use a lightweight LLM to compress retrieved chunks into only the relevant sentences before injecting into the main LLM's prompt.

### Strategy 6: Tool-Based Knowledge Access (Agentic Pattern)

Instead of stuffing all knowledge into the context window, give the LLM **tool access** to a knowledge API. The LLM decides when it needs to look something up and queries only the relevant entries:

```
LLM reasoning: "The user is asking about refund logic. Let me check the knowledge base."
Tool call: searchKnowledgeBase({ domain: "billing", query: "refund eligibility window" })
Result: [REQ-BILLING-042] → injected into context only when needed
```

This keeps the base context window lean and only loads knowledge on demand — the same principle as lazy loading in frontend applications.

### Key Takeaway
The solution is not "store less" — it is **store smart**. Use hierarchical tiers (compressed principles → domain summaries → full detail), deduplicate semantically, structure entries with metadata for filtered retrieval, implement decay-based pruning, and leverage tool-based access patterns so the LLM loads knowledge lazily rather than carrying everything in its context window at all times.
