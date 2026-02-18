# MAKO Specification v0.1.0

**Status:** Draft
**Last Updated:** 2026-02-18

> **Versioning note:** `v0.1.0` is the version of this *specification document*. The protocol version used in MAKO frontmatter is `"1.0"` (the `mako` field). These are distinct: the document version tracks spec revisions, while the protocol version identifies the format that agents negotiate. When the protocol itself introduces breaking changes, the frontmatter version will increment.

## 1. Introduction

MAKO (Markdown Agent Knowledge Optimization) is an open standard that defines how web pages serve semantically optimized content to AI agents. Unlike auto-conversion approaches (HTML-to-Markdown), MAKO provides a per-page, purpose-built representation designed for maximum LLM comprehension with minimum token usage.

### 1.1 Goals

- Define a per-page content format optimized for LLM consumption
- Declare available actions, semantic links, and metadata alongside content
- Minimize token usage while maximizing information density
- Enable pre-download relevance filtering via HTTP headers (type, language, tokens)
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

MAKO assumes the agent is an **HTTP client** capable of sending custom `Accept` headers and parsing HTTP response headers. This is the primary access model. Agents that can additionally parse HTML gain access to supplementary discovery mechanisms (see §6.4). MAKO does not require JavaScript execution at any point.

When an agent encounters a URL with no MAKO support, it MUST fail silently and process the response as standard HTML. Specifically, an agent concludes that a URL does not support MAKO only when **all four** of the following conditions are true:

1. HEAD response does not include `Content-Type: text/mako+markdown`
2. HEAD response does not include a `Link` header with `rel="alternate" type="text/mako+markdown"`
3. GET response does not include `Content-Type: text/mako+markdown`
4. The HTML body (if received) does not contain `<script type="text/mako+markdown">` or `<link rel="alternate" type="text/mako+markdown">`

#### Recommended Agent Flow

The following sequence describes the recommended interaction pattern. Each step is independent — agents MAY skip to any step based on prior knowledge.

```
1. GET /.well-known/mako
   → 200: site declares MAKO support (positive signal)
   → 404: MAKO support unknown (NOT a negative signal — the site
          may support content negotiation without a discovery endpoint)

2. HEAD <url> with Accept: text/mako+markdown
   → Evaluate X-Mako-* headers for relevance (type, language, tokens)
   → No body downloaded — pre-filter decision at near-zero cost

3. GET <url> with Accept: text/mako+markdown
   → Content-Type: text/mako+markdown → MAKO document received, done
   → Content-Type: text/html → no content negotiation support, go to step 4

4. Parse the HTML already received in step 3 (no additional request)
   → Look for <script type="text/mako+markdown"> → extract inline MAKO
   → Look for <link rel="alternate" type="text/mako+markdown"> → follow link
   → Neither found → URL does not support MAKO
```

A minimal agent uses only step 3. A sophisticated agent uses all four steps for maximum efficiency. MAKO scales with agent capability.

### 6.1 Accept Header

Agents request MAKO content using the `Accept` header:

```
Accept: text/mako+markdown
```

If the server supports MAKO, it MUST respond with `Content-Type: text/mako+markdown`.

If the server does not support MAKO, standard HTTP content negotiation applies (fallback to `text/html`).

### 6.2 HEAD Requests

Servers SHOULD support `HEAD` requests that return MAKO headers without the body. This enables agents to evaluate metadata (type, language, token count) for relevance before downloading content.

Servers SHOULD include an `ETag` header in all MAKO responses (both HEAD and GET). Without a server-provided ETag, the conditional request mechanisms described below have no basis to operate.

#### Coherence Between HEAD and GET

The HEAD and GET responses are **not atomic**. Content may change between the two requests (e.g., price updates, stock changes). Agents MUST NOT assume that metadata evaluated during a HEAD request is coherent with a subsequent GET response.

To verify coherence, agents SHOULD include `If-None-Match` with the ETag from the HEAD response when making the subsequent GET:

