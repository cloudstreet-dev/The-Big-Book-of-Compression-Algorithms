# Chapter 4: Dictionary Methods — LZ77, LZ78, LZW

In 1977 and 1978, Abraham Lempel and Jacob Ziv published two papers that fundamentally changed compression. Their insight was different from Huffman's: instead of assigning shorter codes to frequent symbols, replace *repeated strings* with references to where they appeared before.

This is the *dictionary* approach: build a dictionary of previously seen data and encode new data as references into that dictionary. It's the foundation of zip, gzip, zlib, PNG, WebP, and dozens more formats.

## Why Dictionary Methods Work

Text, code, and many binary formats are full of repetition:
- Words and phrases repeat in text
- Function names repeat in source code
- Packet headers repeat in network captures
- UI element descriptions repeat in HTML

Statistical coders (Huffman, arithmetic) can only exploit *symbol frequency* — if 'e' is common, give it a short code. But they can't directly exploit the pattern "the " appearing 10,000 times in a document. Dictionary methods can: encode "the " as a single token referencing its first occurrence.

```
Example: "abcdeabcde"

Statistical coder sees: 10 mostly-equal-frequency symbols.
Not much to gain; each symbol might get ~3 bits.
Total: ~30 bits.

Dictionary coder sees:
  "abcde" → encode literally (5 symbols)
  "abcde" → reference to 5 characters back, length 5

  Output: literal "abcde", then reference (offset=5, length=5)
  Potentially much shorter!
```

## LZ77: The Sliding Window

LZ77 (1977) uses a *sliding window*: a buffer of recently seen data that serves as the dictionary. When encoding, the algorithm looks backward in the window for the longest match to the current position.

```
LZ77 window structure:

  ◄──── Search buffer (past data) ────►◄── Lookahead buffer ──►
  │ already encoded / "dictionary"    │ data to encode next    │
  └───────────────────────────────────┴────────────────────────┘
         typically 32KB or 64KB              typically 258 bytes

Current position is the boundary between the two.
```

**Encoding a position**: Find the longest string in the search buffer that matches the beginning of the lookahead buffer. Output a triple:

```
(offset, length, next_char)
  offset: how far back the match starts
  length: how long the match is
  next_char: the first character after the match (that didn't match)
```

If no match found, output `(0, 0, current_char)` — a literal.

### Step-by-Step Example

Encoding `"AABCBBABC"` with a window size of 8:

```
Position 0: Lookahead = "AABCBBABC"
  Search buffer is empty. No match possible.
  Output: (0, 0, 'A')   [literal A]

Position 1: Lookahead = "ABCBBABC"
  Search buffer: "A"
  Match: 'A' at offset 1, length 1
  Next char after match: 'B'
  Output: (1, 1, 'B')   [go back 1, copy 1 char, then 'B']

Position 3: Lookahead = "CBBABC"
  Search buffer: "AAB"
  No match for 'C' in buffer.
  Output: (0, 0, 'C')   [literal C]

Position 4: Lookahead = "BBABC"
  Search buffer: "AABC"
  'B' found at offset 2 (buffer position 2 from right).
  Match length 1. Next char: 'B'
  Output: (2, 1, 'B')

Position 6: Lookahead = "ABC"
  Search buffer: "AABCBB"
  'A' at various positions. Let's check for "AB":
    "AB" found starting at offset 5 (the "AB" in "AABC").
  Then 'C' follows — does the match extend to "ABC"? Yes!
  Output: (5, 3, '') — but we're at the end, so maybe (5, 3)
  (In practice, a special end-of-stream marker is used)
```

Full encoded sequence: `(0,0,'A'), (1,1,'B'), (0,0,'C'), (2,1,'B'), (5,3,...)`

Without accounting for encoding overhead, we encoded 9 characters using 5 tokens — some savings, more dramatic on longer repeated sequences.

### Finding Matches Efficiently

Naively, finding the longest match requires scanning the entire search buffer for each position — O(window_size × lookahead_size) per position, which is very slow for a 32KB window.

Real LZ77 implementations use hash tables to find candidate match positions in O(1), then verify and extend. DEFLATE uses a hash on the first 3 characters of the lookahead to find candidates.

