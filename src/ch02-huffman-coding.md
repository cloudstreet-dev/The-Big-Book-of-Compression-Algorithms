# Chapter 2: Huffman Coding

In 1952, MIT student David Huffman was taking an information theory course and was given an assignment: prove which of two methods for constructing optimal prefix codes was better. He couldn't prove either was optimal — so he invented a third method that was demonstrably optimal, and published it. His professor, Robert Fano, had been working on the problem for years. Huffman's algorithm was simpler and better.

The core idea: **give shorter codes to more frequent symbols, longer codes to rarer ones.** This is the same intuition behind Morse code (E is `·`, Z is `−−··`). Huffman's contribution was an algorithm that produces *provably optimal* variable-length prefix codes.

## Prefix Codes and the No-Prefix Property

Before the algorithm, let's understand what we're building. A *prefix code* (also called a *prefix-free code*) is a set of codewords where no codeword is a prefix of another.

Why does this matter? Because without it, decoding is ambiguous:

```
Bad code (not prefix-free):

Symbol  Code
  A      0
  B      01
  C      10
  D      11

Receiving "010":
  - Could be A, B      (0, 10... wait, is it A then B starting with 0? or 01 then?)
  - Ambiguous!
```

A prefix-free code can be decoded greedily: read bits until you match a codeword, output the symbol, repeat. No backtracking needed.

```
Good code (prefix-free):

Symbol  Code
  A      00
  B      01
  C      10
  D      11

Receiving "010011":
  - 01 → B
  - 00 → A
  - 11 → D
  - Clear!
```

## Building the Huffman Tree

Huffman codes are built from a binary tree. Leaves are symbols; the path from root to leaf defines the codeword (left branch = 0, right branch = 1).

The algorithm:

1. Count the frequency of each symbol.
2. Create a leaf node for each symbol, with its frequency as its weight.
3. Insert all nodes into a min-priority queue.
4. While more than one node remains:
   a. Extract the two nodes with lowest weight.
   b. Create a new internal node with these two as children, weight = sum of children's weights.
   c. Insert the new node back into the queue.
5. The remaining node is the root.

### Step-by-Step Example

Input: `"ABRACADABRA"` (11 characters)

**Step 1: Count frequencies**

```
Symbol  Count  Probability
  A       5       0.455
  B       2       0.182
  R       2       0.182
  C       1       0.091
  D       1       0.091
```

**Step 2: Build initial priority queue (lowest weight first)**

```
[C:1] [D:1] [B:2] [R:2] [A:5]
```

**Step 3: Combine C and D**

```
Extract C:1 and D:1 → create internal node CD:2

         CD:2
        /    \
      C:1   D:1

Queue: [B:2] [R:2] [CD:2] [A:5]
```

**Step 4: Combine B and R**

```
Extract B:2 and R:2 → create internal node BR:4

         BR:4
        /    \
      B:2   R:2

Queue: [CD:2] [A:5] [BR:4]
  → sorted: [CD:2] [BR:4] [A:5]
```

**Step 5: Combine CD and BR**

```
Extract CD:2 and BR:4 → create internal node CDBR:6

              CDBR:6
             /       \
           CD:2     BR:4
          /    \   /    \
        C:1  D:1 B:2  R:2

Queue: [A:5] [CDBR:6]
```

**Step 6: Combine A and CDBR**

```
Extract A:5 and CDBR:6 → create root node ALL:11

                     ALL:11
                    /       \
                  A:5      CDBR:6
                           /       \
                         CD:2     BR:4
                        /    \   /    \
                      C:1  D:1 B:2  R:2
```

**Step 7: Read off codewords (left=0, right=1)**

```
Symbol  Path          Code   Length
  A     left           0       1
  C     right,left,left    100     3
  D     right,left,right   101     3
  B     right,right,left   110     3
  R     right,right,right  111     3
```

**Verify the compression:**

```
Original (fixed 3-bit codes, since 5 symbols needs 3 bits):
  11 chars × 3 bits = 33 bits

Huffman:
  A appears 5 times × 1 bit  =  5 bits
  B appears 2 times × 3 bits =  6 bits
  R appears 2 times × 3 bits =  6 bits
  C appears 1 time  × 3 bits =  3 bits
  D appears 1 time  × 3 bits =  3 bits
  Total = 23 bits

Savings: 33 - 23 = 10 bits (30% reduction)
```

