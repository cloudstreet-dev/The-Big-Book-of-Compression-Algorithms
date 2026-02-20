# Chapter 5: Modern Lossless — Deflate, Brotli, Zstandard

The LZ family gave us the dictionary method. Huffman coding gave us the entropy stage. The insight of DEFLATE was: combine them. Use LZ77 to remove repetition, then Huffman to compress the residue. The result became the compression workhorse of the internet — used inside gzip, zlib, PNG, PDF, and zip files.

Brotli and Zstandard improved on DEFLATE with better probability modeling, larger dictionaries, and faster algorithms. Zstd in particular has become the default choice for new systems, with a design that's both faster and more compressible than gzip.

## DEFLATE: LZ77 + Huffman

DEFLATE was defined by Phil Katz in 1991 (for PKZIP) and later formalized in RFC 1951. It's not one algorithm — it's a carefully designed combination:

```
DEFLATE pipeline:

  Raw data
     │
     ▼
  LZ77 matching
  (find back-references in 32KB sliding window)
     │
     ▼
  Token stream: literals and (offset, length) pairs
     │
     ▼
  Huffman encoding
  (two trees: one for literals/lengths, one for offsets)
     │
     ▼
  Compressed bitstream
```

### The Two Huffman Trees

DEFLATE uses two Huffman trees simultaneously:

**Literal/Length tree**: Encodes 286 possible values.
- 0–255: literal byte values
- 256: end-of-block marker
- 257–285: match lengths (lengths 3–258, with some range-coded)

**Distance tree**: Encodes 30 possible distance codes (representing the back-reference offset). Short distances get short codes; long distances (near 32KB) get longer codes.

```
DEFLATE symbol types:

  Literal 'A' (0x41):
  ┌──────────┐
  │  0x41    │ → Literal/Length Huffman code for 0x41
  └──────────┘

  Back-reference (offset=7, length=5):
  ┌──────────┬─────────────┐
  │ Len code │  Dist code  │
  │  (261)   │    (4)      │
  └──────────┴─────────────┘
  Length 261 means "length 5" (with no extra bits)
  Distance 4 means "offset 7-8" (with 1 extra bit)
```

### Block Types

DEFLATE has three block types, and the compressor chooses per-block which to use:

- **Type 0 (stored)**: No compression. Used when the input is already compressed or random.
- **Type 1 (fixed Huffman)**: Uses predefined, hardcoded Huffman codes. Fast to encode, slightly suboptimal.
- **Type 2 (dynamic Huffman)**: Stores custom Huffman trees at the start of each block, then uses them. Optimal but has header overhead.

```
gzip file structure:

┌─────────────────────────────────────────────────────┐
│ gzip header (10 bytes)                              │
│  - Magic: \x1f\x8b                                  │
│  - Compression method: 8 (DEFLATE)                  │
│  - Flags, modification time, OS                     │
├─────────────────────────────────────────────────────┤
│ DEFLATE compressed data (multiple blocks)           │
│  Block 1: [type][Huffman tables if type 2][data]    │
│  Block 2: [type][Huffman tables if type 2][data]    │
│  ...                                                │
│  Final block: BFINAL bit set                        │
├─────────────────────────────────────────────────────┤
│ CRC-32 of original data (4 bytes)                   │
│ Original size mod 2^32 (4 bytes)                    │
└─────────────────────────────────────────────────────┘
```

### DEFLATE's Limitations

DEFLATE was designed in 1991 and it shows:
- **32KB window**: Modern files are often megabytes or gigabytes. Patterns farther than 32KB apart can't be referenced.
- **Huffman entropy coding**: As we saw in Chapter 3, arithmetic coding and ANS can do better for skewed probabilities.
- **Static two-tree model**: The Huffman trees are trained on the block, not on the specific statistics of literals vs. back-references.
- **Sequential only**: No parallelism; compressor must process the stream serially.

Despite these limitations, DEFLATE is everywhere. The decompressor is simple enough to implement in firmware, the format is standardized and stable, and it's good enough for most uses.

## Brotli: Google's Web Compressor

Brotli was released by Google in 2013 and standardized in RFC 7932 (2016). It's designed primarily for compressing web content (HTML, CSS, JS), and it has a trick that DEFLATE doesn't: **a built-in static dictionary**.

