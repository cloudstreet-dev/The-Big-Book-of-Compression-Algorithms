# Further Reading

## Foundational Papers

**Shannon, C.E. (1948)**
*A Mathematical Theory of Communication*
Bell System Technical Journal, 27(3), 379–423.
The paper that started everything. Defines entropy, channel capacity, and the source coding theorem. Still readable and worth it.

**Huffman, D.A. (1952)**
*A Method for the Construction of Minimum-Redundancy Codes*
Proceedings of the IRE, 40(9), 1098–1101.
The original paper. Three pages. Elegant.

**Ziv, J. & Lempel, A. (1977)**
*A Universal Algorithm for Sequential Data Compression*
IEEE Transactions on Information Theory, 23(3), 337–343.
LZ77. The sliding window.

**Ziv, J. & Lempel, A. (1978)**
*Compression of Individual Sequences via Variable-Rate Coding*
IEEE Transactions on Information Theory, 24(5), 530–536.
LZ78. The explicit dictionary.

**Burrows, M. & Wheeler, D.J. (1994)**
*A Block-sorting Lossless Data Compression Algorithm*
Digital SRC Research Report 124.
The BWT. Makes bzip2 work. Clever sorting trick.

## Books

**Sayood, K. (2017)**
*Introduction to Data Compression*, 5th edition. Morgan Kaufmann.
The textbook. Comprehensive, mathematically rigorous. Good for going deep on entropy coding.

**Salomon, D. & Motta, G. (2010)**
*Handbook of Data Compression*, 5th edition. Springer.
Reference encyclopedia. Almost everything is in here somewhere.

**Blelloch, G. (2001)**
*Introduction to Data Compression*
Carnegie Mellon University lecture notes.
Free, concise, excellent. https://www.cs.cmu.edu/~guyb/realworld/compression.pdf

**Nelson, M. & Gailly, J.L. (1995)**
*The Data Compression Book*, 2nd edition. M&T Books.
Older but has detailed DEFLATE and LZW implementation walkthroughs.

## Modern Algorithm Papers

**Deutsch, P. (1996)**
*DEFLATE Compressed Data Format Specification* (RFC 1951).
The standard. Short, precise, implementable from this document alone.

**Collet, Y. & Kucherawy, M. (2021)**
*Zstandard Compression and the 'application/zstd' Media Type* (RFC 8878).
Zstandard specification with good explanation of design decisions.

**Alakuijala, J. et al. (2016)**
*Brotli: A General-Purpose Data Compressor*
Google open-source documentation and original paper.

**Duda, J. (2009, 2013)**
*Asymmetric Numeral Systems: Entropy Coding Combining Speed of Huffman Coding with Compression Rate of Arithmetic Coding*
arXiv:1311.2540. The ANS paper. Dense but important.

## Image and Media

**Wallace, G.K. (1991)**
*The JPEG Still Picture Compression Standard*
IEEE Transactions on Consumer Electronics.
How JPEG actually works, from the people who designed it.

**Richardson, I. (2003)**
*H.264 and MPEG-4 Video Compression: Video Coding for Next-Generation Multimedia*
Wiley. The most readable treatment of H.264 internals.

**Alliance for Open Media**
*AV1 Bitstream & Decoding Process Specification*
https://aomediacodec.github.io/av1-spec/
The full AV1 spec. Long.

## Tools and Benchmarks

**lzbench** — Benchmark 70+ compressors on your own data
https://github.com/inikep/lzbench

**Squash compression benchmark** — Web-based benchmark with many algorithms
https://quixdb.github.io/squash-benchmark/

**Silesia corpus** — Standard benchmark dataset (various file types)
https://sun.aei.polsl.pl/~sdeor/index.php?page=silesia

**Canterbury corpus** — Classic benchmark for text compression
https://corpus.canterbury.ac.nz/

**Hutter Prize** — Compressing 100MB of Wikipedia
http://prize.hutter1.net/

## LLMs and Compression

**Hutter, M. (2006)**
*Universal Algorithmic Intelligence: A Mathematical Top-Down Approach*
Springer. The theoretical framework connecting compression and intelligence.

**Deletang, G. et al. (2023)**
*Language Modeling Is Compression*
arXiv:2309.10668.
The DeepMind paper that made this argument rigorous: LLMs as compressors and the connection to intelligence.

**Huang, C. et al. (2022)**
*In-context Retrieval-Augmented Language Models*
arXiv:2302.00083.
One of the early RAG papers, addressing the "what gets lost" problem.

## Implementation References

**zlib source code**
https://github.com/madler/zlib
Mark Adler's reference implementation. Readable, annotated, definitive.

**zstd source code**
https://github.com/facebook/zstd
Facebook's implementation. Fast, well-commented, with extensive documentation.

**The reference bzip2 implementation**
https://sourceware.org/bzip2/
Small, clear C code. Good way to understand BWT in practice.

**FFmpeg** — The reference for audio and video codecs
https://ffmpeg.org/
Contains implementations of nearly every audio/video codec. Source is large but well-organized.

**libvpx / libaom** — VP9 and AV1 reference encoders/decoders
https://chromium.googlesource.com/webm/libvpx
https://aomedia.googlesource.com/aom/

## Going Further

**Matt Mahoney's data compression FAQ**
http://mattmahoney.net/dc/dce.html
Matt Mahoney runs the Hutter Prize and has an encyclopedic knowledge of data compression. His FAQ is one of the most detailed resources available.

**Fabian Giesen's blog**
https://fgiesen.wordpress.com/
Deep technical posts on entropy coding, ANS, and low-level compression implementation. His ANS posts are the best English-language explanation of the algorithm.

**Charles Bloom's blog**
https://cbloomrants.blogspot.com/
Compression engineering from someone who has built production codecs (Oodle, used in many game engines). Practical, opinionated, excellent.