How close are we to Shannon's entropy limit?

```
H = -(5/11)log₂(5/11) - 2×(2/11)log₂(2/11) - 2×(1/11)log₂(1/11)
  ≈ 0.455×1.14 + 2×0.182×2.46 + 2×0.091×3.46
  ≈ 0.518 + 0.895 + 0.630
  ≈ 2.043 bits per symbol

Entropy for 11 symbols: 11 × 2.043 ≈ 22.5 bits
Huffman achieved: 23 bits  ✓ (within 2.2% of theoretical minimum)
```

## Python Implementation

Here's a clean, readable implementation of Huffman coding:

```python
import heapq
from collections import Counter
from dataclasses import dataclass, field
from typing import Optional

@dataclass(order=True)
class HuffNode:
    weight: int
    symbol: Optional[str] = field(default=None, compare=False)
    left: Optional['HuffNode'] = field(default=None, compare=False)
    right: Optional['HuffNode'] = field(default=None, compare=False)

def build_huffman_tree(text: str) -> HuffNode:
    """Build a Huffman tree from input text."""
    freq = Counter(text)

    # Initialize priority queue with leaf nodes
    heap = [HuffNode(weight=count, symbol=char)
            for char, count in freq.items()]
    heapq.heapify(heap)

    while len(heap) > 1:
        left = heapq.heappop(heap)
        right = heapq.heappop(heap)

        internal = HuffNode(
            weight=left.weight + right.weight,
            left=left,
            right=right
        )
        heapq.heappush(heap, internal)

    return heap[0]

def build_codebook(root: HuffNode) -> dict[str, str]:
    """Extract symbol → bitstring mapping from tree."""
    codebook = {}

    def traverse(node: HuffNode, path: str):
        if node.symbol is not None:       # leaf node
            codebook[node.symbol] = path or '0'  # single-symbol edge case
        else:
            traverse(node.left, path + '0')
            traverse(node.right, path + '1')

    traverse(root, '')
    return codebook

def huffman_encode(text: str) -> tuple[str, dict]:
    """Encode text, return bitstring and codebook."""
    if not text:
        return '', {}

    tree = build_huffman_tree(text)
    codebook = build_codebook(tree)

    encoded = ''.join(codebook[char] for char in text)
    return encoded, codebook

def huffman_decode(bits: str, codebook: dict) -> str:
    """Decode bitstring using codebook."""
    # Invert codebook: bitstring → symbol
    reverse = {v: k for k, v in codebook.items()}

    result = []
    current = ''
    for bit in bits:
        current += bit
        if current in reverse:
            result.append(reverse[current])
            current = ''

    return ''.join(result)

# Demo
text = "ABRACADABRA"
encoded, codebook = huffman_encode(text)

print(f"Original: {len(text) * 8} bits (naive 8-bit ASCII)")
print(f"Encoded:  {len(encoded)} bits")
print(f"Codebook: {codebook}")
print(f"Decoded:  {huffman_decode(encoded, codebook)}")
```

Output:
```
Original: 88 bits (naive 8-bit ASCII)
Encoded:  23 bits
Codebook: {'A': '0', 'C': '100', 'D': '101', 'B': '110', 'R': '111'}
Decoded:  ABRACADABRA
```

## The Optimality Guarantee

Huffman coding is provably optimal among all prefix codes. The proof is an exchange argument: if you have an optimal code that doesn't satisfy Huffman's tree properties, you can always swap two codewords to make it "more Huffman" without increasing average code length.

Key properties of an optimal Huffman tree:
1. More frequent symbols have shorter (or equal) codes.
2. The two least frequent symbols have the same length.
3. The two least frequent symbols differ only in the last bit.

The algorithm constructs a tree satisfying all three properties by construction.

**One caveat**: Huffman is optimal among prefix codes, but prefix codes are not always optimal encoders. Arithmetic coding (Chapter 3) can beat Huffman by encoding fractional bits per symbol — relevant when probabilities aren't nice powers of 1/2.

## Canonical Huffman Codes