```http
HEAD /product/123
Accept: text/mako+markdown
→ ETag: "mako-a1b2c3"
→ X-Mako-Tokens: 280

GET /product/123
Accept: text/mako+markdown
If-None-Match: "mako-a1b2c3"
→ 304 Not Modified (content matches HEAD snapshot)
→ 200 OK (content changed since HEAD — re-evaluate)
```

A `304` confirms the GET content matches the HEAD metadata. A `200` indicates the content has changed and the agent should re-evaluate rather than rely on previously inspected headers.

### 6.3 Discovery

MAKO defines three complementary discovery mechanisms:

#### Site-level: `/.well-known/mako`

Servers MAY provide a discovery endpoint at `/.well-known/mako` declaring MAKO support:

```json
{
  "mako": "1.0"
}
```

The only required field is `mako` — the protocol version. This minimal response is sufficient for agents to confirm MAKO support. Implementations MAY include additional fields, but the protocol intentionally keeps discovery lightweight.

#### Page-level: HTML `<link>` element

Servers SHOULD advertise MAKO support in HTML pages using a standard alternate link:

```html
<link rel="alternate" type="text/mako+markdown">
```

When the `href` attribute is **omitted**, the link signals that the **same URL** serves MAKO content when requested with `Accept: text/mako+markdown` — standard content negotiation.

When the `href` attribute is **present**, it specifies an **explicit endpoint** that serves the MAKO content for this page:

```html
<link rel="alternate" type="text/mako+markdown" href="https://example.com/api/mako?path=/product/123">
```

This is a valid and practical implementation strategy: the publisher exposes a dedicated MAKO endpoint (e.g., `/api/mako`) that serves MAKO content for any page via a path parameter. Each HTML page includes a `<link>` pointing to its corresponding MAKO URL. The endpoint URL does not need to match the page URL — the `<link>` element is simply the discovery mechanism.

Both patterns are equally valid:

| Pattern | `<link>` | Agent retrieval |
|---------|----------|-----------------|
| Content negotiation | `<link rel="alternate" type="text/mako+markdown">` | GET same URL with `Accept: text/mako+markdown` |
| Explicit endpoint | `<link rel="alternate" type="text/mako+markdown" href="...">` | GET the `href` URL directly |

This flexibility allows servers that cannot modify response headers or implement content negotiation (e.g., static hosting, shared hosting, CDNs) to still serve MAKO content via a separate endpoint.

#### Page-level: HTML `<script>` element

When `html_embedding` is enabled (see Section 6.4), agents can discover embedded MAKO content directly in the HTML `<head>`:

```html
<script type="text/mako+markdown" id="mako">
```

This allows agents to extract MAKO content without content negotiation, enabling immediate adoption without requiring changes from LLM providers or crawlers.

### 6.4 HTML Embedding

MAKO content MAY be embedded directly in HTML pages using a `<script>` tag in the document `<head>`. This is a **fallback mechanism** for sites that **cannot implement server-side content negotiation** — for example, static sites, client-only SPAs without a backend, or environments where modifying HTTP response headers is not possible.

Sites that already support content negotiation (via middleware, edge functions, or server-side routing) do **not** need HTML embedding. Content negotiation is the preferred delivery mechanism: it is more efficient (agents download only the MAKO content, not the full HTML page) and provides fresher data (see §6.5 Freshness).

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

HTML embedding is designed for adoption scenarios where content negotiation is not available. It follows the same pattern as JSON-LD:

1. **Works today** — LLMs and crawlers already parse HTML. No server-side changes required.
2. **Publisher controlled** — The publisher decides exactly what the agent sees.
3. **Progressive enhancement** — HTML embedding coexists with content negotiation. Agents that support `Accept: text/mako+markdown` get the optimized response; agents that only parse HTML still find the MAKO content in the `<script>` tag.

**When to use HTML embedding vs content negotiation:**

