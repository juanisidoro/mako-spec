# MAKO Specification v0.1.0

**Status:** Draft
**Last Updated:** 2026-02-16

> **Versioning note:** `v0.1.0` is the version of this *specification document*. The protocol version used in MAKO frontmatter is `"1.0"` (the `mako` field). These are distinct: the document version tracks spec revisions, while the protocol version identifies the format that agents negotiate. When the protocol itself introduces breaking changes, the frontmatter version will increment.

## 1. Introduction

MAKO (Markdown Agent Knowledge Optimization) is an open standard that defines how web pages serve semantically optimized content to AI agents. Unlike auto-conversion approaches (HTML-to-Markdown), MAKO provides a per-page, purpose-built representation designed for maximum LLM comprehension with minimum token usage.

### 1.1 Goals

- Define a per-page content format optimized for LLM consumption
- Enable pre-download relevance filtering via compact embeddings in HTTP headers
- Declare available actions, semantic links, and metadata alongside content
- Minimize token usage while maximizing information density
- Complement existing standards (WebMCP, llms.txt) without replacing them

### 1.2 Non-Goals

- Replace HTML for human consumption
- Define how LLMs should process the content internally
- Mandate specific embedding models (MAKO defines the transport format, not the model)
- Replace WebMCP for action execution in browsers

## 2. MAKO File Format

A MAKO file is a UTF-8 encoded Markdown document with a YAML frontmatter block. The file extension is `.mako.md`.

### 2.1 Structure

```
---
[YAML frontmatter]
---

[Markdown body]
```

Producers MAY include YAML comments in the frontmatter block for identification:

```yaml
---
# @mako — Machine-Accessible Knowledge Object
# Spec: https://makospec.vercel.app
mako: "1.0"
...
---
```

Comments are ignored by parsers and have no semantic meaning.

### 2.2 Frontmatter (Required Fields)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mako` | string | Yes | Spec version (e.g., `"1.0"`) |
| `type` | string | Yes | Content type (see Section 5) |
| `entity` | string | Yes | Primary entity described by this page |
| `updated` | string (ISO 8601) | Yes | Last content update date |
| `tokens` | integer | Yes | Estimated token count of the body |
| `language` | string (BCP 47) | Yes | Content language (e.g., `"en"`, `"es"`) |

### 2.3 Frontmatter (Optional Fields)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `embedding-model` | string | No | CEF model used (e.g., `"mako-cef-v1"`) |
| `canonical` | string (URL) | No | Canonical URL of the HTML version |
| `summary` | string | No | One-line summary (max 160 chars) |
| `actions` | array | No | Available actions (see Section 3) |
| `links` | object | No | Semantic links (see Section 4) |
| `related` | array | No | Related page paths |
| `tags` | array | No | Content tags/categories |
| `audience` | string | No | Target audience (e.g., `"developers"`, `"consumers"`) |
| `freshness` | string | No | Content freshness: `"realtime"`, `"hourly"`, `"daily"`, `"weekly"`, `"monthly"`, `"static"` |
| `media` | object | No | Available media on the source page (see Section 2.5) |

### 2.5 Media Metadata

The `media` field describes what media types are available on the source HTML page. It serves two purposes: a representative image (`cover`) that agents can display directly, and counts that convey the visual richness of the source page.

#### Schema

```yaml
media:
  cover:                # Optional. ONE representative image of the page
    url: string         # Relative or absolute URL to the image
    alt: string         # Semantic description of the image
  images: integer       # Total content images (excluding icons/UI)
  video: integer        # Total video elements
  audio: integer        # Total audio elements
  interactive: integer  # Interactive elements (configurators, calculators, 3D viewers)
  downloads: integer    # Downloadable files (PDFs, datasets)
```

All fields within `media` are optional. Only include count fields with non-zero values. Counts refer to the source HTML page, not the MAKO representation.

#### Cover Image

The `cover` field provides a single representative image that an agent can display as a thumbnail or preview in a conversational interface. Only one image — if the agent needs more, it has `canonical` to access the full page.

