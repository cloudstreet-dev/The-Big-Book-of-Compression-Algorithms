# Introduction

> "The fundamental problem of communication is that of reproducing at one point either exactly or approximately a message selected at another point."
>
> — Claude Shannon, *A Mathematical Theory of Communication*, 1948

Compression is everywhere and mostly invisible. When you stream a video, your browser is decoding a stream that was compressed with H.264 or AV1. When you send a file over Slack, it's gzip'd in transit. When you store a photograph, the JPEG algorithm made a hundred small decisions about which visual details your eye probably won't notice. When a Parquet file loads in ten milliseconds instead of ten seconds, it's because the columnar layout lets LZ4 skip over 90% of the bytes.

Most developers interact with compression through flags and library calls: `--level 9`, `Content-Encoding: gzip`, `COMPRESS=zstd`. This book is for the developer who wants to understand what's happening underneath those flags — not just *which* algorithm to reach for, but *why* it works, *where* it breaks down, and *what* it's actually trading off.

## What This Book Covers

We start at the foundation: information theory. Shannon's entropy tells us how much compression is theoretically possible for any given data. Everything else in this book is the story of how humanity has gotten closer and closer to that limit, in different contexts, with different constraints.

From there, we work through the major algorithm families:

- **Statistical coders** (Huffman, arithmetic coding) — assign shorter codes to more frequent symbols
- **Dictionary methods** (LZ77, LZ78, LZW, Deflate, Brotli, Zstandard) — replace repeated patterns with references
- **Transform coders** (JPEG's DCT, audio's MDCT) — convert data to a domain where redundancy is easier to remove
- **Predictive coders** (FLAC, video inter-frame) — model what comes next and encode only the surprise

We then look at how these primitives combine in real systems: zip files, PNG, JPEG, MP3, H.264, Parquet, HTTP compression. Each domain has its own constraints and exploits different structure in the data.

We close with a chapter that's half-serious, half-provocative: the argument that large language models are, at a deep level, just very expensive lossy compressors. It's a useful frame for understanding what LLMs are good at, what they hallucinate, and why.

## How to Read This Book

Each chapter is designed to stand largely on its own after the first two foundational chapters. If you're already comfortable with entropy and information theory, you can skip ahead. If you're primarily interested in image compression, jump to Chapter 8 — but do read Chapters 1–4 first for the vocabulary.

Code examples are primarily in Python (for readability) and C (for the low-level parts that matter). They're illustrative, not production-ready. Where a real implementation differs significantly from the illustrative one, we'll say so.

ASCII diagrams appear throughout because compression algorithms are fundamentally about data structures and data transformations — and those are often clearest as pictures.

## A Note on Benchmarks

Compression benchmarks are notoriously sensitive to the data being compressed. A compressor that wins on text will lose on binary. A compressor that wins on small files will lose on large ones. Whenever we cite numbers, we'll note the context. When in doubt: measure on your actual data.

Let's begin.
