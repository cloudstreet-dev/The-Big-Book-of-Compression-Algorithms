# Chapter 7: Run-Length Encoding and Simple Methods

Not every compression problem requires a sophisticated algorithm. Some data has structure so obvious that a simple encoder can compress it optimally. Run-length encoding is the oldest compression technique — simple enough to implement in an afternoon, yet still used in fax machines, printer languages, and as a preprocessing step in far more complex compressors.

This chapter covers the "simple" methods: RLE, delta encoding, and the Burrows-Wheeler Transform. None of these alone produces state-of-the-art compression, but they each illuminate an important principle, and BWT in particular is a beautiful rearrangement that enables bzip2's remarkable compression ratios.

## Run-Length Encoding (RLE)

The idea is almost embarrassingly simple: instead of encoding a run of repeated values literally, encode the value once and the count.

```
RLE example:

Input:   AAAAAABBBCCDDDDDDDDDD
         ──────╌──╌──────────
         6 A's, 3 B's, 2 C's, 10 D's

Output:  6A 3B 2C 10D

Compression: 21 bytes → 8 bytes
```

### The Encoding Ambiguity Problem

Naive RLE has an ambiguity problem: how do you tell runs from literals? If your encoded byte `6A` means "6 A's", what does `1B` mean — one B (literal), or a run of 1 B? And what if the data contains the digit '6'?

Different formats resolve this differently:

**Fixed format (always a pair)**: Always output (count, value). Problem: single non-repeated bytes still cost 2 bytes to encode.

```
Input: ABCAAAA
With fixed (count, value):
  Output: 1A 1B 1C 4A = 8 bytes (worse than 7-byte input!)
```

**Run/literal flag**: Use the high bit of the count byte to distinguish runs from literal sequences.

```
PackBits format (used in TIFF, Macintosh PICT):

  Count byte n:
    n = 0..127: copy next (n+1) bytes literally
    n = -1..-127 (signed, or 0x81..0xFF): repeat next byte (-n+1) times
    n = -128: no-op

  Input:  AAAAAABBBCCDDDDDDDDDD
  As PackBits:
    -5, A           → repeat A 6 times
    -2, B           → repeat B 3 times
    -1, C           → repeat C 2 times
    -9, D           → repeat D 10 times
  Output: 4 pairs = 8 bytes ✓
```

**Escape-based**: Use an escape character to signal a run. All non-escape bytes are literals.

```python
def rle_encode(data: bytes, min_run: int = 3) -> bytes:
    """
    RLE encoder. Uses escape byte 0x90 followed by count.
    Single occurrence of 0x90 in input is encoded as 0x90, 0x00.
    """
    ESCAPE = 0x90
    result = bytearray()
    i = 0

    while i < len(data):
        run_char = data[i]
        run_len = 1

        while i + run_len < len(data) and data[i + run_len] == run_char:
            run_len += 1

        if run_len >= min_run or run_char == ESCAPE:
            # Encode as run(s) — max run length 255
            while run_len > 0:
                chunk = min(run_len, 255)
                if chunk == 1 and run_char != ESCAPE:
                    result.append(run_char)
                else:
                    result.append(ESCAPE)
                    result.append(chunk)
                    result.append(run_char)
                run_len -= chunk
        else:
            # Encode literally
            result.extend(data[i:i+run_len])

        i += run_len

    return bytes(result)
```

### Where RLE Shines (and Doesn't)

RLE is optimal for highly repetitive data and terrible for random data:

```
Compression ratio by data type:

Data type              RLE performance
─────────────────────────────────────────────────
Solid color regions    Excellent (100:1 possible)
Black/white fax pages  Excellent (used in G3/G4 fax)
Palette-based graphics Good (large solid areas)
Natural photographs    Terrible (rarely adjacent pixels match)
Binary executable      Terrible to neutral
Compressed data        Terrible (already near-random)

Overhead for random data:
  A byte of escape value in the input → 2 bytes out. Up to 100% expansion.
```

**Group 4 fax compression** (the ITU-T standard for fax transmission) is essentially 2D RLE on black-and-white bit images, combined with a reference line. A typical fax page of text compresses from ~1MB to ~10KB — a 100:1 ratio — because pages of text are overwhelmingly white with small black runs for characters.

**BMP and TGA** image formats optionally use RLE for palette-based images where large areas share the same color.

## Delta Encoding