### The Static Dictionary

Brotli ships with a 120KB dictionary of common web strings. Things like `"<!DOCTYPE html>"`, `"Content-Type: "`, `"text/javascript"`, `"function "`, and thousands of common JavaScript, HTML, and CSS tokens.

References to this static dictionary can be made without the dictionary appearing in the compressed stream at all — the decompressor already has it. This gives Brotli a massive advantage on small web resources where a traditional compressor would waste most of its window learning the format.

```
Brotli static dictionary example:

Compressing: '<html lang="en">'

Without static dict:
  Encode each character literally, or hope it appears earlier in the window.

With Brotli static dict:
  '<html ' → static dict entry #1234 (immediate reference, no prior occurrence needed)
  'lang="en"' → static dict entry #5678
  '>' → literal

Much better compression for the first occurrence of common patterns.
```

### Brotli's Other Improvements

Beyond the static dictionary, Brotli improves on DEFLATE with:

- **Larger sliding window**: Up to 16MB (vs 32KB for DEFLATE)
- **Context modeling**: Uses 64 context maps, conditioning symbol probabilities on the context (previous bytes, literal vs. length/distance)
- **Distance ring buffer**: Tracks the four most recent distances; referencing a recent distance costs fewer bits
- **Better match finding**: More sophisticated algorithms for finding long matches

```
Compression ratios on web content (lower bits/byte = better):

Format                  HTML    JS     CSS    Average
────────────────────────────────────────────────────
Uncompressed (bytes)    100%   100%   100%    100%
gzip level 6             27%    32%    23%     27%
Brotli level 6           22%    26%    19%     22%
Brotli level 11          19%    23%    17%     20%
```

Brotli compresses better, but it's slower than gzip at comparable quality settings. This is acceptable for static assets that are compressed once and served many times. For dynamic content, gzip or Zstd are better choices.

**Brotli's decompression speed** is comparable to gzip — fast enough for real-time web serving.

## Zstandard: The New General Purpose

Zstandard (zstd) was developed at Facebook and open-sourced in 2016. It's designed to be:
- Fast at both compression and decompression
- Competitive with gzip at comparable quality
- Better than gzip at higher quality settings
- A drop-in replacement for zlib in many contexts

Zstandard has become the default compression algorithm for Linux kernels, Python wheel packages, Facebook's storage infrastructure, and Debian packages.

### Zstd's Core Architecture

```
Zstandard encoding pipeline:

  Raw data
     │
     ├──► LZ77 matching (larger window than DEFLATE, configurable)
     │
     ├──► Sequence encoding (literals + match tokens)
     │
     ├──► Literal compression (tANS/FSE entropy coding)
     │
     ├──► Sequence header compression (tANS for match commands)
     │
     ▼
  zstd frame (with optional frame header, checksums)
```

The key innovations in Zstd:

**tANS (Finite State Entropy)**: Zstd uses tANS instead of Huffman for entropy coding. As discussed in Chapter 3, this approaches arithmetic coding quality at Huffman speed.

**Training dictionaries**: Zstd supports *trained* dictionaries — if you're compressing many similar small files (like JSON API responses), you can train a dictionary on a sample set. References to the dictionary dramatically improve compression of small inputs.

**Configurable window size**: DEFLATE is stuck at 32KB. Zstd can use windows up to 2GB, enabling better compression of large files with distant repetitions.

**Parallel compression**: zstd can split the input into blocks and compress them in parallel using multiple cores (with the `--threads` flag). Output is still a single stream, fully compatible with single-threaded decompression.

### Zstd Compression Levels

Zstd's level system is broader than gzip's 1-9:

```
Zstd compression levels:

Level   Speed (encode)   Ratio      Notes
──────────────────────────────────────────────────────
 1      Very fast        Moderate   Competitive with LZ4
 3      Fast             Good       Default; beats gzip speed
 6      Moderate         Good       Near gzip -6 quality, faster
 9      Moderate         Better     Better than gzip -9
 12     Slow             Best fast  Starts using more memory
 19     Very slow        Excellent  Near Brotli quality
 22     Extremely slow   Maximum    (--ultra flag required)

Negative levels (e.g. -1, -5): faster than level 1, less compression.
Useful for real-time or log compression where CPU is the bottleneck.
```