| Scenario | Recommended mechanism |
|----------|----------------------|
| Server-side framework (Next.js, Express, WordPress, etc.) | Content negotiation via middleware |
| Edge function available (Cloudflare Workers, Vercel Edge) | Content negotiation at the edge |
| Static site with API backend | `<link>` pointing to API endpoint |
| Fully static site, no backend | HTML embedding via `<script>` |
| Client-only SPA, no server | HTML embedding for static routes; API endpoint for dynamic routes |

#### Rules

1. The `<script>` tag MUST use `type="text/mako+markdown"`. Browsers ignore unknown script types.
2. The content MUST be a complete, valid MAKO file (YAML frontmatter + markdown body).
3. The embedded content SHOULD match what the server would return via content negotiation.
4. If both HTML embedding and content negotiation are available, agents SHOULD prefer content negotiation (lower bandwidth).
5. There MUST be at most one `<script type="text/mako+markdown">` tag per page.

#### Delivery Considerations

The MIME type `text/mako+markdown` is not yet registered with IANA (see §12). Some CDN providers, security proxies, or WAF configurations may strip `<script>` tags with unrecognized `type` attributes. Implementors SHOULD verify that their delivery pipeline preserves `<script type="text/mako+markdown">` elements intact. Content negotiation (§6.1) is not affected by this limitation, as it operates at the HTTP level rather than within the HTML document.

### 6.5 Single Page Applications

MAKO is rendering-agnostic. The content negotiation mechanism operates at the server level, independent of how the client renders the page.

Single Page Applications (SPAs) serve a minimal HTML shell and render content via JavaScript in the browser. AI agents do not execute JavaScript. Without MAKO, an agent visiting a SPA receives an empty `<div>` — no content, no metadata, no structure.

The server that powers a SPA already has the data: it serves it as JSON to the frontend via API routes. MAKO adds one more response format to the same server:

```
Browser request:   Accept: text/html          → HTML shell + JS bundle
SPA frontend:      Accept: application/json   → JSON data via API
MAKO agent:        Accept: text/mako+markdown → Structured MAKO document
```

Same URL, same server, same data. Only the `Accept` header changes.

#### Implementation by Architecture

| Architecture | MAKO Mechanism | Dynamic Routes |
|-------------|----------------|:-:|
| SSR frameworks (Next.js, Nuxt, Remix, SvelteKit) | Server middleware or route handler | Yes |
| Edge-rendered (Cloudflare Workers, Vercel Edge) | Edge function intercepts request | Yes |
| Static site generation (Astro, Gatsby, Next.js SSG) | Build-time `<script>` embedding per page | Yes (at build time) |
| Client-only SPA (plain React, Vue, Angular) | API backend or edge function | Yes |

For **client-only SPAs with dynamic routes**, the HTML embedding mechanism (§6.4) alone is insufficient: the static `index.html` is identical for all routes, so route-specific MAKO content cannot be embedded without server-side logic. Content negotiation at the server or edge layer is the canonical solution.

For **static or pre-rendered pages** (home, about, landing), HTML embedding via `<script>` works without any server-side changes.

#### Freshness

Content negotiation provides fresher data than embedded `<script>` elements. An embedded `<script>` reflects the state at page generation time and remains stale until the HTML is regenerated. Content negotiation responds in real time from the API or database. For applications serving frequently changing data (prices, stock, availability), servers SHOULD use content negotiation with short-lived `Cache-Control` values and `ETag` headers (§6.2).

## 7. HTTP Headers

See [headers.md](headers.md) for the complete HTTP headers reference.

MAKO uses [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) keywords to classify response headers into three levels. This classification ensures that a minimal MAKO response is lightweight, while allowing servers to progressively enhance with standard HTTP caching and advanced features.

### 7.1 Required (MUST)

Without these headers, the response is not a valid MAKO response. They provide the minimum metadata an agent needs to identify the format, evaluate relevance, and make pre-download decisions.

