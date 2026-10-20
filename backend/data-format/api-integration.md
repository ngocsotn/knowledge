# Using Data Formats in APIs

An API format is part of its contract: clients and servers must agree on how request and response bodies are encoded, identified, validated, and evolved.

## HTTP representation headers

- `Content-Type` identifies the representation format of the current body, for example `application/json` or `application/xml`.
- `Accept` lets a client state which response media types it can receive. A server may select a supported representation or return `406 Not Acceptable`.
- `Content-Encoding` describes a content coding such as gzip; it is not the same as the media type.

For HTTP protocol behavior, see [HTTP](../../network/protocols/http/index.md). For API architecture and design, see [Backend API](../api/index.md).

## Choosing a representation

JSON is a practical default for new APIs with broad client support and object-shaped data. XML may be required by an external contract or useful where namespaces and XML tooling are established. JSONL/NDJSON can serve large exports or streaming endpoints when clients can process records incrementally.

Do not switch representation without considering existing clients. If an endpoint supports multiple media types, document negotiation, error behavior, and whether representations have identical semantics.

## Contract and validation

Validate media type, body size, syntax, schema, and business rules. Return clear client errors for invalid input; do not treat a parser failure as an empty/default object. Document required and optional fields, nullability, numeric ranges, date representation, unknown-field behavior, and versioning.

Treat serialization as an API boundary. Avoid leaking internal object fields by serializing domain objects directly without a deliberate response model. Do not assume a JSON or XML parser validates authorization or business constraints.

### Q1: What is the difference between `Content-Type` and `Accept`?
* Answer: `Content-Type` describes the current request or response body. `Accept` communicates which response media types the client can handle.

### Q2: When might an API use JSONL/NDJSON?
* Answer: For large or streamed collections where clients should consume complete records incrementally rather than buffer one large response.

### Q3: Why should APIs use explicit response models?
* Answer: They prevent accidental exposure of internal fields and give the API a stable, intentional contract independent of internal object changes.
