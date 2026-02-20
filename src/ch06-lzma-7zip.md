# Chapter 6: LZMA and 7-Zip

LZMA (Lempel-Ziv-Markov chain Algorithm) is the compression algorithm behind 7-Zip's `.7z` format, and also used in `.xz` — the format that replaced `.bz2` in most Linux distributions. It achieves compression ratios that gzip can't touch, at the cost of significantly more time and memory.

Understanding LZMA means understanding what happens when you take every technique we've discussed and combine them: large sliding windows, Markov chain probability models, and range coding for entropy. The result is a compressor that's slow by design — but for cold storage, software distribution, and archiving, slow-to-compress is fine.

## The LZMA Architecture

LZMA is best understood as a three-layer system:

```
LZMA compression layers:

  Raw data
     │
     ▼
  ┌─────────────────────────────────┐
  │  Layer 1: LZ77 Matching         │
  │  - Sliding window (up to 4GB)   │
  │  - Match finder (hash chains,   │
  │    binary trees, or BT4)        │
  │  - Output: literal bytes +      │
  │    (offset, length) pairs       │
  └─────────────────────────────────┘
     │
     ▼
  ┌─────────────────────────────────┐
  │  Layer 2: Markov Model          │
  │  - Conditions each bit on       │
  │    context (recent bits,        │
  │    match/literal state machine) │
  │  - Hundreds of probability      │
  │    models, one per context      │
  └─────────────────────────────────┘
     │
     ▼
  ┌─────────────────────────────────┐
  │  Layer 3: Range Coding          │
  │  - Arithmetic-coding-equivalent  │
  │  - Each bit encoded using its   │
  │    model's probability estimate │
  └─────────────────────────────────┘
     │
     ▼
  .7z / .xz compressed stream
```

## Layer 1: The Large Sliding Window

DEFLATE uses a 32KB sliding window. Brotli goes up to 16MB. LZMA goes up to **4GB** (configurable, with common settings at 4MB–64MB for a balance of memory and compression).

Why does window size matter so much?

```
Example: Compressing a 10MB C source code archive

Pattern "    return NULL;" appears 847 times throughout the file.
The first occurrence is at byte 0, the last at byte 9,900,000.

DEFLATE (32KB window):
  First occurrence: encode literally
  Subsequent occurrences within 32KB: back-reference (cheap)
  After moving 32KB past last reference: must re-encode (expensive!)
  Result: can only reference the ~last 32,000 bytes.

LZMA (64MB window):
  First occurrence: encode literally
  ALL 847 subsequent occurrences: back-reference (cheap)
  Even occurrences 9MB later can reference the first.
```

For software distributions — which typically contain many similar files, headers, and boilerplate — large windows dramatically improve compression.

### Match Finders

Finding the best match in a large window requires sophisticated data structures. LZMA uses several:

**Hash chains** (fast, less optimal): Hash the first 2-3 characters of each potential match position; store chains of positions with the same hash.

**Binary trees** (BT3, BT4): For each hash bucket, maintain a binary tree sorted by the actual string content at each position. The `4` in BT4 means the first 4 bytes are hashed before tree lookup. This finds near-optimal matches but requires more memory and CPU.

```
Match finder complexity tradeoff:

Finder    Memory     Speed     Match quality
─────────────────────────────────────────────
HC4       Medium     Fast      Good
BT2       High       Slow      Excellent
BT3       Higher     Slower    Excellent
BT4       Highest    Slowest   Excellent (best for text)
```

## Layer 2: Markov Chain Models

This is where LZMA gets unusual. After the LZ77 stage produces a stream of tokens (literals and back-references), LZMA doesn't just Huffman-code them. It uses a rich set of *probability models*, one per *context*.

The insight: the probability of the next bit depends heavily on what state the encoder is in. A bit that's part of a match length is statistically different from a bit that's part of a literal. A bit that's the high bit of a match offset is different from a bit that's the low bit.

LZMA defines a finite state machine with states like:
- `LIT`: Last token was a literal
- `MATCH`: Last token was a match
- `LONGREP`: Last token was a "long repeat" match
- `SHORTREP`: Last token was a short repeat (single-byte repeat)

And within each state, each *bit position* within each field (literal byte, offset, length) has its own probability model.