The cover source depends on the content type:

| Type | Cover Source |
|------|-------------|
| `product` | Product featured image |
| `article` | Featured image / post thumbnail |
| `landing` | Hero banner or `og:image` |
| `listing` | Category image or first product thumbnail |
| `recipe` | Photo of the finished dish |
| `profile` | Avatar / profile photo |
| `event` | Poster or event banner |
| `docs` / `faq` | Generally none (omit `cover`) |

#### Examples

**Product:**

```yaml
media:
  cover:
    url: /uploads/bolso-roma-frontal.webp
    alt: "Bolso Bandolera Roma - Piel Natural"
  images: 5
  video: 1
```

**Article:**

```yaml
media:
  cover:
    url: /uploads/guia-running-2026.webp
    alt: "Guía completa de zapatillas running 2026"
  images: 8
```

**Landing:**

```yaml
media:
  cover:
    url: /uploads/banner-nueva-coleccion.webp
    alt: "Nueva Colección Primavera Verano 2026"
  images: 47
  video: 1
  interactive: 2
```

**Docs (no cover):**

```yaml
media:
  images: 3
```

### 2.4 Markdown Body

The body MUST follow these principles:

1. **Lead with the most important information** — The first paragraph should answer the primary query an agent might have about this page.
2. **Use structured sections** — Headings (`##`) to organize information logically.
3. **Prefer lists and key-value pairs** — Over prose paragraphs when presenting factual data.
4. **Include context and comparisons** — Help the LLM understand relative positioning (competitors, alternatives, trade-offs).
5. **Omit navigation, legal boilerplate, and UI text** — Only include substantive content.
6. **Keep it concise** — Never exceed 1,000 tokens. Recommended ranges by type:

| Type | Tokens | Rationale |
|------|--------|-----------|
| `product` | 200–400 | Facts, pricing, context. No marketing fluff. |
| `article` | 300–500 | Summary, key points, context. |
| `landing` | 300–500 | Value prop, features, pricing, alternatives. |
| `listing` | 200–400 | Overview, top items, filters. |
| `recipe` | 300–500 | Ingredients, steps, notes. |
| `profile` | 150–300 | Bio, key info, notable work. |
| `event` | 200–300 | Date, location, description, registration. |
| `docs` | 300–500 | Overview, usage, parameters. |
| `faq` | 200–400 | Question-answer pairs. |
| `custom` | 200–500 | Adapt to content. |

## 3. Actions

Actions declare what operations an AI agent can perform related to this page. They bridge MAKO content with WebMCP or direct API calls.

### 3.1 Action Schema

```yaml
actions:
  - name: string          # Required. Unique identifier (snake_case)
    description: string    # Required. Natural language description
    endpoint: string       # Optional. API endpoint URL
    method: string         # Optional. HTTP method (default: POST)
    webmcp: boolean        # Optional. If true, action is available via WebMCP
    params:                # Optional. Expected parameters
      - name: string
        type: string
        required: boolean
        description: string
```

### 3.2 Example

```yaml
actions:
  - name: add_to_cart
    description: "Add this product to the shopping cart"
    endpoint: /api/cart/add
    method: POST
    params:
      - name: quantity
        type: integer
        required: false
        description: "Number of items (default: 1)"

  - name: check_availability
    description: "Check stock availability by size"
    endpoint: /api/stock
    method: GET
    params:
      - name: size
        type: string
        required: true
        description: "Size to check (e.g., '42', 'L')"
```

## 4. Semantic Links

Links in MAKO are not just URLs — they include **context** that helps the LLM understand the relationship.

### 4.1 Link Schema

```yaml
links:
  internal:               # Links within the same domain
    - url: string         # Required. Path or full URL
      context: string     # Required. Why this link is relevant
      type: string        # Optional. Relationship type
  external:               # Links to other domains
    - url: string
      context: string
      type: string

related:                  # Shorthand for closely related pages (same domain)
  - /path/to/page
```

