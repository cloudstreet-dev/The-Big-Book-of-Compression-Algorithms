# Chapter 12: Network Compression

Network compression has a constraint that no other compression context has: both sender and receiver must agree on the algorithm, negotiate it at connection time, and do it all quickly enough not to increase latency. The compression must be faster than the network would take without it — otherwise you've made things slower.

This creates a different set of tradeoffs. The question isn't just "what compresses best?" but "what compresses fast enough that the savings in transmission time outweigh the cost of compression itself?"

## The Break-Even Analysis

For compression to help, the time saved transmitting fewer bytes must exceed the time spent compressing:

```
Break-even calculation:

Original size:      S bytes
Compressed size:    C bytes (where C < S)
Network bandwidth:  B bytes/second
Compression speed:  V bytes/second (throughput of compressor)

Time to send without compression:  S / B
Time with compression:             S / V  +  C / B

Compression helps when:
  S / V  +  C / B  <  S / B
  S / V  <  S/B - C/B = (S-C)/B
  S / V  <  (S-C) / B
  B      <  V × (S-C)/S = V × compression_savings_ratio

Compression always helps if B << V (slow network, fast compressor).
Compression never helps if B >> V (fast network, slow compressor).
```

For a 1 Gbps LAN with gzip at 25 MB/s: compression speed (200 Mbps) << network speed (1 Gbps). Gzip would *hurt*. For a 10 Mbps WAN connection with gzip at 25 MB/s (200 Mbps): network is the bottleneck, gzip helps dramatically.

This is why HTTP compression is standard for internet-facing services but rare for internal datacenter traffic (unless the data is very large or the network is saturated).

## HTTP Content Negotiation

HTTP compression is opt-in and negotiated per-request. The client advertises what it supports; the server picks what it sends.

```
HTTP compression negotiation:

Client request:
  GET /api/data HTTP/1.1
  Host: example.com
  Accept-Encoding: gzip, deflate, br, zstd

Server response:
  HTTP/1.1 200 OK
  Content-Encoding: br
  Content-Type: application/json
  Content-Length: 4832

  [Brotli-compressed body]

Client: decompress Brotli body, present to application.
```

**`Content-Encoding`** specifies the compression applied to the response body. Multiple encodings can be chained (rarely used):
```
Content-Encoding: gzip, identity
```

**`Accept-Encoding`** priorities: Clients can give quality factors: `Accept-Encoding: gzip;q=1.0, br;q=0.9, identity;q=0.5`. In practice, most servers ignore q-factors and just pick the best supported algorithm.

### gzip in HTTP

gzip is the universal HTTP compression format — supported since HTTP/1.1 and by essentially every client and server in existence. Nginx and Apache enable it with:

```nginx
# nginx gzip configuration
gzip on;
gzip_comp_level 6;          # Level 1-9; 6 is the sweet spot
gzip_min_length 1000;       # Don't compress tiny responses
gzip_types
    text/plain
    text/css
    text/javascript
    application/javascript
    application/json
    application/xml
    image/svg+xml;

# Don't compress responses that are already compressed
# (JPEG, PNG, MP4 etc. are not in the list above)
```

**Pitfall**: Compressing already-compressed data wastes CPU and sometimes *increases* size. The types list should include only text-based formats.

### Brotli in HTTP

Brotli (`br`) is supported by all major browsers (since 2016/2017) and produces ~15-25% better compression than gzip on typical web content.

```nginx
# nginx Brotli configuration (requires ngx_brotli module)
brotli on;
brotli_comp_level 6;        # 0-11; 6 is reasonable default
brotli_min_length 1000;
brotli_types text/plain text/css application/javascript application/json;
```

Brotli is slower to compress than gzip at equivalent levels, making it more suitable for:
- **Pre-compressed static assets**: Compress at build time, serve from cache. `file.js.br` stored alongside `file.js.gz`.
- **Moderately dynamic content**: Where the compression overhead is acceptable.

```bash
# Pre-compress static assets
for f in dist/*.js dist/*.css; do
  brotli --best --output "${f}.br" "$f"
  gzip --best --keep --output "${f}.gz" "$f"
done
```

