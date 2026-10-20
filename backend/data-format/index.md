# Data Formats

JSON, JSON Lines (JSONL/NDJSON), and XML represent structured data in text. They solve similar exchange problems but differ in structure, streaming behavior, compatibility, and ecosystem support.

## Guides

- [JSON](./json.md)
- [JSON Lines: JSONL and NDJSON](./jsonl-ndjson.md)
- [XML](./xml.md)
- [Using Formats in APIs](./api-integration.md)
- [Files and Streaming](./file-and-streaming.md)

## Compare the formats

| Format | Structure | Natural unit | Streaming fit | Common uses |
| --- | --- | --- | --- | --- |
| JSON | Objects, arrays, and scalar values | One complete JSON value | Possible with streaming parsers; a top-level array is not naturally record-delimited | APIs, configuration, messages |
| JSONL / NDJSON | One JSON value per line | One line / record | Strong; process records incrementally | Logs, exports, bulk ingestion |
| XML | Nested elements, attributes, and text | One XML document or a parsed event/node | Strong with SAX/StAX-style parsers; whole-document trees use more memory | Document exchange, legacy and enterprise integrations |

JSON is usually the simplest choice for new web APIs. Choose JSONL/NDJSON when records should be independently processed as a stream. Choose XML when an integration requires XML, or when namespaces, mixed content, or established XML schemas matter.

These are text formats, not transport protocols. HTTP carries them as representations with media types; see [HTTP](../../network/protocols/http/index.md). For API design details, see [Backend API](../api/index.md). For general file operations, see [Backend File](../file/index.md).

Language libraries affect parsing and serialization details. See [Node.js and JavaScript](../../specific-language/nodejs/index.md), [TypeScript](../../specific-language/nodejs/typescript/index.md), and [.NET data access](../../specific-language/dotnet/data-access/index.md) for related runtime material.

## Shared design concerns

- **Syntax is not a schema.** A document can parse successfully but still violate application rules. Validate required fields, types, ranges, and cross-field constraints separately.
- **Define compatibility rules.** Decide how readers handle missing, unknown, and renamed fields before changing a format.
- **Treat input as untrusted.** Bound input size and nesting depth, handle parser errors explicitly, and avoid unsafe XML entity resolution.
- **Know the trade-offs.** Text is inspectable and broadly supported, but can be larger and slower to parse than compact binary formats.

### Q1: How should you choose between JSON, JSONL/NDJSON, and XML?
* Answer: Start with the consumers and integration contract. JSON fits general APIs, JSONL/NDJSON fits independent streamed records, and XML fits contracts that require XML features or existing XML tooling.

### Q2: Does successful parsing mean data is valid?
* Answer: No. Parsing checks format syntax. Application-level schema and business-rule validation are separate.

### Q3: Are these formats transport protocols?
* Answer: No. They encode data. A protocol such as HTTP transports the representation and can identify it with a media type.
