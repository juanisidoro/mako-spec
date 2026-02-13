# CEF — Compact Embedding Format v0.1.0

**Status:** Draft
**Last Updated:** 2026-02-13

## 1. Introduction

CEF (Compact Embedding Format) defines a standard method for compressing semantic embedding vectors so they can be transported in HTTP headers. This enables AI agents to evaluate content relevance **before downloading** the full page content.

## 2. Problem Statement

A typical embedding vector (512 dimensions, float32) occupies 2,048 bytes — too large for practical HTTP header transport. CEF reduces this to approximately **470 bytes** while preserving >98% of semantic similarity accuracy.

## 3. Encoding Pipeline

```
┌─────────────────┐     ┌──────────────┐     ┌────────────┐     ┌──────────────┐
│  float32 vector │ ──► │  Quantize    │ ──► │  Compress   │ ──► │   Encode     │
│  (2,048 bytes)  │     │  to int8     │     │  gzip       │     │   base64url  │
│                 │     │  (512 bytes) │     │  (~350 B)   │     │   (~470 B)   │
└─────────────────┘     └──────────────┘     └────────────┘     └──────────────┘
```

### 3.1 Step 1: Quantization (float32 → int8)

Convert each float32 value to an 8-bit signed integer using min-max scaling:

```
For each dimension i:
  int8_value[i] = round((float32_value[i] - min) / (max - min) * 255) - 128

Where:
  min = minimum value across all dimensions
  max = maximum value across all dimensions
```

The `min` and `max` values MUST be included in the compressed payload as a 8-byte prefix (two float32 values) to enable decoding.

**Size reduction:** 75% (2,048 → 512 bytes + 8 bytes metadata = 520 bytes)

**Precision loss:** Cosine similarity correlation >0.98 with original float32 vectors.

### 3.2 Step 2: Compression (gzip)

Apply gzip compression (RFC 1952) to the quantized byte array (including the min/max prefix):

```
Input:  520 bytes (8 bytes min/max + 512 bytes int8 vector)
Output: ~350 bytes (variable, depends on vector content)
```

Compression level SHOULD be 6 (default gzip level) for optimal balance of speed and size.

### 3.3 Step 3: Encoding (base64url)

Encode the compressed bytes using base64url (RFC 4648 Section 5) — URL-safe base64 without padding:

```
Input:  ~350 bytes (gzip output)
Output: ~470 characters (base64url string)
```

base64url is used instead of standard base64 because:
- No `+` or `/` characters (HTTP-header safe)
- No `=` padding required
- Safe for use in HTTP headers without escaping

## 4. Header Format

The CEF-encoded embedding is transported in the `X-Mako-Embedding` HTTP header:

```http
X-Mako-Embedding: H4sIAAAAAAAAA2NgGAWjYBSMglEwCkYBNQEAN8zuSAAQAAA
X-Mako-Embedding-Model: mako-cef-v1
X-Mako-Embedding-Dim: 512
```

### 4.1 Required Companion Headers

When `X-Mako-Embedding` is present, the following headers MUST also be present:

| Header | Type | Description |
|--------|------|-------------|
| `X-Mako-Embedding-Model` | string | Identifier of the embedding model used |
| `X-Mako-Embedding-Dim` | integer | Number of dimensions in the original vector |

### 4.2 Size Constraints

| Dimensions | Quantized | Compressed | Encoded | Header safe? |
|-----------|-----------|------------|---------|-------------|
| 256 | 264 B | ~180 B | ~240 chars | Yes |
| 384 | 392 B | ~270 B | ~360 chars | Yes |
| 512 | 520 B | ~350 B | ~470 chars | Yes |
| 768 | 776 B | ~520 B | ~695 chars | Yes |
| 1024 | 1,032 B | ~690 B | ~920 chars | Yes |
| 1536 | 1,544 B | ~1,030 B | ~1,375 chars | Marginal |

**Recommendation:** Use 512 dimensions for optimal balance of semantic quality and header size.

## 5. Decoding Pipeline

```
┌──────────────┐     ┌──────────────┐     ┌────────────┐     ┌─────────────────┐
│  base64url   │ ──► │  Decode      │ ──► │  Decompress│ ──► │   Dequantize    │
│  string      │     │  to bytes    │     │  gunzip    │     │   int8→float32  │
│  (~470 chars)│     │  (~350 B)    │     │  (520 B)   │     │   (2,048 B)     │
└──────────────┘     └──────────────┘     └────────────┘     └─────────────────┘
```

### 5.1 Dequantization