The server then serves `.br` or `.gz` based on `Accept-Encoding`, without compressing on the fly.

### Zstd in HTTP

Zstd HTTP compression is specified in RFC 8878 (2021) and increasingly supported:
- Chrome 118+ (2023)
- Firefox 116+ (2023)
- Safari (as of 2024)

```
Compression ratio comparison on a typical 100KB JSON API response:

Encoding     Size      Ratio    Encode time
──────────────────────────────────────────────
None         100KB     1.0×     —
gzip-6       28KB      3.6×     4ms
br-6         24KB      4.2×     12ms
zstd-6       26KB      3.8×     1ms

For dynamic API responses where compress-on-the-fly matters:
  zstd wins: better than gzip, much faster than Brotli.

For pre-compressed static assets:
  Brotli wins: compress once offline, better ratio.
```

Zstd also has a killer feature for HTTP: **trained dictionaries**. If you pre-train a dictionary on samples of your API responses, Zstd can compress small JSON responses to a fraction of their gzip size.

```python
# Zstd shared compression dictionary for API responses
import zstandard as zstd
import json

# Training (offline, during deployment)
samples = [json.dumps(resp).encode() for resp in sample_api_responses]
dict_data = zstd.train_dictionary(32768, samples)  # 32KB dictionary
dict_data.write_to_file('api_response.dict')

# Server-side compression (with dictionary)
cdict = zstd.ZstdCompressionDict(dict_data.as_bytes())
compressor = zstd.ZstdCompressor(dict_data=cdict, level=3)

def compress_api_response(data: dict) -> bytes:
    return compressor.compress(json.dumps(data).encode())

# Client must have the same dictionary (served once, cached)
# This is nonstandard - Zstd dict IDs would need custom negotiation
# In practice, shared dicts are used for specific known clients (mobile apps)
```

## HPACK: HTTP/2 Header Compression

HTTP headers are surprisingly expensive. A typical HTTP request has 500-800 bytes of headers (cookies, User-Agent, Accept, Authorization, etc.), and for short API responses, headers can be larger than the body.

HTTP/2 introduced HPACK (Header Compression for HTTP/2, RFC 7541) specifically for this problem.

### HPACK's Approach

HPACK is not a generic compression algorithm — it's purpose-built for HTTP headers. It uses:

1. **Static table**: 61 predefined header name-value pairs (`:method GET`, `:status 200`, `content-type application/json`, etc.)
2. **Dynamic table**: A FIFO buffer of recently seen header name-value pairs, shared between encoder and decoder
3. **Huffman coding**: A static Huffman table for header values (based on frequency of characters in HTTP headers)

```
HPACK encoding examples:

:method: GET (static table entry #2):
  Representation: 0x82  (1 byte! Full header in 1 byte)

content-type: text/html (static table entry #31 for name only):
  Literal header, indexed name, literal value
  0x5f 0x1d [huffman-encoded "text/html"]

Authorization: Bearer eyJhbGc...  (not in tables):
  Literal header, incremental indexing (add to dynamic table)
  [huffman-encoded "authorization"] [huffman-encoded "Bearer eyJ..."]
  Future requests reuse this from the dynamic table.
```

The result: many common headers compress to 1-3 bytes. A typical HTTPS API request that would have 800 bytes of headers in HTTP/1.1 might have 100-200 bytes in HTTP/2 after the first request (once the dynamic table is populated).

### HPACK Security: CRIME and BREACH

HPACK doesn't allow request headers and response bodies to share a context — this prevented the CRIME attack (2012), where a TLS+gzip combination allowed attackers to recover session cookies by observing compressed request sizes.

If the compression context is shared between attacker-controlled input and secret data, an attacker can determine the secret by crafting inputs that compress differently based on whether they share patterns with the secret. HPACK addresses this by separating header tables from body compression.

## HTTP/3 and QPACK

HTTP/3 runs over QUIC (UDP-based) instead of TCP. HPACK had a limitation: it required headers to be processed in order (head-of-line blocking). QPACK (RFC 9204) fixes this for HTTP/3, allowing headers to be processed out of order while maintaining equivalent compression.

## When NOT to Compress in Transit

### Already-Compressed Content

