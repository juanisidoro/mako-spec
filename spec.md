# MAKO Specification v0.1.0

**Status:** Draft
**Last Updated:** 2026-02-13

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
| `freshness` | string | No | Content freshness: `"realtime"`, `"daily"`, `"weekly"`, `"static"` |

### 2.4 Markdown Body

The body MUST follow these principles:

1. **Lead with the most important information** — The first paragraph should answer the primary query an agent might have about this page.
2. **Use structured sections** — Headings (`##`) to organize information logically.
3. **Prefer lists and key-value pairs** — Over prose paragraphs when presenting factual data.
4. **Include context and comparisons** — Help the LLM understand relative positioning (competitors, alternatives, trade-offs).
5. **Omit navigation, legal boilerplate, and UI text** — Only include substantive content.
6. **Keep it concise** — Target 200-500 tokens for most pages. Never exceed 1,000 tokens.

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

Servers MAY also provide a MAKO sitemap at `/.well-known/mako.json`:

```json
{
  "mako": "1.0",
  "pages": [
    {
      "url": "/product/shoes",
      "type": "product",
      "tokens": 280,
      "updated": "2026-02-13"
    }
  ]
}
```

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
| `X-Mako-Updated` | No | Last update timestamp |
| `X-Mako-Freshness` | No | Content freshness indicator |
| `X-Mako-Actions` | No | Comma-separated action names |

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

- MAKO files MUST be served over HTTPS
- Embedding vectors MUST NOT encode personally identifiable information
- Servers SHOULD respect `robots.txt` directives for MAKO content
- MAKO files SHOULD NOT contain credentials, API keys, or secrets
- Action endpoints declared in MAKO SHOULD require their own authentication

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
