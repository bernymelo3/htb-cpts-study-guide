# THEORY — Intro to XXE

## ID
213

## Module
Web Attacks

## Kind
notes

## Title
Section 13 — Intro to XXE

## Description
Introduces XML fundamentals (structure, DTDs, entities) and explains how XML External Entity injection arises when user-controlled XML is parsed without sanitization, enabling file disclosure and other server-side attacks.

## Tags
theory, xxe, xml, injection, owasp-top-10, web-attacks

## TL;DR — What's Important
- **XXE = abusing external entities in XML:** If the server parses user-supplied XML and resolves external entities, an attacker can read local files, make SSRF requests, or DoS the server.
- **The attack surface is the DTD:** Custom `<!ENTITY>` declarations with the `SYSTEM` or `PUBLIC` keyword tell the parser to fetch external resources — that's the injection point.
- **Three XML building blocks to know:** Tags (structural keys), Entities (variables referenced as `&name;`), and DTDs (the schema that declares elements and entities).
- **Internal vs. external entities:** Internal entities hold a literal value; external entities use `SYSTEM "file:///..."` or `SYSTEM "http://..."` to pull content from the server's filesystem or network.
- **OWASP Top 10 risk:** XXE is ranked as a top web security risk because a single vulnerable XML endpoint can leak sensitive files or pivot into SSRF.

## Concept Overview
XML External Entity (XXE) injection exploits the XML specification's support for external entities. When an application accepts XML input and the underlying parser resolves entity references, an attacker can define a custom DTD that points an entity at a local file (`file:///etc/passwd`) or an internal URL (`http://169.254.169.254/...`). The parser fetches the resource and substitutes its content into the XML document, which may then be reflected back to the attacker.

## Key Concepts

### Definitions
- **XML (Extensible Markup Language)** — A markup language for structured data transfer and storage, built around nested element trees with a root element.
- **DTD (Document Type Definition)** — A schema that validates an XML document's structure; can be inline or referenced externally via `SYSTEM` or a URL.
- **Entity** — An XML variable declared in a DTD and referenced as `&name;` in the document body; the parser replaces the reference with the entity's value.
- **Internal Entity** — Entity whose value is a literal string defined directly in the DTD (e.g., `<!ENTITY company "Inlane Freight">`).
- **External Entity** — Entity whose value is fetched from a file path or URL via the `SYSTEM` keyword (e.g., `<!ENTITY secret SYSTEM "file:///etc/passwd">`).
- **PCDATA (Parsed Character Data)** — Raw text content within an XML element that the parser processes for entity and markup resolution.

### XML Document Anatomy
| Component | Syntax | Purpose |
|---|---|---|
| Declaration | `<?xml version="1.0" encoding="UTF-8"?>` | Defines XML version and encoding |
| DTD (inline) | `<!DOCTYPE root [ ... ]>` | Declares structure and custom entities |
| DTD (external) | `<!DOCTYPE root SYSTEM "schema.dtd">` | References an external DTD file or URL |
| Element | `<tag>value</tag>` | Data container; nests to form the tree |
| Entity reference | `&name;` | Replaced by the parser with the entity's value |
| Special chars | `&lt; &gt; &amp; &quot;` | Escape reserved XML characters |

### The XXE Attack Chain (Conceptual)
- **Step 1 — Identify XML input:** Look for SOAP APIs, XML-based web forms, file upload endpoints that accept XML/SVG/DOCX.
- **Step 2 — Inject a DTD with an external entity:** Define `<!ENTITY xxe SYSTEM "file:///etc/passwd">` in a custom DOCTYPE.
- **Step 3 — Reference the entity:** Place `&xxe;` where the application reflects content back (e.g., a displayed field).
- **Step 4 — Read the response:** If the entity resolves and the value is reflected, the file contents are disclosed.

## Why It Matters
XXE is deceptively common because XML parsing libraries often enable external entity resolution by default. Any endpoint that consumes XML — REST/SOAP APIs, document uploads (DOCX, XLSX, SVG are all XML-based), or configuration imports — is a potential target. A single vulnerable parser can yield arbitrary file read, SSRF to cloud metadata endpoints, or denial of service via recursive entity expansion (the "Billion Laughs" attack).

## Defender Perspective
- **Primary mitigation:** Disable external entity resolution and DTD processing in the XML parser (e.g., `libxml_disable_entity_loader(true)` in PHP, `FEATURE_SECURE_PROCESSING` in Java).
- **Input validation:** Reject XML input containing `<!DOCTYPE` or `<!ENTITY` declarations when custom DTDs are not needed.
- **Detection:** WAF rules for DTD/ENTITY keywords in XML bodies; monitor for unexpected outbound connections from the XML parsing service.
- **MITRE ATT&CK:** Maps to T1190 (Exploit Public-Facing Application).

## Key Takeaways
- The `SYSTEM` keyword in an entity declaration is what makes XXE possible — it tells the parser to fetch an external resource on the server's behalf.
- Both `SYSTEM` and `PUBLIC` can be used for external entity loading; most XXE payloads use `SYSTEM` but `PUBLIC` works in many parsers too.
- XML special characters (`<`, `>`, `&`, `"`) must be entity-encoded in document content, but this encoding does not prevent XXE — the injection happens in the DTD, not the element text.
- File formats like DOCX, XLSX, and SVG are XML under the hood, making them viable XXE vectors even when the application doesn't explicitly accept "XML."

## Gotchas
- Some parsers silently ignore external entities instead of throwing errors — the absence of an error does not mean the app is safe; you may need blind/OOB techniques (covered in later sections).
- `file:///` works for local file read, but `http://` entities enable SSRF to internal services and cloud metadata APIs — always test both protocols.
- Inline DTDs override external DTDs when both are present, which is useful for injecting entities even when the app specifies its own DTD.