Never compress data that's already compressed. The overhead of attempting to compress JPEG, MP4, ZIP, GZIP, or any already-compressed format is pure waste:

```
Attempting to gzip an already-gzipped response:

Original JSON: 50KB
gzip compressed: 15KB
"Re-gzip" the gzip stream: 15.1KB  (slightly larger due to headers!)
CPU cost: 2× (compress + decompress the inner layer)
```

Configure your server to not compress `Content-Type: image/*`, `video/*`, `audio/*`, `.zip`, `.gz`, `.br`, `.zst`, `.7z`, and similar binary formats.

### Small Responses

For responses under ~1KB, the overhead of compression headers and minimum compressor block sizes can exceed the savings:

```
Compression overhead for small responses:

gzip headers + trailer: ~18 bytes
For a 200-byte response:
  Compressed: 150 bytes + 18 overhead = 168 bytes
  Improvement: 200 → 168 (16% savings, cost: CPU)

For a 50-byte response:
  Compressed: 45 bytes + 18 overhead = 63 bytes
  LARGER than original! Don't compress.

gzip_min_length 1000 (nginx default) is a reasonable threshold.
```

### High-Traffic Real-Time APIs

For APIs handling thousands of requests per second with sub-10ms response time requirements, on-the-fly compression at gzip-6 can add 2-5ms of latency. Options:

1. **Pre-compress**: For cacheable responses, compress once and cache the compressed response.
2. **Lower compression level**: gzip-1 is 3-5× faster than gzip-6 with 10-20% worse ratio.
3. **Zstd-1**: Faster than gzip-1, better ratio than gzip-6.
4. **No compression**: For internal microservice communication on high-speed networks.

### TLS and Compression (CRIME)

After the CRIME and BREACH vulnerabilities, compressing HTTPS responses that include both attacker-controlled content and sensitive cookies/tokens requires care. BREACH showed that even response body compression (not just header compression) can leak secrets if:
- The response contains a secret (CSRF token, session ID)
- The response also contains attacker-controlled data (reflected parameter)
- The response is compressed

BREACH mitigations: disable compression on mixed-content responses, or use response padding/masking. For most modern applications that don't reflect user input in responses containing secrets, the risk is low — but understanding it matters for security-sensitive APIs.

## Compression in Transit vs. At Rest

A common confusion: **content-encoding** (transit compression) and **storage compression** are orthogonal:

```
Content lifecycle and compression:

  Origin server                  CDN / Cache               Client
  ┌──────────┐                  ┌──────────┐              ┌──────────┐
  │ file.js  │ ────Brotli────▶  │ file.js  │ ─Brotli──▶  │ file.js  │
  │(at rest) │   (in transit)   │ (stored  │  (in transit)│(in memory│
  │   zstd   │                  │ as .br   │              │ uncompressed)
  └──────────┘                  └──────────┘              └──────────┘

At rest:       zstd (storage format, high compression, rarely accessed)
In transit:    Brotli (browser understands this encoding)
In memory:     uncompressed (browser needs raw JS to parse and execute)
```

A CDN might:
- Store assets at rest in an efficient format (Zstd, high level)
- Decompress and re-compress as Brotli when serving to browsers that support it
- Decompress and re-compress as gzip when serving to older browsers

Or better: pre-generate `.br` and `.gz` versions at build time, store all three, serve the appropriate one based on `Accept-Encoding`.

## Summary

- Compress when network bandwidth << compressor throughput. Don't compress when bandwidth is fast or compressor is slow.
- **gzip** is universal for HTTP; use level 6 as the default.
- **Brotli** is 15-25% better than gzip on web content; ideal for pre-compressed static assets.
- **Zstd** is the modern choice for dynamic API compression — faster than gzip, better than Brotli at equivalent speed.
- **HPACK** compresses HTTP/2 headers using static+dynamic tables and Huffman coding; common headers drop to 1-3 bytes.
- Never compress already-compressed data types (JPEG, MP4, ZIP, etc.).
- Small responses (<1KB) often expand under compression due to format overhead.
- Understand CRIME/BREACH before enabling compression on responses mixing user input with secrets.

Next: putting it all together with a decision framework for choosing the right algorithm for your context.
