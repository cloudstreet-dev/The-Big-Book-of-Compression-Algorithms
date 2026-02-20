# Chapter 3: Arithmetic Coding

Huffman coding has a fundamental limitation: it assigns a whole number of bits to each symbol. A symbol with probability 0.5 gets 1 bit. A symbol with probability 0.9 gets 1 bit too — even though the Shannon optimal encoding would use only 0.152 bits. This waste accumulates.

*Arithmetic coding* breaks free from this constraint. Instead of assigning a fixed codeword to each symbol, it encodes an entire message as a single number — a fraction between 0 and 1. The length of that fraction (in bits) approaches the entropy of the message, regardless of individual symbol probabilities.

The price: arithmetic coding is more complex to implement and slightly slower. The gain: it can encode at very close to Shannon's theoretical limit, even for symbols with probabilities that don't map cleanly to powers of 1/2.

## The Core Idea: Interval Subdivision

Imagine the number line from 0 to 1. We'll encode a message by progressively narrowing down to a smaller and smaller interval.

**Setup**: Assign each symbol a subinterval of [0, 1) proportional to its probability.

For a two-symbol alphabet (A, B) where P(A) = 0.7 and P(B) = 0.3:

```
Initial interval [0, 1):

0.0                    0.7          1.0
 ├──────────────────────┤────────────┤
 │          A           │     B      │
 │       [0.0, 0.7)     │  [0.7,1.0) │
```

**Encoding**: For each symbol in the message, narrow the current interval to the subinterval corresponding to that symbol, scaled to fit within the current interval.

Let's encode the message "ABA":

```
Step 1: Encode 'A'
  Current interval: [0.0, 1.0)
  A's sub-interval:  [0.0, 0.7) of the current interval
  New interval: [0.0 + 0.0×1.0, 0.0 + 0.7×1.0) = [0.0, 0.7)

  0.0          0.49     0.7       1.0
   ├────────────┤─────────┤─────────┤
   │ 'A' region │  'B' region (old) │

Step 2: Encode 'B'
  Current interval: [0.0, 0.7)
  Width: 0.7
  B's sub-interval: [0.7, 1.0) of the current interval
  New low  = 0.0 + 0.7 × 0.7 = 0.49
  New high = 0.0 + 1.0 × 0.7 = 0.70
  New interval: [0.49, 0.70)

  0.0   0.49    0.553  0.70    1.0
   ├─────┤────────┤──────┤──────┤
         │   A    │  B   │
         │[0.49,0.553)[0.553,0.70)

Step 3: Encode 'A'
  Current interval: [0.49, 0.70)
  Width: 0.21
  A's sub-interval: [0.0, 0.7) of the current interval
  New low  = 0.49 + 0.0 × 0.21 = 0.490
  New high = 0.49 + 0.7 × 0.21 = 0.637
  New interval: [0.490, 0.637)

Final interval: [0.490, 0.637)
```

The output is any value in the final interval. We pick the value that can be represented with the fewest bits. For example, 0.5 is representable as binary `0.1` (1 bit). Unfortunately 0.5 < 0.490 — doesn't work.

Let's try 0.5625 = binary `0.1001` — that's within [0.490, 0.637). Output: `0.1001` → 4 bits.

Compare to Huffman: if we assigned A=0, B=10, message "ABA" would be `0 10 0` = 4 bits. Same here, but arithmetic coding generalizes much better for non-binary-power probabilities.

## The Update Formula

For an interval `[low, high)` and a symbol with cumulative probability range `[cdf_low, cdf_high)`:

```
new_low  = low + (high - low) × cdf_low
new_high = low + (high - low) × cdf_high
```

The `cdf` (cumulative distribution function) values come from the probability model. If we have symbols A (prob 0.7) and B (prob 0.3):

```
Symbol   Prob   CDF_low   CDF_high
  A      0.70     0.00      0.70
  B      0.30     0.70      1.00
```

## Decoding

Decoding is the reverse: given a value `v` in [0, 1), repeatedly determine which symbol's interval contains `v`, output that symbol, and rescale `v` into the new interval.