```python
def lz77_encode(data: bytes, window_size: int = 255, max_match: int = 255):
    """Simple LZ77 encoder (pedagogical, not optimized)."""
    i = 0
    tokens = []

    while i < len(data):
        best_offset = 0
        best_length = 0

        # Search backward in the window
        search_start = max(0, i - window_size)

        for j in range(search_start, i):
            # How long is the match starting at j vs i?
            length = 0
            while (length < max_match
                   and i + length < len(data)
                   and data[j + length] == data[i + length]):
                length += 1
                # Handle overlapping matches (length can exceed i - j)
                if j + length >= i:
                    # Wrap around: copy from j modulo current i
                    pass  # simplified: stop at boundary for clarity

            if length > best_length:
                best_length = length
                best_offset = i - j

        if best_length > 2:  # Only use reference if it saves bits
            next_char = data[i + best_length] if i + best_length < len(data) else None
            tokens.append(('ref', best_offset, best_length, next_char))
            i += best_length + (1 if next_char is not None else 0)
        else:
            tokens.append(('lit', data[i]))
            i += 1

    return tokens
```

## LZ78: The Growing Dictionary

LZ78 (1978) takes a different approach: instead of a sliding window over past data, it builds an explicit dictionary of phrases encountered so far.

**The dictionary grows with the data.** When a new string is seen:
1. Look it up in the dictionary.
2. If found, extend the lookup by one more character.
3. When the extension isn't found, output the longest found phrase's index plus the new character, and add the new phrase to the dictionary.

```
LZ78 encoding of "AABABCABABCABC":

Dictionary starts empty.

Char 'A': Not in dict. Output (0, 'A'). Add "A" → entry 1.
Dict: {1:"A"}

Char 'A': In dict (entry 1). Try to extend.
Char 'B': "AB" not in dict. Output (1, 'B'). Add "AB" → entry 2.
Dict: {1:"A", 2:"AB"}

Char 'A': Entry 1. Extend.
Char 'B': "AB" is entry 2. Extend.
Char 'C': "ABC" not in dict. Output (2, 'C'). Add "ABC" → entry 3.
Dict: {1:"A", 2:"AB", 3:"ABC"}

Char 'A': Entry 1. Extend.
Char 'B': "AB" is entry 2. Extend.
Char 'A': "ABA" not in dict. Output (2, 'A'). Add "ABA" → entry 4.
...

Output sequence: (0,'A'), (1,'B'), (2,'C'), (2,'A'), (3,'B'), (3,'C')
```

LZ78's advantage: the dictionary can capture patterns from anywhere in the past, not just the most recent window. Its disadvantage: the dictionary can grow unbounded (or must be reset), and it's harder to implement efficiently.

## LZW: LZ78 Without the Literal

LZW (Lempel-Ziv-Welch, 1984) simplifies LZ78 by initializing the dictionary with all single characters, eliminating the need to transmit literals.

**Key change**: Output is *just* the dictionary index, never a raw character. Since all single characters are pre-loaded, every output is an index reference.

LZW became famous because it was used in:
- **GIF** (the primary reason GIF images exist)
- **TIFF** (optional compression)
- **PDF** (early versions)
- Unix **compress** utility

It was also famously patented by Unisys, which caused the GIF controversy of the 1990s and drove developers toward PNG (which uses DEFLATE instead).

```
LZW encoding of "ABABABAB":

Initialize dict: {'A':0, 'B':1, 'C':2, ..., 'Z':25}
                 (all 256 ASCII values for binary data)
Next code: 256

i=0: Read 'A'. Match so far: "A"
     Read 'B'. "AB" not in dict.
     Output: 0 (code for 'A')
     Add "AB" → 256. Match = "B"

i=1: Match: "B". Read 'A'. "BA" not in dict.
     Output: 1 (code for 'B')
     Add "BA" → 257. Match = "A"

i=2: Match: "A". Read 'B'. "AB" is in dict! (code 256). Match = "AB"
     Read 'A'. "ABA" not in dict.
     Output: 256 (code for "AB")
     Add "ABA" → 258. Match = "A"

...

Output: 0, 1, 256, 256, ...
```

The elegant trick in LZW: the decompressor can reconstruct the dictionary in sync with the encoder, because each dictionary entry is added exactly when the encoder adds it. The decompressor needs no dictionary transmitted separately.

## How ZIP Actually Works

ZIP files use DEFLATE compression (Chapter 5), but understanding how a zip container is structured illuminates how dictionary compression is packaged for real use.

