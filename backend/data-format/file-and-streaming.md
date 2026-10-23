# Data Format Files and Streaming

Format choice affects memory use, recovery, validation, and how safely an application writes or reads data. A large text document is not automatically safe to load into memory as one value.

## Reading large inputs

- **JSON document:** A regular parser often builds the complete object tree. Use a streaming parser or a record-oriented format when the document may exceed available memory.
- **JSONL/NDJSON:** Read one line at a time, then parse and validate that record. Bound line length; a single malicious or accidental huge line can still exhaust resources.
- **XML:** A DOM parser builds a tree; SAX/StAX-style event parsing processes nodes incrementally. Configure parser limits and disable unsafe external entity processing for untrusted input.

Incremental parsing reduces peak memory, but does not remove the need for input-size limits, validation, error reporting, and backpressure in a producer-consumer pipeline.

## Writing and exchanging files

Agree on encoding (usually UTF-8), schema/version, newline convention where relevant, and whether a final newline is expected. File extensions are useful hints, not proof that content is valid. Validate data at trust boundaries.

For generated files, write to a temporary file in the target filesystem and rename into place after successful completion when atomic replacement is required and supported. This avoids exposing partially written output. Define cleanup and retry behavior for failures.

For independently recoverable records, JSONL/NDJSON can make append and replay workflows simpler than rewriting one large JSON array. It does not itself provide transactions, deduplication, checksums, or indexes; add these only when the application requires them.

See [Backend File](../file/index.md) for file-handling fundamentals and [JSONL/NDJSON](./jsonl-ndjson.md) for line-oriented record rules.

### Q1: Does streaming parsing make input safe?
* Answer: No. It lowers memory pressure, but applications still need size and complexity limits, validation, error handling, and appropriate parser security settings.

### Q2: Why might JSONL/NDJSON work better than one large JSON file for exports?
* Answer: Consumers can process and recover individual records incrementally. The format does not provide transactions or deduplication automatically.

### Q3: How can a writer avoid exposing a partially generated file?
* Answer: Write to a temporary file and rename it into place after successful completion, when the filesystem and application requirements support atomic replacement.