Delta encoding doesn't compress repeated values — it compresses *slowly changing* values. Instead of storing absolute values, store the difference between consecutive values.

```
Delta encoding of time-series data:

Raw timestamps (Unix epoch, seconds):
  1700000000
  1700000060
  1700000120
  1700000180
  ...

Deltas:
  1700000000  ← first value, store literally (4 bytes)
  60          ← delta, fits in 1 byte!
  60
  60
  ...

Savings: 4 bytes per value → 1 byte per value after the first.
```

Delta encoding is the foundation of many domain-specific compressors:

**Audio**: CD audio is 16-bit samples at 44,100 Hz. Consecutive samples differ by small amounts (audio signals are smooth). Delta encoding turns these into small numbers that compress well with RLE or Huffman. FLAC uses a generalization: it fits a polynomial to blocks of samples and stores the *residual* (error).

**Image compression**: PNG uses delta encoding as a *filter* before compression:
- Subtract adjacent pixels (`Sub` filter)
- Subtract from the pixel above (`Up` filter)
- Average of left and above (`Average` filter)
- Paeth predictor (best linear predictor from left, above, upper-left)

```
PNG filter example (Sub filter):

Original row:  100 102 105 108 112 115
Filtered row:  100   2   3   3   4   3  (subtract previous pixel)

The filtered row has small values — easier to compress.
```

**Columnar databases**: Sorted timestamp or monotonically increasing ID columns compress extremely well with delta encoding + RLE.

```
Columnar delta + RLE example (user IDs in a sorted table):

User IDs (sorted): 1001, 1002, 1003, 1004, 1005, ...

After delta:  1001, 1, 1, 1, 1, ...

After RLE on the delta stream:
  1001 (literal), then (count=N-1, value=1)

Compression: (N × 4 bytes) → 8 bytes for any sorted sequential range.
```

## The Burrows-Wheeler Transform (BWT)

The Burrows-Wheeler Transform is remarkable: it's a *reversible rearrangement* of data that makes it much easier to compress. It doesn't compress anything by itself — but after a BWT, the data often compresses dramatically better with simple methods like RLE and Huffman.

BWT is the core of **bzip2**, which consistently beats gzip for text compression despite being simpler conceptually.

### How BWT Works

Given input string S, the BWT:
1. Forms all rotations of S (appended with a special end-of-string marker $)
2. Sorts them lexicographically
3. Outputs the last column of the sorted matrix

```
BWT of "banana$":

All rotations:
  banana$
  anana$b
  nana$ba
  ana$ban
  na$bana
  a$banan
  $banana

Sorted:
  $banana
  a$banan
  ana$ban
  anana$b
  banana$
  na$bana
  nana$ba

Last column (BWT output): annb$aa

```

The magic: the last column tends to have long runs of repeated characters, because rotations that sort together often differ only at the end — meaning the character *before* the current sorted context is often the same.

For "banana": after BWT, we get "annb$aa" — three runs of 2+ characters. Not impressive here, but on real text:

```
BWT effect on English text (first 100KB of War and Peace):

Before BWT: Character runs of length 1 dominate (~99%)
After BWT:  Character runs of length 3+ become common (~40%)

BWT output example fragment:
  ...eeeeeeeeeeeeeeeeeeetttttttttttttt...
  (long runs of common characters that follow similar contexts)
```

### Inverting the BWT