```
ZIP file structure:

┌─────────────────────────────────────────────────────────────┐
│  Local file header + compressed data (for each file)        │
│  ┌──────────────────┬──────────────────────────────────┐    │
│  │ Local File Header│ Compressed file data              │    │
│  │ - Magic: PK\x03\x04                                 │    │
│  │ - Version needed                                    │    │
│  │ - Compression method (0=store, 8=DEFLATE)           │    │
│  │ - Last-modified time/date                           │    │
│  │ - CRC-32                                            │    │
│  │ - Compressed size                                   │    │
│  │ - Uncompressed size                                 │    │
│  │ - Filename length + filename                        │    │
│  └──────────────────┴──────────────────────────────────┘    │
│  ... (repeated for each file)                               │
│                                                             │
│  Central Directory (one entry per file)                     │
│  - Same metadata as local headers                           │
│  - Offset to local header in archive                        │
│  - Extra: file comment, external attributes                 │
│                                                             │
│  End of Central Directory Record                            │
│  - Offset and size of central directory                     │
│  - Archive comment                                          │
└─────────────────────────────────────────────────────────────┘
```

Key design decisions in ZIP:
- **Per-file compression**: Each file is compressed independently. This allows random access to individual files without decompressing the whole archive.
- **No archive-level dictionary**: Files don't share a dictionary. This is why `.tar.gz` (compress all, then archive) often beats `.zip` for collections of similar files — tar first concatenates everything, then gzip can find cross-file patterns.
- **Central directory at the end**: Allows ZIP writers to stream (compress file, then seek back and write metadata), and allows quick listing without decompressing data.

```python
import zipfile
import io

# Creating a ZIP in Python
buf = io.BytesIO()
with zipfile.ZipFile(buf, 'w', compression=zipfile.ZIP_DEFLATED) as zf:
    zf.writestr('hello.txt', 'Hello, world! ' * 1000)
    zf.writestr('data.csv', 'id,name,value\n' * 500)

# Reading it back
buf.seek(0)
with zipfile.ZipFile(buf, 'r') as zf:
    print(zf.namelist())           # ['hello.txt', 'data.csv']
    print(zf.getinfo('hello.txt').compress_size)  # much smaller than original
```

## Comparing the LZ Variants

```
Algorithm   Dictionary    Window       Best for
──────────────────────────────────────────────────────────────────
LZ77        Sliding       Recent data  Streaming, simple decoder
LZ78        Growing       All past     Better adaptation over time
LZW         Growing       All past     Simple implementation, patents
DEFLATE     Sliding+Hash  32KB         Universal (see Chapter 5)
LZMA        Sliding+PPM   Up to 4GB    Maximum compression (Ch 6)
```

The LZ77 family (sliding window) is much more common in practice because:
1. Bounded memory: a fixed-size window means predictable memory use
2. Simple decoder: just copy bytes from the output buffer
3. Better for streaming: doesn't require dictionary reset

The LZ78 family excels when data has patterns that are far apart, but the unbounded dictionary is a practical challenge.

## The Asymmetry That Matters

One of the most important properties of LZ-based compression: **decompression is much faster than compression**.

Compression requires searching for matches — expensive. Decompression just follows back-references — basically a `memcpy`. This asymmetry is intentional and valuable: you compress once (or slowly in batch), you decompress many times (fast, on demand).

```
Typical LZ77-family speed asymmetry:

gzip    compress:  ~25 MB/s    decompress:  ~250 MB/s   (10× asymmetry)
Zstd    compress: ~400 MB/s    decompress: ~1200 MB/s   (3× asymmetry)
LZ4     compress: ~700 MB/s    decompress: ~4000 MB/s   (6× asymmetry)
```

LZ4 is interesting: it's designed so that decompression is the bottleneck for *nothing* — it's fast enough that you can decompress data faster than RAM can deliver it on most systems. This is the design goal for in-memory and database compression.

## Summary

- LZ77 uses a sliding window as a dictionary; references are `(offset, length)` pairs.
- LZ78 builds an explicit, growing dictionary of seen phrases.
- LZW pre-initializes the dictionary and eliminates explicit literals.
- ZIP uses DEFLATE (LZ77 + Huffman) per file, enabling random access at the cost of cross-file compression.
- The asymmetry between compression and decompression speed is a feature, not a bug.

The next chapter shows how modern compressors — DEFLATE, Brotli, and Zstandard — took the LZ77 core and built major improvements around it.
