# The Big Book of Compression Algorithms

**Everything compresses differently.**

A survey of the major algorithms — Huffman to Zstandard, lossless to lossy, zip files to video codecs — covering mechanics, tradeoffs, and ideal contexts. Closes with a bonus section on why LLMs are, at their core, just really expensive lossy compressors.

## Read Online

The book is published at: **https://cloudstreet-dev.github.io/The-Big-Book-of-Compression-Algorithms/**

## What's Inside

### Part I — Foundations

| Chapter | Topic |
|---------|-------|
| [What is Compression?](src/ch01-what-is-compression.md) | Entropy, Shannon's theorem, lossless vs lossy |
| [Huffman Coding](src/ch02-huffman-coding.md) | Frequency trees, prefix codes, canonical Huffman |
| [Arithmetic Coding](src/ch03-arithmetic-coding.md) | Interval subdivision, range coding, ANS/FSE |

### Part II — Dictionary and String Methods

| Chapter | Topic |
|---------|-------|
| [LZ77, LZ78, LZW](src/ch04-dictionary-methods.md) | Sliding windows, growing dictionaries, how ZIP works |
| [Deflate, Brotli, Zstandard](src/ch05-modern-lossless.md) | Why Zstd won, tunable tradeoffs |
| [LZMA and 7-Zip](src/ch06-lzma-7zip.md) | Ultra-high compression, Markov models, range coding |

### Part III — Specialized and Simple Methods

| Chapter | Topic |
|---------|-------|
| [RLE and Simple Methods](src/ch07-run-length-simple.md) | RLE, delta encoding, BWT, bzip2 pipeline |
| [Image Compression](src/ch08-image-compression.md) | PNG, JPEG, WebP, AVIF — what you lose and when it matters |
| [Audio Compression](src/ch09-audio-compression.md) | FLAC, MP3, AAC, Opus — psychoacoustics and bitrate tradeoffs |

### Part IV — Systems and Scale

| Chapter | Topic |
|---------|-------|
| [Video Compression](src/ch10-video-compression.md) | H.264, H.265, AV1 — inter-frame prediction, why video is different |
| [Database and Columnar Compression](src/ch11-database-columnar.md) | Snappy, LZ4, Parquet, dictionary/RLE/delta encodings |
| [Network Compression](src/ch12-network-compression.md) | HTTP compression, HPACK, when to compress in transit vs at rest |

### Part V — Making Decisions

| Chapter | Topic |
|---------|-------|
| [Choosing the Right Algorithm](src/ch13-choosing-algorithm.md) | Decision framework, speed vs ratio vs compatibility |

### Bonus

| Chapter | Topic |
|---------|-------|
| [LLMs as Lossy Compressors](src/ch14-llms-lossy-compressors.md) | Training as compression, inference as decompression, hallucinations as artifacts |

## Building Locally

Requires [mdBook](https://rust-lang.github.io/mdBook/):

```bash
cargo install mdbook
mdbook build     # outputs to ./book/
mdbook serve     # serves at http://localhost:3000 with live reload
```

## About

Published by [CloudStreet](https://cloudstreet.dev). Written for developers who want to understand not just which algorithm to use, but *why* it works and *where* it breaks down.

Licensed under [CC BY 4.0](LICENSE).