The inversion is the elegant part. Given only the last column (and the position of the original string's end-of-string marker), you can reconstruct the full original string.

The key insight: you can reconstruct the *first* column by sorting the last column. And the first column plus last column together give you enough to "walk" the permutation:

```python
def bwt_encode(s: str) -> tuple[str, int]:
    """Compute BWT. Returns (transformed, index of original)."""
    s = s + '\x00'  # Append null byte as end-of-string marker
    n = len(s)
    rotations = sorted(s[i:] + s[:i] for i in range(n))
    bwt = ''.join(r[-1] for r in rotations)
    idx = rotations.index(s[1:] + s[0])  # index of sorted rotation starting with original
    return bwt, idx

def bwt_decode(bwt: str, idx: int) -> str:
    """Invert BWT. Returns original string (without end-of-string marker)."""
    n = len(bwt)
    # Build table of (char, original_position) pairs
    table = sorted((char, i) for i, char in enumerate(bwt))
    # Walk the permutation
    row = idx
    result = []
    for _ in range(n):
        char, row = table[row]
        result.append(char)
    return ''.join(reversed(result))[1:]  # strip the null byte
```

### The Full bzip2 Pipeline

bzip2 applies BWT as part of a pipeline:

```
bzip2 compression pipeline:

  Raw data
     │
     ▼
  Run-Length Encoding (pass 1)
  - Pre-compress runs of 4+ identical bytes
  - Prevents the BWT from sorting very long runs badly
     │
     ▼
  Burrows-Wheeler Transform
  - Sort all rotations, output last column
  - Groups similar contexts → long runs of same character
     │
     ▼
  Move-to-Front Transform
  - Convert characters to ranks in an MRU (Most Recently Used) list
  - Recently seen characters get small indices (0, 1, 2, ...)
  - Makes output dominated by small integers
     │
     ▼
  Run-Length Encoding (pass 2)
  - Compress the runs of 0s that MTF produces
     │
     ▼
  Huffman Coding
  - Entropy code the MTF+RLE output
  - bzip2 uses multiple Huffman tables, switching between them
     │
     ▼
  .bz2 file
```

### Move-to-Front (MTF)

MTF is the step that makes BWT+Huffman work well together. It converts characters into indices of an MRU list:

```
MTF encoding of "aaabbbcccaaa":

Initial list: [a, b, c, d, e, ...]  (all possible symbols)

Symbol 'a': index 0. Output 0. Move 'a' to front: [a,b,c,...]
Symbol 'a': index 0. Output 0. List unchanged.
Symbol 'a': index 0. Output 0.
Symbol 'b': index 1. Output 1. Move 'b' to front: [b,a,c,...]
Symbol 'b': index 0. Output 0. (b is now at front)
Symbol 'b': index 0. Output 0.
Symbol 'c': index 2. Output 2. Move 'c' to front: [c,b,a,...]
...

MTF output: 0, 0, 0, 1, 0, 0, 2, 0, 0, 2, 0, 0

The BWT output has long character runs.
MTF converts those runs to long runs of 0s.
RLE compresses the 0s.
Huffman codes the remaining non-zero values.
```

### bzip2 in Practice

```
bzip2 vs. gzip on common file types:

File type          gzip -6    bzip2    xz -6
────────────────────────────────────────────
English text        28%        22%      18%
C source code       22%        18%      15%
HTML files          25%        21%      17%
Binary executable   47%        40%      32%
JPEG (already cmp) 100%       100%     100%
```

bzip2 is typically 10-15% smaller than gzip on text, but compresses and decompresses more slowly. xz (LZMA) is typically another 15-20% smaller again.

bzip2 also has a useful property: its output consists of independent 900KB blocks, each with its own Huffman tables. This means:
- Parallel decompression is straightforward (process blocks independently)
- Corruption only destroys affected blocks, not the whole archive
- Random access is possible at block boundaries (with a block index)

## When Simple Methods Still Win

It's tempting to think that simple methods have been made obsolete by sophisticated ones. Not so:

**Sensor data**: IoT temperature sensors produce slowly-changing values. Delta encoding (1-2 bytes per reading) + Huffman easily beats generic lossless compression and is implementable on a microcontroller.

**Screen recordings / VNC**: Consecutive frames differ only in regions of mouse activity. A simple delta between frames (XOR, then RLE), followed by Huffman, achieves 50-100:1 compression with negligible CPU.

**Database WAL logs**: Write-ahead logs are sequences of delta operations (insert row, update field). Delta encoding the operations themselves, not the data, can achieve 10:1 compression.

**Palette images**: 8-bit-per-pixel images with large solid areas compress better with RLE than JPEG or even PNG deflate.

The general rule: if you can characterize the structure of your specific data, a targeted simple method beats a general-purpose complex one.

## Summary

- **RLE** encodes runs of repeated values as (count, value). Excellent for highly repetitive data, terrible for random data.
- **Delta encoding** stores differences between consecutive values. Excellent for slowly-changing sequences (timestamps, sensor data, sorted IDs).
- **BWT** rearranges data by sorting all rotations — a reversible transform that groups similar contexts and enables long runs of common characters.
- **MTF** converts character sequences into MRU indices, turning character runs into 0-runs that compress extremely well.
- **bzip2** chains RLE → BWT → MTF → RLE → Huffman into a surprisingly effective pipeline.
- Simple methods often beat general-purpose compressors for domain-specific data with known structure.

Next, we enter the world of lossy compression with images — where the interesting question shifts from "can we reconstruct it exactly?" to "what can we safely throw away?"
