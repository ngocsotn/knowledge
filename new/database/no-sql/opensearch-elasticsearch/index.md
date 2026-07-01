# OpenSearch & Elasticsearch Architecture

Under the hood guide covering inverted indexes, segment search, sharding, replicas, and relevance scoring.

## Search Internals

### 1. Inverted Index
Traditional databases map rows to columns. Search engines map analyzed tokens/words to document IDs (Inverted Index). This allows near-instantaneous full-text keyword lookups.

### 2. Segment
Indexes are divided into multiple on-disk immutable files called Segments.
- **Immutability:** Segments are never modified. New writes create new segments. Background thread runs "Segment Merging" to clean deleted documents and merge files.

### 3. Shard & Replica
- **Primary Shard:** A self-contained instance of Apache Lucene. Indexes are split across primary shards to scale horizontally.
- **Replica Shard:** A redundant copy of a primary shard to guarantee high availability and scale read throughput.

### 4. Search Relevance (TF-IDF & BM25)
- **Term Frequency (TF):** How often a keyword appears in a document.
- **Inverse Document Frequency (IDF):** How rare the keyword is across all documents.
- **BM25:** Modern standard relevance ranking algorithm adjusting TF-IDF based on document length saturation.

## Interview Questions & Answers

### Q1: Why are Elasticsearch segments immutable, and how are updates handled?
- **Answer:** Immutability removes lock contention (no concurrency locks needed during searches), allows the OS page cache to cache segment files heavily, and enables efficient search filters compression. 
  - To handle **Updates**, Elasticsearch writes a new segment containing the updated document, and marks the old document version as "deleted" in a separate `.del` bit-vector file. During searches, the deleted documents are simply filtered out of the results. Deleted documents are physically purged from disk during background **Segment Merging**.