| Header | Description | Example |
|--------|-------------|---------|
| `Content-Type` | Identifies the MAKO format | `text/mako+markdown; charset=utf-8` |
| `X-Mako-Version` | Protocol version for negotiation | `1.0` |
| `X-Mako-Tokens` | Token budget — agent decides if it can afford to read | `280` |
| `X-Mako-Type` | Content type — agent filters by category | `product` |
| `X-Mako-Lang` | Language — agent filters by locale | `es` |
| `Vary` | CDN correctness — ensures HTML and MAKO are cached separately | `Accept` |

### 7.2 Recommended (SHOULD)

Standard HTTP headers that improve caching, conditional requests, and interoperability. MAKO does not reinvent these — it reuses existing HTTP semantics.

| Header | Description | Example |
|--------|-------------|---------|
| `ETag` | Content fingerprint for conditional requests (`If-None-Match` → 304) | `"mako-a1b2c3"` |
| `Cache-Control` | Caching strategy (see `freshness` mapping in [headers.md](headers.md)) | `public, max-age=86400` |
| `Last-Modified` | Last content update timestamp | `Mon, 30 Jan 2023 00:00:00 GMT` |
| `Content-Location` | Canonical URL of the HTML version | `https://example.com/page` |

### 7.3 Optional (MAY)

Extra metadata for advanced agent capabilities. Not required for a valid response, but adds value for discovery and semantic matching.

| Header | Description | Example |
|--------|-------------|---------|
| `X-Mako-Actions` | Comma-separated action names — capability discovery in HEAD | `add_to_cart, share` |
| `X-Mako-Embedding` | **EXPERIMENTAL.** CEF-encoded embedding vector — semantic pre-filtering without body (see §8) | `H4sIAAAA...` |
| `X-Mako-Embedding-Model` | **EXPERIMENTAL.** Embedding model identifier (REQUIRED if `X-Mako-Embedding` is present) | `nomic-embed-text` |
| `X-Mako-Embedding-Dim` | **EXPERIMENTAL.** Embedding dimensions (REQUIRED if `X-Mako-Embedding` is present) | `512` |

## 8. Compact Embedding Format (CEF) — EXPERIMENTAL

> **Status: EXPERIMENTAL.** This feature is defined for early adopters and is subject to change in future versions. Interoperability across different embedding models is not guaranteed. Implementations SHOULD NOT depend on cross-provider embedding compatibility. Level 1 and Level 2 conformance (§9.1) do not require CEF.

See [cef.md](cef.md) for the full CEF specification.

### 8.1 Summary

CEF defines how to compress embedding vectors for transport in HTTP headers:

1. **Quantize** — float32 to int8 via symmetric per-tensor quantization
2. **Compress** — gzip (variable reduction)
3. **Encode** — base64url (HTTP-safe encoding)

Result: a 512-dimension embedding fits in ~470 bytes.

### 8.2 Quantization Scheme

CEF uses **symmetric per-tensor int8 quantization**. The dequantization formula is:

```
scale = max(abs(min(v)), abs(max(v))) / 127
quantized[i] = round(v[i] / scale)
dequantized[i] = quantized[i] * scale
```

Where `v` is the original float32 vector. The `scale` factor MUST be stored as the first 4 bytes (float32, little-endian) of the compressed payload, followed by the int8 values.

Implementations MUST use this exact scheme. Two conforming implementations using the same embedding model MUST produce compatible quantized vectors.

### 8.3 Model Declaration

Servers that include `X-Mako-Embedding` MUST declare the model via `X-Mako-Embedding-Model` and the dimensions via `X-Mako-Embedding-Dim`. Agents SHOULD skip relevance filtering if the declared model is unknown to them.

The RECOMMENDED reference model is **`nomic-embed-text`** (512 dimensions, open weights, Apache-2.0 license, available via multiple inference providers). This recommendation is not mandatory — implementations MAY use any embedding model provided it is declared in the headers.