### 4.2 Relationship Types

| Type | Description |
|------|-------------|
| `parent` | Parent category or section |
| `child` | Sub-page or detail page |
| `sibling` | Same-level alternative |
| `source` | Original source of information |
| `competitor` | Competing product/service |
| `reference` | Supporting documentation |

### 4.3 Example

```yaml
links:
  internal:
    - url: /category/running-shoes
      context: "Browse all running shoes in this category"
      type: parent
    - url: /product/nike-pegasus-40
      context: "Similar shoe, lighter weight, higher price"
      type: sibling
  external:
    - url: https://nike.com/air-max-90
      context: "Official manufacturer product page"
      type: source

related:
  - /product/adidas-ultraboost
  - /product/new-balance-1080
  - /guides/choosing-running-shoes
```

## 5. Content Types

MAKO defines standard content types. Each type has recommended sections for the markdown body.

### 5.1 `product`

```markdown
# [Product Name]
[One-line description]

## Key Facts
[Price, availability, specs as list]

## Context
[Competitors, positioning, trade-offs]

## Reviews Summary
[Aggregated sentiment]
```

### 5.2 `article`

```markdown
# [Title]
[Author, date, reading time]

## Summary
[2-3 sentence summary of the article]

## Key Points
[Bulleted main arguments/findings]

## Context
[Why this matters, related topics]
```

### 5.3 `docs`

```markdown
# [Page Title]
[What this documentation covers]

## Overview
[Brief explanation]

## Usage
[Code examples, commands]

## Parameters / API
[Structured reference]

## See Also
[Related documentation]
```

### 5.4 `landing`

```markdown
# [Page/Company Name]
[What this is in one sentence]

## What It Does
[Core functionality/value proposition]

## Key Features
[Bulleted feature list]

## Pricing
[Pricing tiers if applicable]

## Alternatives
[Competitors/alternatives with context]
```

### 5.5 `listing`

```markdown
# [Category/Search Name]
[What this listing contains]

## Items
[Structured list of items with key attributes]

## Filters Available
[What filtering options exist]
```

### 5.6 `profile`

```markdown
# [Name]
[Role/description]

## About
[Brief bio/description]

## Key Information
[Contact, location, expertise]

## Notable Work
[Achievements, publications, projects]
```

### 5.7 `event`

```markdown
# [Event Name]
[Type of event]

## Details
[Date, time, location, format]

## Description
[What the event is about]

## Registration
[How to attend, pricing]
```

### 5.8 `recipe`

```markdown
# [Recipe Name]
[Cuisine, difficulty, time]

## Ingredients
[Structured ingredient list]

## Steps
[Numbered steps]

## Notes
[Tips, variations, dietary info]
```

### 5.9 `faq`

```markdown
# [Topic] FAQ

## [Question 1]
[Answer 1]

## [Question 2]
[Answer 2]
```

### 5.10 `custom`

Any content that doesn't fit the above types. MUST still follow the general body principles from Section 2.4.

## 6. Content Negotiation

### 6.1 Accept Header

Agents request MAKO content using the `Accept` header:

```
Accept: text/mako+markdown
```

If the server supports MAKO, it MUST respond with `Content-Type: text/mako+markdown`.

If the server does not support MAKO, standard HTTP content negotiation applies (fallback to `text/html`).

### 6.2 HEAD Requests

Servers SHOULD support `HEAD` requests that return MAKO headers without the body. This enables agents to check embeddings for relevance before downloading content.

### 6.3 Discovery

Servers SHOULD advertise MAKO support in HTML pages using:

```html
<link rel="alternate" type="text/mako+markdown" href="/page.mako.md">
```

Servers MAY also provide a discovery endpoint at `/.well-known/mako`:

```json
{
  "mako": "1.0",
  "site": "example.com",
  "content_negotiation": true,
  "html_embedding": true,
  "accept": "text/mako+markdown",
  "spec": "https://makospec.vercel.app"
}
```