```
Read min, max from first 8 bytes (two float32 values)
For each dimension i:
  float32_value[i] = (int8_value[i] + 128) / 255 * (max - min) + min
```

## 6. Embedding Model Requirements

### 6.1 Standard Models

MAKO does not mandate a specific embedding model. However, for interoperability, the following model identifiers are recognized:

| Model ID | Base Model | Dimensions | Description |
|----------|-----------|------------|-------------|
| `mako-cef-v1` | TBD (open source) | 512 | Default MAKO embedding model |

### 6.2 Custom Models

Servers MAY use custom embedding models. The model identifier MUST be included in the `X-Mako-Embedding-Model` header. Agents that do not recognize the model identifier MAY:

1. Skip the embedding and download the full content
2. Attempt cosine similarity anyway (embeddings from different models may still correlate)
3. Use a model mapping service to translate between embedding spaces

### 6.3 Model Selection Criteria

An embedding model suitable for CEF SHOULD:

- Support 512 dimensions or fewer
- Be publicly available (open source or open API)
- Produce normalized vectors (unit length)
- Perform well on semantic similarity benchmarks (STS, MTEB)

## 7. Similarity Computation

Agents compute relevance using cosine similarity between their query embedding and the page's CEF embedding:

```
similarity = dot(query_vector, page_vector) / (norm(query_vector) * norm(page_vector))
```

### 7.1 Recommended Thresholds

| Similarity | Interpretation | Agent Action |
|-----------|---------------|-------------|
| > 0.85 | Highly relevant | Download MAKO content immediately |
| 0.70 - 0.85 | Potentially relevant | Download if within token budget |
| 0.50 - 0.70 | Marginally relevant | Skip unless no better results |
| < 0.50 | Not relevant | Skip entirely |

These thresholds are advisory. Agents MAY adjust based on their specific requirements.

## 8. Security Considerations

- CEF embeddings represent semantic content, not source text — the original text cannot be reconstructed from an embedding
- Embedding vectors SHOULD be generated from the MAKO content only, not from private data
- Servers MUST NOT embed user-specific information in CEF vectors

## 9. Pseudocode Reference

### Encoding

```python
import gzip
import base64
import struct
import numpy as np

def cef_encode(vector: np.ndarray) -> str:
    """Encode a float32 embedding vector to CEF format."""
    # Step 1: Quantize
    v_min = float(vector.min())
    v_max = float(vector.max())
    normalized = (vector - v_min) / (v_max - v_min)
    quantized = np.round(normalized * 255).astype(np.int8) - 128

    # Prepend min/max as metadata
    metadata = struct.pack('ff', v_min, v_max)
    payload = metadata + quantized.tobytes()

    # Step 2: Compress
    compressed = gzip.compress(payload, compresslevel=6)

    # Step 3: Encode
    encoded = base64.urlsafe_b64encode(compressed).rstrip(b'=').decode('ascii')

    return encoded

def cef_decode(encoded: str, dim: int) -> np.ndarray:
    """Decode a CEF string back to a float32 embedding vector."""
    # Add padding if needed
    padding = 4 - len(encoded) % 4
    if padding != 4:
        encoded += '=' * padding

    # Step 1: Decode
    compressed = base64.urlsafe_b64decode(encoded)

    # Step 2: Decompress
    payload = gzip.decompress(compressed)

    # Step 3: Dequantize
    v_min, v_max = struct.unpack('ff', payload[:8])
    quantized = np.frombuffer(payload[8:], dtype=np.int8)
    vector = (quantized.astype(np.float32) + 128) / 255 * (v_max - v_min) + v_min

    return vector
```

### JavaScript

```javascript
async function cefEncode(vector) {
  // Step 1: Quantize
  const min = Math.min(...vector);
  const max = Math.max(...vector);
  const range = max - min;

  const metadata = new Float32Array([min, max]);
  const quantized = new Int8Array(vector.length);
  for (let i = 0; i < vector.length; i++) {
    quantized[i] = Math.round(((vector[i] - min) / range) * 255) - 128;
  }

  // Combine metadata + quantized
  const payload = new Uint8Array(8 + quantized.length);
  payload.set(new Uint8Array(metadata.buffer), 0);
  payload.set(new Uint8Array(quantized.buffer), 8);

  // Step 2: Compress (using CompressionStream API)
  const stream = new Blob([payload]).stream().pipeThrough(new CompressionStream('gzip'));
  const compressed = await new Response(stream).arrayBuffer();

  // Step 3: Encode (base64url)
  const base64 = btoa(String.fromCharCode(...new Uint8Array(compressed)));
  return base64.replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}
```
