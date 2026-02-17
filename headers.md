# MAKO HTTP Headers Reference

**Status:** Draft
**Last Updated:** 2026-02-17

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

#### `X-Mako-Actions`

Comma-separated list of available action names. Allows agents to discover actions from headers alone.

```http
X-Mako-Actions: add_to_cart, check_availability, compare
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
X-Mako-Actions: add_to_cart, check_availability
X-Mako-Embedding: H4sIAAAAAAAAA2NgGAWjYBSMglEwCkYBNQEAN8zuSAAQAAA
X-Mako-Embedding-Model: mako-cef-v1
X-Mako-Embedding-Dim: 512
Vary: Accept
ETag: "mako-a1b2c3"
Cache-Control: public, max-age=86400
Last-Modified: 2026-02-13T10:30:00Z
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
X-Mako-Actions: add_to_cart, check_availability
Vary: Accept
ETag: "mako-a1b2c3"
Cache-Control: public, max-age=86400

---
mako: "1.0"
type: product
entity: "Nike Air Max 90"
updated: 2026-02-13
tokens: 280
language: en
canonical: "https://example.com/product/nike-air-max-90"
freshness: daily
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

## Caching and Standard HTTP Headers

MAKO relies on standard HTTP caching mechanisms rather than custom headers for freshness, timestamps, and canonical URLs. This avoids reinventing HTTP semantics.

### Required

| Header | Description |
|--------|-------------|
| `Vary: Accept` | **REQUIRED.** Ensures CDNs serve HTML vs MAKO correctly based on request. |

### Recommended

| Header | Description |
|--------|-------------|
| `ETag` | Content fingerprint for conditional requests (`If-None-Match`). |
| `Cache-Control` | Caching strategy (e.g., `public, max-age=3600`). |
| `Last-Modified` | Last content update timestamp (replaces the need for a custom update header). |
| `Content-Location` | Canonical URL of the HTML version, if different from the request URL. |

### Mapping `freshness` to `Cache-Control`

The `freshness` field in YAML frontmatter declares the content's update cadence. Servers SHOULD set `Cache-Control: max-age` accordingly:

| `freshness` | Suggested `max-age` |
|-------------|---------------------|
| `realtime` | `0` (no-cache) |
| `hourly` | `3600` |
| `daily` | `86400` |
| `weekly` | `604800` |
| `monthly` | `2592000` |
| `static` | `31536000` |

### Example

```http
Vary: Accept
ETag: "mako-a1b2c3"
Cache-Control: public, max-age=86400
Last-Modified: 2026-02-13T10:30:00Z
```