The discovery endpoint declares **site-level** MAKO capabilities. Required fields: `mako` (protocol version) and `site` (domain). All other fields are optional.

For **page-level** discovery, servers SHOULD use a standard sitemap or a dedicated MAKO sitemap (e.g., `/mako-sitemap.json`) listing individual pages with their type, token count, and last update date.

### 6.4 HTML Embedding

MAKO content MAY be embedded directly in HTML pages using a `<script>` tag in the document `<head>`. This allows agents that parse HTML to extract MAKO content without content negotiation, enabling immediate adoption without requiring changes from LLM providers or crawlers.

```html
<head>
  <script type="text/mako+markdown">
---
mako: "1.0"
type: product
entity: "Bolso Bandolera Roma"
updated: 2026-02-14
tokens: 210
language: es
summary: "Bolso bandolera artesanal en piel natural, diseño italiano."
media:
  cover:
    url: /uploads/bolso-roma-frontal.webp
    alt: "Bolso Bandolera Roma - Piel Natural"
  images: 5
  video: 1
actions:
  - name: add_to_cart
    description: "Añadir al carrito"
links:
  internal:
    - url: /bolsos/bandolera
      context: "Todos los bolsos bandolera, 23 modelos"
      type: parent
---

# Bolso Bandolera Roma

Bolso bandolera artesanal en piel natural italiana.
Diseño compacto para uso diario con cierre magnético.

## Key Facts

- **Precio:** €89.90
- **Material:** Piel vacuno natural, curtido vegetal
- **Colores:** Cognac, Negro, Camel
- **Stock:** Disponible (envío 24-48h)

## Context

Gama media de Moniisima. Más compacto que Florencia (€119).
Piel natural vs sintética en competidores a precio similar.
  </script>
</head>
```

#### Why HTML Embedding

HTML embedding follows the same adoption pattern as JSON-LD:

1. **Works today** — LLMs and crawlers already parse HTML. No protocol changes needed.
2. **Publisher controlled** — The publisher decides exactly what the agent sees.
3. **Progressive enhancement** — HTML embedding coexists with content negotiation. Agents that support `Accept: text/mako+markdown` get the optimized response; agents that only parse HTML still find the MAKO content in the `<script>` tag.

#### Rules

1. The `<script>` tag MUST use `type="text/mako+markdown"`. Browsers ignore unknown script types.
2. The content MUST be a complete, valid MAKO file (YAML frontmatter + markdown body).
3. The embedded content SHOULD match what the server would return via content negotiation.
4. If both HTML embedding and content negotiation are available, agents SHOULD prefer content negotiation (lower bandwidth).
5. There MUST be at most one `<script type="text/mako+markdown">` tag per page.

## 7. HTTP Headers

See [headers.md](headers.md) for the complete HTTP headers reference.

### 7.1 Response Headers Summary

| Header | Required | Description |
|--------|----------|-------------|
| `X-Mako-Version` | Yes | Spec version |
| `X-Mako-Tokens` | Yes | Token count of body |
| `X-Mako-Type` | Yes | Content type |
| `X-Mako-Lang` | Yes | Content language |
| `X-Mako-Embedding` | No | CEF-encoded embedding vector |
| `X-Mako-Embedding-Model` | No | Embedding model identifier |
| `X-Mako-Embedding-Dim` | No | Embedding dimensions |
| `X-Mako-Actions` | No | Comma-separated action names |

Servers SHOULD also include standard HTTP caching headers (`ETag`, `Cache-Control`, `Last-Modified`, `Vary: Accept`). These replace the need for custom headers for freshness, update timestamps, and canonical URLs — see [headers.md](headers.md) for details.

## 8. Compact Embedding Format (CEF)

See [cef.md](cef.md) for the full CEF specification.

### 8.1 Summary

CEF defines how to compress embedding vectors for transport in HTTP headers:

1. **Quantize** — float32 to int8 (75% size reduction)
2. **Compress** — gzip (variable reduction)
3. **Encode** — base64url (HTTP-safe encoding)

