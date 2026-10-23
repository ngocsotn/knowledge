# XML

XML (Extensible Markup Language) represents data with nested elements, attributes, and text. It is used in document formats, enterprise integrations, and systems with established XML-based contracts.

```xml
<order id="o-104">
  <customer>Ada</customer>
  <items>
    <item sku="book-1" quantity="2"/>
  </items>
</order>
```

Elements can contain child elements, text, or both. Attributes hold metadata about an element. Namespaces distinguish names that would otherwise collide across vocabularies.

## Well-formed, valid, and typed

- **Well-formed** XML follows syntax rules: one root element, properly nested tags, quoted attributes, and escaped reserved characters.
- **Valid** XML conforms to a declared schema such as XSD or a DTD, if the application uses one.
- **Application-valid** XML also satisfies business rules that a parser or schema may not capture.

These checks are distinct. A parser can build a tree from well-formed XML that is still invalid for the application.

XML supports comments, processing instructions, CDATA sections, and mixed content. These features help with document-oriented data but can make conversion to simple object models lossy. Namespace prefixes are aliases; namespace URIs identify names.

## Strengths and costs

- Mature standards and tooling, namespaces, and schema-validation options.
- Can represent document-centric and mixed-content structures naturally.
- Verbose syntax and more complex parsing than typical JSON APIs.
- Multiple equivalent representations and namespace handling can complicate comparisons and transformations.
- Tree-building parsers can use memory proportional to document size; event-based parsers allow incremental processing.

Choose XML when a partner, standard, or existing ecosystem requires it, or when its document and namespace features are valuable. Avoid choosing it for a new simple API solely because it appears more expressive; define and support only the needed features.

## Security

XML parsers must be configured safely for untrusted input. Disable external entity resolution and external DTD loading unless explicitly required, and limit entity expansion, nesting depth, and total document size. Unsafe entity processing can expose local resources or cause excessive resource use.

For API transport and file-processing considerations, see [Using Formats in APIs](./api-integration.md) and [Files and Streaming](./file-and-streaming.md).

### Q1: What is the difference between well-formed and valid XML?
* Answer: Well-formed XML follows XML syntax. Valid XML also satisfies a declared schema or DTD.

### Q2: Why use XML namespaces?
* Answer: Namespaces distinguish element and attribute names from different vocabularies using namespace URIs.

### Q3: What is a key XML parser security risk?
* Answer: Unsafe external entity or DTD processing can read external resources or consume excessive resources. Disable it for untrusted input unless required.
