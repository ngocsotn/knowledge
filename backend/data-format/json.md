# JSON

JSON (JavaScript Object Notation) is a text format for structured values. Despite its name, it is language-independent and is commonly used for HTTP APIs, configuration, and messages.

## Data model and syntax

JSON supports objects, arrays, strings, numbers, booleans, and `null`.

```json
{
  "id": 42,
  "name": "Ada",
  "active": true,
  "roles": ["reader", "editor"],
  "manager": null
}
```

An object maps string keys to values; an array is an ordered sequence. JSON has no native date, binary, decimal, or `undefined` type. Applications need an explicit convention for such values, such as an ISO 8601 string for a timestamp.

Standard JSON does not allow comments, trailing commas, `NaN`, or `Infinity`. Object member names must be strings. Interoperability can suffer when numbers exceed the exact integer range of a consumer's numeric type; identifiers are often safer as strings.

## Why use it?

- Human-readable and supported by nearly every modern language.
- Maps naturally to common in-memory objects, arrays, and scalar values.
- Compact enough for most APIs and easy to inspect while debugging.

Costs and limits:

- Text parsing and repeated field names add CPU and bandwidth overhead compared with many binary encodings.
- The basic format does not define schemas, references, comments, or a date type.
- Duplicate object names are syntactically possible but handled inconsistently by parsers; producers should avoid them.
- A single JSON document needs a framing rule when many values share one stream. See [JSONL/NDJSON](./jsonl-ndjson.md).

## Validation and compatibility

Parsing answers whether input is valid JSON; it does not establish whether it is valid for an application. Validate required fields, types, ranges, and business constraints after parsing. A schema system such as JSON Schema can formalize structural constraints, but application rules may still need code.

When evolving an API, readers should generally tolerate unknown fields where safe, and producers should avoid changing field meaning or type in place. Define how missing values differ from explicit `null`.

## APIs and files

HTTP APIs commonly use `Content-Type: application/json` for JSON request and response bodies. File extensions such as `.json` are conventions, not validation. Use UTF-8 and reject or report malformed input rather than silently substituting defaults.

For content negotiation and API concerns, see [Using Formats in APIs](./api-integration.md). For large files and incremental parsing, see [Files and Streaming](./file-and-streaming.md).

### Q1: What value types does JSON define?
* Answer: Objects, arrays, strings, numbers, booleans, and `null`. Dates and binary data need application conventions.

### Q2: Does JSON preserve object member order?
* Answer: Consumers should not rely on object order for meaning. Arrays are ordered; objects are mappings.

### Q3: What is the difference between parsing and validation?
* Answer: Parsing checks JSON syntax and produces a value. Validation checks whether that value matches an expected schema and application rules.