```
Decoding 0.5625 ("0.1001" in binary):

Step 1: Which symbol does 0.5625 fall in?
  A covers [0.0, 0.7) — yes, 0.5625 < 0.7
  Output: 'A'
  Rescale: v = (0.5625 - 0.0) / 0.7 = 0.80357...

Step 2: 0.80357 falls in which symbol's range?
  A covers [0.0, 0.7) — no, 0.80357 > 0.7
  B covers [0.7, 1.0) — yes
  Output: 'B'
  Rescale: v = (0.80357 - 0.7) / 0.3 = 0.34524...

Step 3: 0.34524 falls in:
  A covers [0.0, 0.7) — yes
  Output: 'A'
  (Stop when we've decoded the expected number of symbols)

Result: "ABA" ✓
```

## Implementation: Integer Arithmetic Coding

Working with floating-point numbers is numerically unstable for long messages (precision runs out quickly). Real implementations use fixed-precision integers and a *renormalization* trick to keep the interval values in a workable range.

The key insight: when the high and low values share a leading bit, that bit is "settled" and can be output. We then shift both values left by one bit to maintain precision.

```python
PRECISION = 32
FULL    = 1 << PRECISION           # 2^32
HALF    = FULL >> 1                # 2^31
QUARTER = FULL >> 2                # 2^30

def arithmetic_encode(symbols: list[int], model: list[int]) -> bytes:
    """
    Encode symbols using arithmetic coding.

    model: list of cumulative frequencies (CDF), length = alphabet_size + 1
           model[i] = cumulative count up to (not including) symbol i
           model[-1] = total count
    """
    low  = 0
    high = FULL
    pending_bits = 0  # bits to output after we resolve the "almost half" case
    output_bits = []

    def emit_bit(bit):
        output_bits.append(bit)
        # Emit any pending opposite bits
        for _ in range(pending_bits):
            output_bits.append(1 - bit)
        return 0  # reset pending count

    total = model[-1]

    for sym in symbols:
        range_size = high - low
        high = low + (range_size * model[sym + 1]) // total
        low  = low + (range_size * model[sym])     // total

        # Renormalize: emit settled bits
        while True:
            if high <= HALF:
                # Both in lower half: emit 0
                pending_bits = emit_bit(0)
            elif low >= HALF:
                # Both in upper half: emit 1
                low  -= HALF
                high -= HALF
                pending_bits = emit_bit(1)
            elif low >= QUARTER and high <= 3 * QUARTER:
                # Straddling the middle: defer
                low  -= QUARTER
                high -= QUARTER
                pending_bits += 1
            else:
                break

            low  <<= 1
            high <<= 1

    # Flush remaining bits
    pending_bits += 1
    if low < QUARTER:
        emit_bit(0)
    else:
        emit_bit(1)

    return output_bits

def build_model(text: str) -> list[int]:
    """Build a simple frequency model (cumulative counts)."""
    from collections import Counter
    freq = Counter(text)
    alphabet = sorted(freq.keys())
    # Map characters to indices
    char_to_idx = {c: i for i, c in enumerate(alphabet)}

    counts = [freq.get(c, 0) for c in alphabet]
    cdf = [0]
    for c in counts:
        cdf.append(cdf[-1] + c)

    return cdf, char_to_idx, alphabet
```

## Adaptive Arithmetic Coding

The version above requires knowing the full probability model upfront — which means two passes over the data, or buffering the entire message. *Adaptive arithmetic coding* solves this by updating the model after each symbol.

The decompressor can mirror the encoder's updates exactly, because it sees the same symbols in the same order. This makes adaptive arithmetic coding a powerful tool for streaming compression.

```
Adaptive model update (simplified):

Initialize: all symbols have count = 1 (Laplace smoothing)

After encoding/decoding each symbol s:
  freq[s] += 1

The probability model continuously refines itself to match
the actual distribution of the data.

   Time →
   ┌──────────────────────────────────────────────┐
   │ Early: uniform priors, poor compression       │
   │ After 100 symbols: model adapting             │
   │ After 1000 symbols: model closely matches     │
   │                     true distribution         │
   └──────────────────────────────────────────────┘
```

Real-world adaptive coders (like those in LZMA and context-mixing compressors) use much more sophisticated context models — they condition the probability of the current symbol on the last N symbols, the current bit position, and other features. This is the heart of state-of-the-art compression.

## Range Coding: The Practical Variant

Arithmetic coding as described works on the interval [0, 1) with arbitrary precision. *Range coding*, introduced by G. Nigel N. Martin in 1979, is an equivalent but practically cleaner implementation that works on integer ranges directly. It's faster and avoids some of the floating-point subtleties.

Range coding is what's used inside LZMA (7-zip, XZ), and is the standard in high-performance entropy coders.

