# MAKO HTTP Headers Reference

**Status:** Draft
**Last Updated:** 2026-02-13

## Request Headers

### Accept

Agents request MAKO content using the standard HTTP `Accept` header:

```http
Accept: text/mako+markdown
```

Agents MAY include fallback types:

```http
Accept: text/mako+markdown, text/markdown;q=0.8, text/html;q=0.5
```

## Response Headers

### Required Headers

These headers MUST be present in every MAKO response:

#### `X-Mako-Version`

The MAKO specification version.

```http
X-Mako-Version: 1.0
```

#### `X-Mako-Tokens`

Estimated token count of the response body (excluding headers and frontmatter).

```http
X-Mako-Tokens: 280
```

#### `X-Mako-Type`

The content type of the page (see spec.md Section 5).

```http
X-Mako-Type: product
```

#### `X-Mako-Lang`

Content language in BCP 47 format.

```http
X-Mako-Lang: en
```

#### `Content-Type`

Standard HTTP content type header:

```http
Content-Type: text/mako+markdown; charset=utf-8
```

### Optional Headers

#### `X-Mako-Embedding`

CEF-encoded embedding vector (see cef.md). Enables pre-download relevance filtering.

```http
X-Mako-Embedding: H4sIAAAAAAAAA2NgGAWjYBSMglEwCkYBNQEAN8zuSAAQAAA
```

#### `X-Mako-Embedding-Model`

Identifier of the embedding model used. REQUIRED when `X-Mako-Embedding` is present.

```http
X-Mako-Embedding-Model: mako-cef-v1
```

#### `X-Mako-Embedding-Dim`

Number of dimensions in the original embedding vector. REQUIRED when `X-Mako-Embedding` is present.

```http
X-Mako-Embedding-Dim: 512
```

#### `X-Mako-Updated`

ISO 8601 timestamp of the last content update.

```http
X-Mako-Updated: 2026-02-13T10:30:00Z
```

#### `X-Mako-Freshness`

How frequently the content changes. Helps agents decide caching strategy.

```http
X-Mako-Freshness: daily
```

Values: `realtime`, `hourly`, `daily`, `weekly`, `monthly`, `static`

#### `X-Mako-Actions`

Comma-separated list of available action names. Allows agents to discover actions from headers alone.

```http
X-Mako-Actions: add_to_cart, check_availability, compare
```

#### `X-Mako-Entity`

The primary entity described by the page.

```http
X-Mako-Entity: Nike Air Max 90
```

#### `X-Mako-Canonical`

URL of the HTML version of this page.

```http
X-Mako-Canonical: https://example.com/product/nike-air-max-90
```

## Complete Response Example

### HEAD Request (relevance check)

```http
HEAD /product/nike-air-max-90 HTTP/2
Accept: text/mako+markdown
```

```http
HTTP/2 200 OK
Content-Type: text/mako+markdown; charset=utf-8
X-Mako-Version: 1.0
X-Mako-Tokens: 280
X-Mako-Type: product
X-Mako-Lang: en
X-Mako-Entity: Nike Air Max 90
X-Mako-Updated: 2026-02-13T10:30:00Z
X-Mako-Freshness: daily
X-Mako-Actions: add_to_cart, check_availability
X-Mako-Embedding: H4sIAAAAAAAAA2NgGAWjYBSMglEwCkYBNQEAN8zuSAAQAAA
X-Mako-Embedding-Model: mako-cef-v1
X-Mako-Embedding-Dim: 512
X-Mako-Canonical: https://example.com/product/nike-air-max-90
Content-Length: 0
```

### GET Request (full content)

```http
GET /product/nike-air-max-90 HTTP/2
Accept: text/mako+markdown
```

```http
HTTP/2 200 OK
Content-Type: text/mako+markdown; charset=utf-8
X-Mako-Version: 1.0
X-Mako-Tokens: 280
X-Mako-Type: product
X-Mako-Lang: en
X-Mako-Embedding: H4sIAAAAAAAAA2NgGAWjYBSMglEwCkYBNQEAN8zuSAAQAAA
X-Mako-Embedding-Model: mako-cef-v1
X-Mako-Embedding-Dim: 512

---
mako: "1.0"
type: product
entity: "Nike Air Max 90"
updated: 2026-02-13
tokens: 280
language: en
...
---

# Nike Air Max 90
...
```

## No MAKO Support

If a server does not support MAKO, it SHOULD respond with standard content negotiation:

```http
HTTP/2 406 Not Acceptable
```

Or fall back to HTML:

```http
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
```

Agents MUST handle both cases gracefully.

## Caching

MAKO responses are cacheable. Servers SHOULD include standard HTTP caching headers:

```http
Cache-Control: public, max-age=3600
ETag: "mako-abc123"
Vary: Accept
```

The `Vary: Accept` header is REQUIRED to ensure CDNs serve the correct version (HTML vs MAKO) based on the request.