```
LZMA state machine (simplified):

      ┌─────────────────────────────────────────┐
      │              STATES                     │
      │                                         │
      │  LIT ──────► MATCH ──────► LONGREP      │
      │   ▲           │   ▲         │  ▲        │
      │   │           ▼   │         ▼  │        │
      │   └────── SHORTREP ◄──── LONGREP1       │
      └─────────────────────────────────────────┘

Each state has its own probability tables for the next token type.
Each bit within each token field has its own probability model
conditioned on the current state + previously coded bits.

Total probability models: hundreds
```

This is why LZMA compresses so well — and why it's slow. The price of accurate probability models is maintaining and querying hundreds of them.

## Layer 3: Range Coding

The probability estimates from the Markov models feed directly into a range coder (the arithmetic coding variant from Chapter 3). Each bit is encoded using its own probability, approaching the Shannon limit.

Range coding here operates at the *bit* level (binary arithmetic coding): each bit is either 0 or 1, and the model gives the probability of 0. The range coder narrows its interval accordingly.

```python
# Simplified LZMA range coder core (pedagogical)
class RangeCoder:
    RANGE_TOP = 1 << 24
    MODEL_BITS = 11
    MODEL_TOTAL = 1 << MODEL_BITS  # 2048

    def __init__(self):
        self.low = 0
        self.range = 0xFFFFFFFF
        self.output = []

    def encode_bit(self, prob: int, bit: int):
        """
        Encode one bit using probability model.
        prob: probability of bit=0, in range [0, 2048)
        """
        bound = (self.range >> self.MODEL_BITS) * prob

        if bit == 0:
            self.range = bound
            # Update model toward 0 (increase prob of 0)
            prob += (self.MODEL_TOTAL - prob) >> 5
        else:
            self.low += bound
            self.range -= bound
            # Update model toward 1 (decrease prob of 0)
            prob -= prob >> 5

        # Renormalize
        while self.range < self.RANGE_TOP:
            self.output.append((self.low >> 24) & 0xFF)
            self.low = (self.low << 8) & 0xFFFFFFFF
            self.range <<= 8

        return prob  # return updated probability for next call
```

The model update (`prob += (MODEL_TOTAL - prob) >> 5`) is an exponential moving average — recent bits influence the probability more than old ones. The `>> 5` factor controls the adaptation rate: faster adaptation tracks local statistics, slower adaptation tracks global statistics.

## The 7-Zip Container Format (.7z)

LZMA (and its enhanced version LZMA2) is the primary codec in the `.7z` container format.

```
.7z file structure:

  ┌─────────────────────────────────────────────────┐
  │  Signature: '7z\xBC\xAF\x27\x1C' (6 bytes)     │
  │  Version + CRC of "Next Header Info" (12 bytes) │
  ├─────────────────────────────────────────────────┤
  │  Compressed data streams                        │
  │  - Optionally solid (all files in one stream)   │
  │  - Multiple codecs can be chained:              │
  │    e.g., BCJ (x86 filter) → LZMA2              │
  ├─────────────────────────────────────────────────┤
  │  Header (compressed)                            │
  │  - File list, sizes, timestamps, attributes     │
  │  - Offsets to each file's data in stream        │
  └─────────────────────────────────────────────────┘
```

**Solid archives**: A key 7-Zip feature is *solid* compression — all files are concatenated into one stream before compression. This allows the compressor to exploit similarities across files (e.g., many `.c` files sharing headers). Solid archives compress much better but require decompressing from the beginning to extract a single file.

```
Archive type comparison (100 similar text files, 10MB total):

              Ratio    Extract-single-file
──────────────────────────────────────────
zip (per-file)   40%    Fast (seek to offset)
7z non-solid     42%    Fast
7z solid         28%    Slow (decompress from start)
```

For software distribution archives (read-once, extract-all), solid is clearly better. For random-access archives (extract individual files frequently), non-solid or per-file compression wins.

### The BCJ Filter

Executable files (x86, ARM, etc.) contain instruction addresses — absolute memory addresses embedded in jump and call instructions. These addresses look random to a compressor, even though they have structure.

LZMA uses preprocessor filters (BCJ = Branch-Call-Jump filters) that convert these absolute addresses to relative ones before compression. Relative addresses are much more redundant (many calls are to nearby functions, so many relative addresses are small and similar).