Cosine similarity between embeddings from different models is mathematically meaningless. The model declaration exists so that agents can determine compatibility before computing similarity, avoiding silent incorrect results.

### 8.4 Interoperability

Embedding interoperability requires that server and agent use the same embedding model. For the open web, where server and agent implementations are independent, this is not guaranteed. The practical implications are:

- **Same-provider deployments** (e.g., your agent querying your own MAKO-enabled site): full interoperability with any model.
- **Open web**: agents ignore embeddings from unknown models and fall back to downloading the MAKO body for evaluation. No functionality is lost — only the pre-filtering optimization is unavailable.

## 9. Conformance

### 9.1 MAKO Levels

| Level | Requirements | Status |
|-------|-------------|--------|
| **Level 1** | Valid frontmatter + optimized markdown body | Stable |
| **Level 2** | Level 1 + HTTP content negotiation + response headers | Stable |
| **Level 3** | Level 2 + CEF embedding in headers (§8) | **Experimental** |

Level 1 and Level 2 are stable and self-contained. A conforming implementation needs only Level 1 or Level 2 — there is no requirement to implement Level 3. Level 3 is subject to change in future versions as the embedding ecosystem matures.

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

> This section applies to implementations that use the experimental CEF feature (§8).

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

> This section applies to implementations that use the experimental CEF feature (§8).

- Embedding vectors represent semantic content, not source text — the original text cannot be reconstructed from an embedding
- Embedding vectors MUST NOT encode personally identifiable information
- Servers MUST NOT embed user-specific information in CEF vectors

### 10.6 Authentication and Metadata Exposure

Servers MUST NOT include `X-Mako-*` headers in `401 Unauthorized` or `403 Forbidden` responses. Exposing content metadata (type, token count, language, actions) on authentication failures enables **resource enumeration attacks**: an unauthenticated actor can map private resources, their types, sizes, and languages by issuing HEAD requests across URL ranges.

For authenticated MAKO content, the standard HTTP flow applies:

1. The agent includes authentication credentials (e.g., `Authorization: Bearer <token>`) in the request
2. If authenticated, the server responds with the full MAKO response including `X-Mako-*` headers
3. If not authenticated, the server responds with `401` or `403` with no MAKO-specific headers

Discovery of authenticated MAKO endpoints SHOULD use `/.well-known/mako` with appropriate scope declarations rather than per-URL probing.

## 11. Privacy Considerations

- MAKO files are public content representations — they MUST NOT contain private user data
- Embedding vectors represent semantic content, not user behavior
- Servers MAY track MAKO requests for analytics but MUST comply with applicable privacy regulations

## 12. IANA Considerations

This specification defines the media type `text/mako+markdown` which SHOULD be registered with IANA upon standardization.

## 13. Future Directions

The following capabilities are defined as experimental in this version and are candidates for stabilization in future revisions:

- **Semantic pre-filtering via embeddings (§8):** CEF enables agents to evaluate content relevance from HTTP headers alone, without downloading the body. The current challenge is embedding model interoperability across independent implementations. A future version may define a canonical model or a negotiation mechanism (e.g., `Accept-Mako-Embedding-Models` header) once the embedding ecosystem matures.
- **Authenticated content discovery:** Patterns for discovering MAKO-enabled content behind authentication, beyond the current standard HTTP auth flow (§10.6).
- **Site-level manifests:** Extended `/.well-known/mako` schemas that declare available content types, supported languages, and embedding models at the site level.

## 14. References

- [llms.txt Specification](https://llmstxt.org/)
- [WebMCP W3C Specification](https://webmachinelearning.github.io/webmcp/)
- [Cloudflare Markdown for Agents](https://developers.cloudflare.com/fundamentals/reference/markdown-for-agents/)
- [RFC 7231 - HTTP/1.1 Semantics and Content](https://datatracker.ietf.org/doc/html/rfc7231)
- [RFC 7763 - The text/markdown Media Type](https://datatracker.ietf.org/doc/html/rfc7763)
