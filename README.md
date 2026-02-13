<p align="center">
  <strong>MAKO</strong>
</p>

<h1 align="center">MAKO</h1>
<p align="center"><strong>Markdown Agent Knowledge Optimization</strong></p>
<p align="center">The open standard for serving LLM-optimized content on the web.</p>

<p align="center">
  <a href="spec.md">Spec</a> &middot;
  <a href="cef.md">CEF Format</a> &middot;
  <a href="examples/">Examples</a> &middot;
  <a href="CONTRIBUTING.md">Contributing</a>
</p>

---

## The Problem

AI agents consume billions of web pages daily. Today, they receive raw HTML full of navigation bars, ads, scripts, and boilerplate — wasting **80-95% of tokens** on noise. Even Cloudflare's Markdown for Agents only auto-converts format without semantic optimization.

There is no standard way for a website to tell an AI agent: *"Here is exactly what you need to know about this page, in the most efficient format possible."*

## The Solution

**MAKO** defines a per-page, semantically optimized markdown format that includes:

- **Optimized content** — structured for LLM comprehension, not human browsing
- **Compact embeddings** — pre-computed semantic vectors in HTTP headers (470 bytes) for instant relevance filtering
- **Declared actions** — what an agent can do on this page
- **Semantic links** — internal/external links with context
- **Metadata** — token count, content type, language, freshness

## How It Works

```
1. Agent sends:  HEAD /product/shoes  →  Accept: text/mako+markdown

2. Server returns headers only:
   X-Mako-Version: 1.0
   X-Mako-Tokens: 280
   X-Mako-Embedding: H4sIAAAA...  (470 bytes, compressed)

3. Agent checks embedding similarity against query:
   → Low similarity  → skip (zero tokens wasted)
   → High similarity → GET request for full MAKO content (280 tokens)

4. Result: 280 tokens instead of 4,500+ from HTML (94% reduction)
```

## Quick Example

```markdown
---
mako: 1.0
type: product
entity: "Nike Air Max 90"
updated: 2026-02-13
tokens: 280
language: en

actions:
  - name: add_to_cart
    description: "Add product to shopping cart"
    endpoint: /api/cart

links:
  internal:
    - url: /category/running
      context: "Browse running shoes"
  external:
    - url: https://nike.com/air-max-90
      context: "Official manufacturer page"
---

# Nike Air Max 90

Mid-range casual running shoe.

## Key Facts
- Price: 79.99 EUR (was 149.99, -47%)
- In stock, sizes 38-46
- Rating: 4.3/5 (234 reviews)

## Context
Direct competitors: Adidas Ultraboost (129€), New Balance 1080 (139€).
Strength: price. Weakness: narrow fit per reviews.
```

## Why MAKO?

| Current approach | With MAKO |
|-----------------|-----------|
| Download full HTML (4,500+ tokens) | Download MAKO file (280 tokens) — **94% reduction** |
| Parse and strip HTML noise | Clean, structured content ready to use |
| No way to pre-filter relevance | Embedding in header filters in **470 bytes** |
| One llms.txt per site | One MAKO file **per page** |
| Auto-converted markdown | **Semantically optimized** content |
| No action discovery | Actions declared in frontmatter |

## Built to Complement, Not Compete

MAKO is designed to work alongside existing standards, not replace them:

- **[Cloudflare Markdown for Agents](https://developers.cloudflare.com/fundamentals/reference/markdown-for-agents/)** handles automatic HTML-to-Markdown conversion at the CDN edge. MAKO adds a **semantic optimization layer** on top, with structured metadata, actions, and embeddings.
- **[WebMCP](https://webmachinelearning.github.io/webmcp/)** enables action execution on web pages (the "verbs"). MAKO provides the content and context (the "knowledge"). Together they form a complete AI-ready web stack.
- **[llms.txt](https://llmstxt.org/)** provides a site-level overview. MAKO extends this idea to **per-page** granularity with richer metadata.
- **[Schema.org](https://schema.org/)** structures data for search engines. MAKO structures content for LLM agents. Both can coexist on the same page.

## Specification

| Document | Description |
|----------|-------------|
| [spec.md](spec.md) | Full MAKO specification |
| [cef.md](cef.md) | Compact Embedding Format (CEF) specification |
| [examples/](examples/) | Example MAKO files for different content types |
| [headers.md](headers.md) | HTTP headers reference |

## Ecosystem

### Open Source
- **mako-spec** — This specification (you are here)
- **mako-js** — JavaScript library for parsing/generating MAKO files + CEF *(coming soon)*
- **mako-cli** — Command-line validator and generator *(coming soon)*
- **mako-wp** — WordPress plugin *(coming soon)*

### Content Types Supported
- `product` — E-commerce product pages
- `article` — Blog posts, news articles
- `docs` — Technical documentation
- `landing` — Landing pages, homepages
- `profile` — People, company profiles
- `listing` — Category pages, search results
- `event` — Events, meetups, conferences
- `recipe` — Recipes, how-to guides
- `faq` — FAQ pages
- `custom` — Any other content type

## The Numbers

| Metric | HTML | Cloudflare MD | MAKO |
|--------|------|--------------|------|
| Avg tokens per page | 4,500 | 900 | 280 |
| Semantic optimization | None | None | Full |
| Pre-download filtering | Impossible | Impossible | 470 bytes |
| Action discovery | No | No | Yes |
| Link context | No | No | Yes |

## Getting Started

### For website owners

Add MAKO support to your site:

```bash
# Install the CLI
npm install -g @mako-spec/cli

# Generate MAKO files for your site
mako generate https://your-site.com

# Validate your MAKO files
mako validate ./mako-files/
```

### For AI agent developers

Request MAKO content:

```bash
# Check if a page supports MAKO
curl -I https://example.com/page -H "Accept: text/mako+markdown"

# Download MAKO content
curl https://example.com/page -H "Accept: text/mako+markdown"
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines. We welcome contributions from everyone.

## License

This specification is released under [Apache 2.0](LICENSE).