The key difference: instead of maintaining [low, high) ⊂ [0, 1), maintain a range [low, low + range) where `range` is kept in a fixed-precision integer window by outputting bytes when it becomes too large.

```
Range coder state machine:

  range = 0xFFFFFFFF  (initial range, 32-bit)
  low   = 0

  For each symbol:
    1. Compute symbol's frequency band within current range
    2. Update low and range accordingly
    3. If range < BOTTOM_VALUE (e.g., 1<<24):
       - Output high byte of low
       - Shift low and range left by 8 bits (renormalize)
```

## Probability Models: The Real Differentiator

The arithmetic coder itself is just a mechanism for converting probabilities to bits. What separates good compressors from great ones is the *probability model* — how accurately it predicts the next symbol.

**Order-0 model**: Probability of each symbol is its frequency in the file. Simple, fast, works well for uniform text.

**Order-N model**: Probability of the next symbol conditioned on the last N symbols. An order-4 model for English text uses context like "tion" to predict that the next character is probably a space, punctuation, or another vowel.

**Prediction by Partial Matching (PPM)**: Uses the longest available context, falling back to shorter contexts when the long one hasn't been seen. PPM was state-of-the-art for general compression through the 1990s and still produces excellent compression ratios.

**Context Mixing (CM)**: Combines predictions from multiple models using mixing weights. PAQ, the reference-grade maximum-compression tool, uses this approach and achieves remarkable compression ratios at enormous computational cost.

```
Context model comparison on Canterbury Corpus (text):

Model          Bits/char    Decoder complexity
────────────────────────────────────────────────
Order-0           4.6             O(1)
Order-1           3.5             O(1)
Order-4           2.4           O(alphabet)
PPM-C             1.9           O(alphabet × context)
PAQ8              1.7           O(many models × context)
Shannon limit    ~1.3              -
```

## Where Arithmetic Coding Is Used

Arithmetic coding never became as ubiquitous as Huffman, partly because of patent issues in the 1990s (IBM and others held key patents), and partly because Huffman is simpler to implement and fast enough for most uses.

But arithmetic coding (specifically range coding) is central to:
- **LZMA/7-Zip**: Range coding for the entropy stage
- **HEVC/H.265 video**: CABAC (Context-Adaptive Binary Arithmetic Coding) for syntax element coding
- **JPEG 2000**: Arithmetic coding replaces Huffman from JPEG
- **WebM/VP9**: Optimized boolean arithmetic coder
- **Zstandard's ANS backend**: A modern alternative (see below)

## ANS: Asymmetric Numeral Systems

*Asymmetric Numeral Systems* (ANS), invented by Jarosław Duda in 2009 and published in 2014, is the modern successor to arithmetic coding for high-performance systems. It achieves the same near-optimal compression as arithmetic coding but is faster because it can be implemented without division operations.

ANS comes in two main flavors:
- **rANS** (range ANS): Processes symbols serially, similar to arithmetic coding
- **tANS** (tabled ANS): Uses precomputed tables for fast lookup-based coding

Zstandard, Facebook's widely-deployed compression library, uses tANS (called FSE — Finite State Entropy — in its implementation). It encodes at speeds comparable to Huffman while approaching arithmetic coding's compression ratio.

```
Rough comparison of entropy coder performance:

Coder          Speed (encode)    Compression ratio    Complexity
──────────────────────────────────────────────────────────────────
Huffman          Very fast           Good               Low
Arithmetic       Moderate           Excellent           High
Range coding     Moderate           Excellent           High
rANS             Fast               Excellent           Medium
tANS/FSE         Very fast          Excellent           Medium
```

ANS is why Zstandard can be both fast and compress well — it uses tANS where DEFLATE uses Huffman, and that's a meaningful part of why Zstd beats gzip at comparable speeds.

## Summary

- Arithmetic coding encodes an entire message as a single number in [0, 1).
- Each symbol narrows the interval proportionally to its probability.
- It can encode at arbitrarily close to Shannon's limit, including fractional bits.
- Adaptive variants update the model on-the-fly — no two-pass requirement.
- Range coding is the practical integer implementation used in real compressors.
- ANS (especially tANS/FSE) is the modern high-speed alternative, used in Zstandard.
- The probability *model* determines compression quality; the entropy coder just converts probabilities to bits efficiently.

In the next chapter, we shift from statistical coding to *dictionary methods* — a completely different approach that replaces repeated patterns with compact references rather than adjusting code lengths.
