# Summary

[Introduction](./introduction.md)

---

# Part I — Foundations

- [What is Compression?](./ch01-what-is-compression.md)
  - Entropy and Information Theory
  - Shannon's Limits
  - Lossless vs. Lossy
- [Huffman Coding](./ch02-huffman-coding.md)
  - Building the Frequency Tree
  - Variable-Length Codes
  - Canonical Huffman
- [Arithmetic Coding](./ch03-arithmetic-coding.md)
  - Interval Subdivision
  - Probability Models
  - Adaptive Arithmetic Coding

---

# Part II — Dictionary and String Methods

- [Dictionary Methods: LZ77, LZ78, LZW](./ch04-dictionary-methods.md)
  - Sliding Window Compression
  - LZ78 and the Static Dictionary
  - LZW and the GIF Format
  - How ZIP Actually Works
- [Modern Lossless: Deflate, Brotli, Zstandard](./ch05-modern-lossless.md)
  - Deflate: LZ77 + Huffman
  - Brotli's Static Dictionary
  - Zstandard and Tunable Tradeoffs
- [LZMA and 7-Zip](./ch06-lzma-7zip.md)
  - Markov Chain Models
  - Range Encoding
  - The Cost of Maximum Compression

---

# Part III — Specialized and Simple Methods

- [Run-Length Encoding and Simple Methods](./ch07-run-length-simple.md)
  - Classic RLE
  - Delta Encoding
  - Burrows-Wheeler Transform
  - Move-to-Front
- [Image Compression](./ch08-image-compression.md)
  - PNG: Filters and Zlib
  - JPEG: DCT and Quantization
  - WebP: VP8 Intra-Frame
  - AVIF: The AV1 Intra-Frame Story
- [Audio Compression](./ch09-audio-compression.md)
  - FLAC: Lossless Prediction
  - MP3: Psychoacoustics and MDCT
  - AAC and Opus

---

# Part IV — Systems and Scale

- [Video Compression](./ch10-video-compression.md)
  - Spatial vs. Temporal Redundancy
  - Motion Estimation and Compensation
  - H.264, H.265, and AV1
- [Database and Columnar Compression](./ch11-database-columnar.md)
  - Row vs. Columnar Storage
  - Snappy and LZ4: Speed Above All
  - Dictionary, RLE, and Delta in Parquet
- [Network Compression](./ch12-network-compression.md)
  - HTTP Compression: gzip and Brotli
  - HPACK: Header Compression in HTTP/2
  - When Not to Compress

---

# Part V — Making Decisions

- [Choosing the Right Algorithm](./ch13-choosing-algorithm.md)
  - The Decision Framework
  - Speed vs. Ratio vs. Compatibility
  - Use-Case Reference Matrix

---

# Bonus

- [LLMs as Lossy Compressors](./ch14-llms-lossy-compressors.md)
  - Training as Compression
  - Inference as Decompression
  - What Gets Lost
  - Why This Matters

---

[Further Reading](./further-reading.md)
