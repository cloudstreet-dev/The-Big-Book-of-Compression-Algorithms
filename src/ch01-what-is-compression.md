# Chapter 1: What is Compression?

Before we can understand how compression algorithms work, we need to understand what they're working against: the information content of data itself. This is the domain of information theory, and it gives us something remarkable — a mathematical lower bound on how small any compressor can make a given piece of data.

## The Basic Idea

Compression is the process of encoding information using fewer bits than the original representation. That's it. Everything else is detail about *how* to do it well and *when* the tradeoffs make sense.

Consider the string `"AAAAAAAAAA"` — ten A's. Stored naively as ASCII, that's 80 bits (10 bytes). But we could also encode it as `"10×A"` — just three symbols. We've compressed 10 symbols into 3. The decompressor knows how to reverse this: see `N×C`, output `C` repeated `N` times.

Now consider `"AJKQMZXBWR"` — ten random letters. There's no shorter description. Each character is unpredictable, so each one has to be transmitted in full.

The key insight: **compressibility is about predictability**. Data that has patterns, regularities, or redundancy can be compressed. Data that is already maximally random cannot be.

## Entropy: The Measure of Surprise

Claude Shannon formalized this intuition in 1948 with the concept of *entropy* — the average amount of information (surprise) in a message, measured in bits.

For a source that produces symbols with probabilities p₁, p₂, ..., pₙ, Shannon's entropy H is:

```
H = -Σ pᵢ × log₂(pᵢ)
```

Each term `-pᵢ × log₂(pᵢ)` is the contribution of symbol i to the total entropy: the probability of seeing it, times how surprising it is when we do.

### Worked Example: A Biased Coin

Suppose we're encoding coin flips where heads comes up 90% of the time.

```
Entropy of a biased coin (p_heads = 0.9):

p_heads = 0.9   →  information content = -log₂(0.9)  ≈ 0.152 bits
p_tails = 0.1   →  information content = -log₂(0.1)  ≈ 3.322 bits

H = -(0.9 × 0.152) + -(0.1 × 3.322)
  = -0.137 + -0.332
  ≈ 0.469 bits per flip
```

A fair coin has entropy of exactly 1 bit per flip (you need a full bit to represent each outcome). Our biased coin only has 0.469 bits per flip — because heads is so predictable, it carries very little information.

If we had a million flips of this coin, a naive encoding would use 1 million bits. An optimal compressor could theoretically use only ~469,000 bits — a 53% reduction, without losing a single flip.

### Entropy of English Text

English has 26 letters (plus punctuation, spaces). If all were equally likely, entropy would be log₂(27) ≈ 4.75 bits per character, and we'd need 5 bits per character minimum.

But English letters aren't equally likely:

```
Letter frequencies in typical English text:

E: 12.7%  ████████████▌
T:  9.1%  █████████
A:  8.2%  ████████▏
O:  7.5%  ███████▌
I:  7.0%  ███████
N:  6.7%  ██████▋
S:  6.3%  ██████▎
H:  6.1%  ██████
R:  6.0%  ██████
...
Z:  0.07% ▏
```

Shannon estimated English text at roughly 1.0–1.5 bits per character of *actual* entropy when accounting for context and word structure. ASCII uses 8 bits per character. That's a theoretical 5–8× compression ratio sitting there waiting to be claimed — and modern text compressors do claim most of it.

## Shannon's Source Coding Theorem

Shannon proved two fundamental theorems about compression. The source coding theorem (his first theorem) states:

> **It is impossible to losslessly compress a message to fewer bits than its Shannon entropy, on average.**

"On average" is important. For any specific message, you might get lucky — a short message might compress well. But you cannot build a compressor that *always* compresses. In fact:

**The incompressibility theorem**: For every compression function, there exist inputs that make the output *larger* than the input. This is a counting argument: if you map N-bit strings to shorter strings, some N-bit strings must collide (pigeonhole principle), and your compressor must handle those by expanding them.

This is why zip files of already-compressed data (like .jpg or .mp4) don't get smaller — the data is already near its entropy limit. Attempting to compress it just adds overhead.

```
Entropy ceiling — no compressor can beat this:

  Bits per    ┌─────────────────────────────────────────┐
  symbol      │                                         │
       8 │────│─── ASCII (8 bits/char) ─────────────────│
         │    │                                         │
       5 │    │─── Uniform 26-letter alphabet ──────────│
         │    │                   ╲ room for compression │
     1-2 │    │─── English entropy (Shannon estimate) ──│
         │    │                                         │
       0 └────┴─────────────────────────────────────────┘
```