There's a problem with Huffman: the decompressor needs the codebook to decode. Sending the full tree with the compressed data adds overhead, and trees can be large.

*Canonical Huffman* solves this by defining a standard way to assign codewords given only the code lengths. The decompressor only needs to know the length of each symbol's codeword — not the codeword itself — to reconstruct the same canonical codebook.

The canonical assignment rule:
1. Sort symbols by code length, then alphabetically within same length.
2. Assign the numerically smallest codeword to the first symbol.
3. Each subsequent symbol at the same length gets the next integer.
4. When moving to a longer length, left-shift and continue.

```
From our ABRACADABRA example:

Sorted by length:
  A: length 1
  B: length 3
  C: length 3
  D: length 3
  R: length 3

Canonical assignment:
  A: 0          (0 in binary)
  B: 100        (shift 0 to 000, +1 → 001... wait, jump from length 1 to 3)
               (last code at length 1 was 0, shift left twice → 000, +1 → 001... no)

  Correct algorithm:
  Start code = 0, length = 1
    A (length 1): code = 0b0 = "0"

  Move to length 3: code = (0 + 1) << (3 - 1) = 1 << 2 = 4 = 0b100
    B (length 3): code = 4 = "100"
    C (length 3): code = 5 = "101"
    D (length 3): code = 6 = "110"
    R (length 3): code = 7 = "111"

Canonical codebook:
  A → "0"
  B → "100"
  C → "101"
  D → "110"
  R → "111"
```

The decompressor only needs to store: the list of code lengths `[1, 3, 3, 3, 3]` (one per symbol, in sorted order). From this alone, it can reconstruct the entire codebook. This is how DEFLATE (the algorithm inside gzip and PNG) stores its Huffman tables.

```python
def canonical_codebook(lengths: dict[str, int]) -> dict[str, str]:
    """Generate canonical Huffman codes from code lengths."""
    # Sort by length, then by symbol
    sorted_syms = sorted(lengths.items(), key=lambda x: (x[1], x[0]))

    codebook = {}
    code = 0
    prev_len = 0

    for symbol, length in sorted_syms:
        if length > prev_len:
            code <<= (length - prev_len)
        codebook[symbol] = format(code, f'0{length}b')
        code += 1
        prev_len = length

    return codebook
```

## Where Huffman Is Used Today

Huffman coding is not usually used alone — it's a building block inside larger formats:

- **DEFLATE** (gzip, zlib, PNG, zip): Two Huffman trees, one for literals/lengths, one for distances.
- **JPEG**: Each coefficient's magnitude is Huffman coded.
- **MP3**: Scale factors and Huffman tables for spectral data.
- **HTTP/2 HPACK**: A static Huffman table for header value compression.

Pure Huffman is rare in modern systems because it requires two passes (one to count frequencies, one to encode) and doesn't handle adaptive probability models well. But it remains foundational: almost every format that does entropy coding has Huffman somewhere inside it.

## Huffman's Limitations

**Two-pass requirement**: You need to know all frequencies before building the tree, which means reading the data twice or buffering it all. Adaptive Huffman (not covered here) addresses this but is more complex.

**Integer code lengths**: Each symbol gets a whole number of bits. If a symbol has probability 0.4, it ideally gets 1.32 bits — but Huffman must assign it 1 or 2. This integer rounding is where arithmetic coding gains its advantage.

**Block-based**: Huffman builds one tree for the whole block. If the data's statistics change mid-block (common in real files), the single tree is suboptimal for parts of the data.

Despite these limitations, Huffman coding's simplicity, efficiency, and optimality guarantee have kept it relevant for over 70 years. It's one of those rare algorithms that was essentially perfect from day one.

## Summary

- Huffman coding assigns shorter binary codewords to more frequent symbols.
- The algorithm builds a binary tree bottom-up using a min-priority queue.
- The resulting prefix-free code is provably optimal among all prefix codes.
- Canonical Huffman reduces decompressor overhead to just storing code lengths.
- Huffman is a building block inside DEFLATE, JPEG, MP3, and dozens of other formats.

Next, we'll look at arithmetic coding — which breaks the "integer bits" barrier and can approach Shannon's limit even more closely.
