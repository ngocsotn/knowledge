# JSON Lines: JSONL and NDJSON

JSON Lines (often `.jsonl`) and Newline Delimited JSON (often `.ndjson`) store a sequence of JSON values, with one value per line. The line boundary frames each record, so a reader can process a large stream without loading it all into memory.

```jsonl
{"event":"login","userId":"u-17"}
{"event":"purchase","userId":"u-17","amount":12.5}
{"event":"logout","userId":"u-17"}
```

JSONL and NDJSON commonly refer to the same line-oriented pattern, though projects may define small convention differences. Agree on the exact contract, including extension, media type, blank-line handling, and whether the final newline is required.

## Record rules

- Each non-empty line is a complete JSON value, commonly an object.
- A raw newline cannot appear inside a JSON string; encode it as `\n`. This keeps physical line boundaries unambiguous.
- Do not add commas between records. Each line parses independently.
- A final newline is useful but not universally required. Readers should follow the producer-consumer contract.
- Blank lines and comments are not JSON values; accept or reject them only according to an explicit contract.

## When to use it

Use JSONL/NDJSON for append-friendly logs, exports, bulk imports, and pipelines where each record can be handled independently. A consumer can read one line, parse and validate it, then release it before reading the next.

It is less suitable when records depend on document-wide structure, when consumers need random access without an index, or when an all-or-nothing transaction across the complete file is required. A malformed record need not invalidate neighboring lines, but recovery behavior must be designed explicitly.

## Robust processing

Set a maximum line size and handle malformed records with a deliberate policy: stop, quarantine the line, or report and continue. Do not silently skip invalid data. Include stable identifiers and versioning where consumers must deduplicate, replay, or evolve records.

Some systems use `application/x-ndjson` as a media type, but media-type support varies. Confirm what the integration expects instead of assuming a universal registration.

Compare with a top-level JSON array in [JSON](./json.md), and see [Files and Streaming](./file-and-streaming.md) for stream processing patterns.

### Q1: Why is JSONL/NDJSON easier to stream than one JSON array?
* Answer: Each line frames one complete value, so consumers can parse records incrementally without waiting for the closing bracket or buffering the whole array.

### Q2: Can a JSONL record contain a newline?
* Answer: A JSON string can represent a newline as the escaped sequence `\n`; a raw newline would end the record and make the line invalid.

### Q3: What should a reader do with a malformed line?
* Answer: Follow an explicit recovery policy, such as failing the import or quarantining and reporting the record. Silent skipping risks data loss.