## Lossless vs. Lossy Compression

This is the fork in the road that divides the entire compression world:

**Lossless compression** guarantees that decompression produces the *exact* original data. Not approximately. Not "close enough." Bit-for-bit identical. This is required for:
- Text and source code (a single wrong bit can break a program)
- Executable files and archives
- Medical images and legal documents
- Any data where exact reconstruction is required

**Lossy compression** accepts some degradation in exchange for dramatically better compression ratios. It's used when:
- Human perception can't detect (or doesn't care about) the lost information
- A "good enough" reconstruction is acceptable
- The original data has more precision than needed

The key insight behind lossy compression: human perception is not a perfect measuring instrument. Our eyes are more sensitive to changes in brightness than color. Our ears can't hear certain frequencies above certain amplitudes. If we discard imperceptible detail, we're still delivering the full perceptual experience — at a fraction of the data.

```
Lossless round-trip:

Original     Compress    Compressed    Decompress    Reconstructed
  Data    ─────────────▶    Data    ─────────────▶    Data
IDENTICAL                                          (bit-for-bit)

Lossy round-trip:

Original     Encode      Compressed    Decode       Approximation
  Data    ─────────────▶    Data    ─────────────▶    of Data
                                                   (information lost)
```

### The Lossy Tradeoff

Lossy compression creates a tunable tradeoff between quality and size. JPEG, for instance, has a "quality" parameter from 0–100 that controls how aggressively to discard high-frequency image data. At quality 95, the difference from the original is invisible to most people. At quality 30, you can see blocky artifacts but the file is 10× smaller.

This isn't "cheating" — it's engineering. The goal isn't to preserve all the bits; the goal is to preserve all the *value*.

## Compression Ratio and Related Metrics

When comparing algorithms, we use several metrics:

**Compression ratio**: original size / compressed size. A ratio of 3:1 means the compressed file is one-third the size.

**Space savings**: `(1 - compressed/original) × 100%`. A 3:1 ratio means 66.7% space savings.

**Bits per symbol**: the average number of bits used per input symbol. For text, lower is better.

**Compression speed**: how fast the compressor runs, usually in MB/s.

**Decompression speed**: often more important than compression speed in practice (you compress once, decompress many times).

```
Example: compressing a 100MB text file

Algorithm    Comp.Size    Ratio    Comp.Speed    Decomp.Speed
─────────────────────────────────────────────────────────────
gzip -6       36 MB       2.8×      25 MB/s       200 MB/s
gzip -9       34 MB       2.9×       8 MB/s       200 MB/s
Brotli-6      30 MB       3.3×      15 MB/s       200 MB/s
Zstd-6        33 MB       3.0×     220 MB/s       700 MB/s
Zstd-19       28 MB       3.6×      12 MB/s       700 MB/s
xz -6         27 MB       3.7×       4 MB/s        50 MB/s
```

Zstd at level 6 compresses nearly as well as gzip at level 9, but nearly 30× faster. Decompression is 3.5× faster. This is why Zstd has largely replaced zlib in performance-sensitive applications. More on this in Chapter 5.

## The Fundamental Limits

Shannon's entropy gives us the *theoretical* lower bound. Real compressors can't always reach it because:

1. **Context model complexity**: To predict what comes next perfectly, you'd need an infinite-order model. Real compressors use finite-order models.

2. **Compressor overhead**: The compressed format must include metadata for the decompressor (code tables, dictionary entries, etc.). For small inputs, this overhead dominates.

3. **Data structure mismatches**: A compressor optimized for text will underperform on binary data, and vice versa.

4. **Finite block sizes**: Compression improves with more context. A compressor working on a 4KB block can't exploit patterns across a 100MB file.

The gap between theoretical entropy and achievable compression is where most of the interesting engineering happens. The chapters ahead are about how different algorithms close that gap for different types of data.

## Summary

- **Entropy** is the theoretical minimum size of any lossless encoding of a data source.
- **Shannon's source coding theorem** proves no compressor can beat entropy on average.
- **Lossless compression** preserves all information; **lossy compression** discards perceptually unimportant information for better ratios.
- Compressibility is about predictability: regular, redundant data compresses; truly random data does not.
- All compression involves modeling the data. Better models → better compression, usually at higher computational cost.

In the next chapter, we'll look at Huffman coding — the first practical algorithm for approaching the entropy limit, and a component that still ships in almost every file format you've ever used.