Result: a 512-dimension embedding fits in ~470 bytes.

## 9. Conformance

### 9.1 MAKO Levels

| Level | Requirements |
|-------|-------------|
| **Level 1** | Valid frontmatter + optimized markdown body |
| **Level 2** | Level 1 + HTTP content negotiation + response headers |
| **Level 3** | Level 2 + CEF embedding in headers |

### 9.2 Validation

A MAKO file is valid if:

1. Frontmatter contains all required fields
2. `mako` version is a recognized version string
3. `type` is a recognized content type
4. `tokens` field matches actual token count (within 10% tolerance)
5. Body follows the structure guidelines for its content type
6. If `actions` are declared, each has `name` and `description`
7. If `links` are declared, each has `url` and `context`

## 10. Security Considerations

### 10.1 Transport Security

- MAKO files MUST be served over HTTPS
- MAKO files SHOULD NOT contain credentials, API keys, or secrets
- Action endpoints declared in MAKO SHOULD require their own authentication
- Servers SHOULD respect `robots.txt` directives for MAKO content

### 10.2 Untrusted Embeddings

The `X-Mako-Embedding` header is provided by the content publisher and MUST be treated as **untrusted input**. Embeddings are a hint for pre-filtering, not a guarantee of relevance or quality.

Consumers SHOULD:

1. Generate their own embeddings from the received MAKO content
2. Use the publisher's embedding only for approximate pre-filtering (e.g., HEAD request triage)
3. Never use publisher-provided embeddings as the sole input for final ranking or recommendations

This is analogous to HTML `<meta name="description">`: publishers declare it, but search engines verify and may override it based on actual content.

### 10.3 Content Authenticity

MAKO does not provide built-in mechanisms to verify that the MAKO representation is faithful to the source HTML. **This is intentional.**

MAKO is a content format and delivery protocol. Verification of content authenticity is the responsibility of consumers, just as it is for HTML:

- **Search engines and crawlers** that already index HTML can compare it against MAKO responses from the same URL
- **LLM providers** can fetch both representations and cross-validate
- **Third-party trust services** may emerge as an optional ecosystem layer

MAKO does not attempt to solve content trust because any self-declared integrity mechanism (hashes, signatures, verification headers) can be trivially gamed by a publisher who controls both the HTML and the MAKO response. The protocol provides verifiable content; it does not provide verification.

### 10.4 Spam and Manipulation

Publishers can generate arbitrary MAKO content, including:

- Optimized embeddings that match popular queries but link to low-quality content
- Inflated `media` counts or misleading `type` classifications
- High-volume URL generation with programmatic MAKO responses

Consumers MUST implement their own spam detection, quality scoring, and ranking mechanisms — the same defenses they apply to HTML content today. MAKO does not change the threat model; it changes the delivery format.

### 10.5 Embedding Privacy

- Embedding vectors represent semantic content, not source text — the original text cannot be reconstructed from an embedding
- Embedding vectors MUST NOT encode personally identifiable information
- Servers MUST NOT embed user-specific information in CEF vectors

## 11. Privacy Considerations

- MAKO files are public content representations — they MUST NOT contain private user data
- Embedding vectors represent semantic content, not user behavior
- Servers MAY track MAKO requests for analytics but MUST comply with applicable privacy regulations

## 12. IANA Considerations

This specification defines the media type `text/mako+markdown` which SHOULD be registered with IANA upon standardization.

## 13. References

- [llms.txt Specification](https://llmstxt.org/)
- [WebMCP W3C Specification](https://webmachinelearning.github.io/webmcp/)
- [Cloudflare Markdown for Agents](https://developers.cloudflare.com/fundamentals/reference/markdown-for-agents/)
- [RFC 7231 - HTTP/1.1 Semantics and Content](https://datatracker.ietf.org/doc/html/rfc7231)
- [RFC 7763 - The text/markdown Media Type](https://datatracker.ietf.org/doc/html/rfc7763)