```python
# Python example using zstandard library
import zstandard as zstd

# Basic compression
cctx = zstd.ZstdCompressor(level=3)
compressed = cctx.compress(data)

dctx = zstd.ZstdDecompressor()
original = dctx.decompress(compressed)

# Streaming (memory-efficient for large data)
cctx = zstd.ZstdCompressor(level=6)
with open('large_file.txt', 'rb') as fin:
    with open('large_file.zst', 'wb') as fout:
        cctx.copy_stream(fin, fout)

# With a trained dictionary (for many small files)
# Train on samples:
samples = [b'{"user":"alice","action":"login"}',
           b'{"user":"bob","action":"view"}',
           ...]  # many samples
params = zstd.ZstdCompressionParameters.from_level(3)
dict_data = zstd.train_dictionary(8192, samples)  # 8KB dict

# Compress with dictionary:
cctx = zstd.ZstdCompressor(dict_data=dict_data, level=3)
compressed = cctx.compress(b'{"user":"carol","action":"login"}')
# Much smaller than without dictionary!
```

### Why Zstd Won

```
Head-to-head: Zstd vs. gzip on a 1GB log file

Metric                   gzip -6      zstd -3      zstd -6
──────────────────────────────────────────────────────────
Compression time          28s          1.8s         4.2s
Compressed size           312MB        320MB        298MB
Decompression time         3.2s        0.9s         0.9s
Compression ratio          3.2×         3.1×         3.4×

Memory usage (compress)   256KB         8MB          16MB
Memory usage (decompress) 32KB          128KB        128KB
```

Zstd is ~15× faster to compress at comparable ratio, and ~3.5× faster to decompress. The cost is slightly higher memory usage — acceptable for almost every modern context.

For static web assets, Brotli at level 11 is better (higher compression ratio). For real-time or batch compression of dynamic data, Zstd is the choice.

## Putting It Together: When to Use What

```
Decision tree for lossless compression:

Is compatibility with old tools required?
├─ Yes → gzip/DEFLATE (ubiquitous)
└─ No → continue

Is compression speed the primary constraint?
├─ Yes (real-time, < 100ms) → LZ4 or Zstd level 1
└─ No → continue

Is decompression speed critical?
├─ Very (database, in-memory) → LZ4 or Snappy
└─ No → continue

Is the data primarily web content?
├─ Yes, static assets → Brotli level 6-11
├─ Yes, dynamic content → Zstd level 3 or gzip
└─ No → continue

Is maximum compression ratio needed?
├─ Yes, time doesn't matter → LZMA/7-zip (Chapter 6)
└─ No → Zstd level 6-9 (sweet spot)
```

## Under the Hood: The zlib API

Most code doesn't call DEFLATE directly — it calls zlib, the C library that wraps DEFLATE and exposes a streaming API. Understanding the zlib API helps when debugging compression issues:

```c
#include <zlib.h>

// Compress in one shot
int compress_buffer(const uint8_t *src, size_t src_len,
                    uint8_t *dst, size_t *dst_len) {
    z_stream strm = {0};
    deflateInit(&strm, Z_DEFAULT_COMPRESSION);  // level 6

    strm.next_in  = (Bytef*)src;
    strm.avail_in = src_len;
    strm.next_out = dst;
    strm.avail_out = *dst_len;

    int ret = deflate(&strm, Z_FINISH);
    *dst_len = strm.total_out;

    deflateEnd(&strm);
    return ret == Z_STREAM_END ? Z_OK : ret;
}
```

The streaming API (`deflate`/`inflate` called repeatedly) handles data larger than memory by processing in chunks. This is how nginx compresses HTTP responses on-the-fly without buffering the entire response body.

## Summary

- **DEFLATE** = LZ77 + Huffman, 32KB window. Universal, fast to decompress, shows its age.
- **Brotli** adds a static web dictionary and better entropy coding. Ideal for static web assets.
- **Zstandard** uses tANS entropy coding, configurable windows, and trained dictionaries. The modern default for general-purpose compression — faster than gzip, better compression at high levels.
- For most new systems: use Zstd. For web serving: serve Brotli for supporting clients, gzip for older ones.

Next up: LZMA and 7-Zip — what happens when you push lossless compression to its practical limit.