```
BCJ filter effect on x86 binary:

Without BCJ:
  CALL instruction at offset 0x1000 targeting 0x5000:
    Bytes: E8 FB 3F 00 00   (relative: 0x3FFB = 0x5000 - 0x1000 - 5)
  CALL instruction at offset 0x1100 targeting 0x5000:
    Bytes: E8 FB 3E 00 00   (relative: 0x3EFB = 0x5000 - 0x1100 - 5)
  These two look different to the compressor.

With BCJ (convert to absolute):
  Both become: E8 00 50 00 00   (stores absolute target 0x5000)
  Now they're identical! Back-reference saves 4 bytes.
```

BCJ filters can reduce compressed executable size by 10-30% compared to raw LZMA.

## XZ: LZMA2 for Unix

The `.xz` format packages LZMA2 (an improved version of LZMA that handles multi-stream and partial block updates better) with a Unix-friendly wrapper:

```
.xz structure:

  ┌────────────────────────────────────────────┐
  │  Stream header                             │
  │  - Magic: \xFD7zXZ\x00                    │
  │  - Stream flags (check type: CRC32/CRC64) │
  ├────────────────────────────────────────────┤
  │  Blocks (LZMA2 compressed data)           │
  │  Each block has its own compressed size   │
  │  and uncompressed size                    │
  ├────────────────────────────────────────────┤
  │  Index (optional block index for seeking) │
  ├────────────────────────────────────────────┤
  │  Stream footer                            │
  │  - Size of index                          │
  │  - Stream flags                           │
  │  - Magic: YZ                              │
  └────────────────────────────────────────────┘
```

XZ replaced bz2 as the standard Linux source distribution format because it compresses significantly better at comparable memory use (bz2 has a 900KB memory limit that hurts compression on large files).

## The Real Cost: Memory and Time

The reason LZMA isn't used everywhere is cost:

```
Compression resource requirements:

                    Memory (compress)  Memory (decompress)  Time
─────────────────────────────────────────────────────────────────
gzip -6               256KB               32KB            1×
Brotli-6              8MB                 1MB             4×
Zstd-6                16MB               128KB            0.3×
xz -6                 97MB               65MB             20×
xz -9                 683MB              65MB             50×
```

xz at `-9` needs 683MB to compress and 65MB to decompress. For a server decompressing a package install, 65MB is fine. For an embedded device — it might not have 65MB free.

The decompression memory requirement is the key constraint for LZMA in embedded contexts. This is why most Linux distributions ship packages compressed with xz (because the build server has plenty of memory and time), but the package manager's memory footprint during installation is bounded.

## When LZMA/7-Zip Makes Sense

```
Use LZMA when:

✓ Cold storage or archival (compress once, rare access)
✓ Software distribution (compress once, download many times)
✓ Solid archives of similar files
✓ Target has adequate decompression memory (~64MB+)
✓ Compression time is not a constraint
✓ Maximizing compression ratio is the priority

Avoid LZMA when:

✗ Real-time or low-latency compression required
✗ Memory is severely constrained (< 64MB available)
✗ Data changes frequently (re-compression would be prohibitive)
✗ Random access to parts of the archive is needed
✗ Compression speed matters (CI/CD pipelines, logging, etc.)
```

## Tuning LZMA

The main knobs:

**Dictionary size** (`-d` in 7z, `--dict` in xz): Bigger window = better compression, more memory. Common values: 4MB, 8MB, 16MB, 64MB.

**Word size / fast bytes** (`-fb`): Maximum match length the match finder will consider. Higher = better matches, slower.

**Match cycles / depth** (`-mc`, `-md`): How hard to search for matches. Diminishing returns past a certain point.

```
xz preset breakdown:

Preset  Dict     Word   Depth   Memory
──────────────────────────────────────
-0      256KB    32     2        2MB
-1      1MB      32     3        4MB
-3      4MB      32     4       15MB
-6      8MB      64     4       97MB
-9      64MB     273   unlimited  683MB
```

## Summary

- LZMA combines large sliding windows (up to 4GB), hundreds of Markov chain probability models, and range coding.
- The Markov models condition each bit's probability on the encoder's current state.
- 7-Zip uses solid archives (compressing all files as one stream) to exploit cross-file redundancy.
- BCJ filters convert absolute addresses in executables to relative ones, dramatically improving compression.
- XZ wraps LZMA2 for Unix with multi-stream support and block-level CRC integrity checks.
- The cost: decompression needs ~64MB; compression can need hundreds of MB. Slow by design.

For everyday use, Zstd is a better choice. For software distribution archives where compression ratio is paramount and compress-once-decompress-many economics apply, LZMA is hard to beat.

Next, we step back to some simpler methods — run-length encoding and its friends — and find out where they still shine.
