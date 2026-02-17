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

MAKO classifies response headers into three levels using [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) keywords. The rationale: Required headers are the minimum for a valid MAKO response, Recommended headers reuse standard HTTP semantics for caching and interoperability, and Optional headers add value for advanced agent capabilities.

### Required (MUST)

Without these, the response is **not a valid MAKO response**. They enable an agent to identify the format, evaluate relevance, and make pre-download decisions from a HEAD request alone.

#### `Content-Type`

Identifies the response as MAKO content.

```http
Content-Type: text/mako+markdown; charset=utf-8
```

#### `X-Mako-Version`

The MAKO protocol version. Allows agents to check compatibility before parsing.

```http
X-Mako-Version: 1.0
```

#### `X-Mako-Tokens`

Estimated token count of the response body (excluding headers and frontmatter). Agents use this for budget decisions — "can I afford to read this page?"

```http
X-Mako-Tokens: 280
```

#### `X-Mako-Type`

Content type of the page (see spec.md Section 5). Agents use this to filter by category — "I only want products".

```http
X-Mako-Type: product
```

#### `X-Mako-Lang`

Content language in BCP 47 format. Agents use this to filter by locale.

```http
X-Mako-Lang: en
```

#### `Vary`

**REQUIRED** to ensure CDNs cache HTML and MAKO responses separately for the same URL.

```http
Vary: Accept
```

### Recommended (SHOULD)

Standard HTTP headers that improve caching, conditional requests, and interoperability. MAKO does not define custom headers for these concerns — it reuses existing HTTP semantics.

#### `ETag`

Content fingerprint. Enables conditional requests via `If-None-Match`, allowing agents to receive a `304 Not Modified` response (0 bytes) when content hasn't changed.

```http
ETag: "mako-a1b2c3"
```

#### `Cache-Control`

Caching strategy. Servers SHOULD set `max-age` based on the `freshness` field in YAML frontmatter (see Mapping section below).

```http
Cache-Control: public, max-age=86400
```

#### `Last-Modified`

Last content update timestamp in HTTP-date format (RFC 7232). Replaces the need for a custom update header.

```http
Last-Modified: Mon, 30 Jan 2023 00:00:00 GMT
```

#### `Content-Location`

Canonical URL of the HTML version of this page, if different from the request URL. Standard HTTP header (RFC 7231 §3.1.4.2).

```http
Content-Location: https://example.com/product/nike-air-max-90
```

### Optional (MAY)

Extra metadata for advanced agent capabilities. Not required for a valid response, but adds value for discovery and semantic pre-filtering.

#### `X-Mako-Actions`

Comma-separated list of available action names. Allows agents to discover capabilities from a HEAD request without downloading the body.

```http
X-Mako-Actions: add_to_cart, check_availability, compare
```

#### `X-Mako-Embedding`

CEF-encoded embedding vector (see cef.md). Enables semantic relevance scoring before downloading: the agent decodes the embedding, computes cosine similarity against its query, and decides whether to GET.

```http
X-Mako-Embedding: H4sIAAAAAAAAA2NgGAWjYBSMglEwCkYBNQEAN8zuSAAQAAA
```

#### `X-Mako-Embedding-Model`

Identifier of the embedding model used. **REQUIRED** when `X-Mako-Embedding` is present.

```http
X-Mako-Embedding-Model: mako-cef-v1
```

#### `X-Mako-Embedding-Dim`

Number of dimensions in the original embedding vector. **REQUIRED** when `X-Mako-Embedding` is present.

```http
X-Mako-Embedding-Dim: 512
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

## Mapping `freshness` to `Cache-Control`

The `freshness` field in YAML frontmatter declares the content's update cadence. Servers SHOULD set `Cache-Control: max-age` accordingly:

| `freshness` | Suggested `max-age` |
|-------------|---------------------|
| `realtime` | `0` (no-cache) |
| `hourly` | `3600` |
| `daily` | `86400` |
| `weekly` | `604800` |
| `monthly` | `2592000` |
| `static` | `31536000` |

