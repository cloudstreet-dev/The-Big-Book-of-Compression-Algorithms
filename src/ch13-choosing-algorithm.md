# Chapter 13: Choosing the Right Algorithm

After twelve chapters of algorithms, the practical question remains: when you sit down with a compression problem, how do you pick? The answer depends on your constraints, and compression problems have many possible constraints: speed, ratio, memory, compatibility, streaming, random access, data type, and more.

This chapter offers a structured decision framework, a set of guiding principles, and a reference matrix you can return to.

## The Four Dimensions

Every compression decision lives in a four-dimensional space:

```
Compression decision dimensions:

  ┌─────────────────────────────────────────────────────────┐
  │  1. SPEED                                               │
  │     Compression speed: how fast can you compress?       │
  │     Decompression speed: how fast must you decompress?  │
  │     Are they the same process? (usually not)            │
  │                                                         │
  │  2. RATIO                                               │
  │     How important is file size?                         │
  │     Is there a hard size budget?                        │
  │     What's the cost of additional bytes? (bandwidth $)  │
  │                                                         │
  │  3. COMPATIBILITY                                       │
  │     Who reads the compressed data?                      │
  │     Can you control both compressor and decompressor?   │
  │     Is this a public format or internal?                │
  │                                                         │
  │  4. DATA CHARACTERISTICS                                │
  │     Text? Binary? Images? Video? Sensor data?           │
  │     Already compressed?                                 │
  │     Regular structure? Random?                          │
  └─────────────────────────────────────────────────────────┘
```

## The Decision Tree

```
START HERE: What kind of data?
│
├── MEDIA (images, audio, video)
│   │
│   ├── Images
│   │   ├── Must be lossless → PNG (photos: use WebP lossless)
│   │   ├── Photographs, quality matters → AVIF (WebP lossy fallback)
│   │   ├── Photographs, compatibility → JPEG quality 80-90
│   │   ├── Icons, UI, transparency → PNG or WebP lossless
│   │   └── Already compressed → don't re-compress
│   │
│   ├── Audio
│   │   ├── Lossless archival → FLAC
│   │   ├── Distribution/streaming → Opus (or AAC-LC)
│   │   ├── Voice/communications → Opus
│   │   └── Legacy compatibility → MP3 VBR V2
│   │
│   └── Video
│       ├── New production/streaming → AV1 (H.265 fallback)
│       ├── Broad compatibility required → H.264
│       ├── Archival/lossless → FFV1 or uncompressed
│       └── Screen recording → VP9 or AV1 (high-ratio intra)
│
├── GENERAL PURPOSE (files, archives, backups)
│   │
│   ├── Need maximum compression, time doesn't matter
│   │   └── 7-Zip / xz (LZMA2) with solid archive
│   │
│   ├── Good compression, reasonable speed
│   │   └── Zstd level 6-9, or Brotli level 6 for web content
│   │
│   ├── Fast compression (logs, telemetry, backup streams)
│   │   └── Zstd level 1-3
│   │
│   ├── Legacy tools / universal compatibility
│   │   └── gzip level 6
│   │
│   └── Already compressed content (JPEG, MP4, ZIP, etc.)
│       └── Store, don't compress
│
├── DATABASE / IN-MEMORY
│   │
│   ├── Fastest possible decompression (cache, OLTP)
│   │   └── LZ4 or Snappy
│   │
│   ├── Good compression + fast decompress (OLAP, analytics)
│   │   └── Zstd level 3 + columnar encodings (Parquet)
│   │
│   ├── Maximum compression, read-heavy (cold storage)
│   │   └── GZIP or Zstd level 9+ in Parquet
│   │
│   └── Time-series / sorted numerics
│       └── Delta + bit-packing + Zstd (columnar format)
│
└── NETWORK / HTTP
    │
    ├── Static web assets (compress once, serve many)
    │   └── Brotli level 9-11 (+ gzip fallback)
    │
    ├── Dynamic API responses
    │   └── Zstd level 3 (or gzip level 6 for compatibility)
    │
    ├── Internal microservice traffic
    │   └── Zstd level 1 (if network is bottleneck), else none
    │
    └── Already-compressed responses (images, video)
        └── No Content-Encoding (pass through as-is)
```

## Guiding Principles

### 1. Measure on Your Actual Data

The most important advice in this chapter. Every benchmark in this book uses synthetic or corpus data. Your data is different. Run actual measurements.

```bash
# Quick benchmark: compare Zstd levels on your data
for level in 1 3 6 9 12 19; do
  echo -n "Level $level: "
  time zstd -$level -q -f your_data.bin -o /tmp/test.zst
  ls -lh /tmp/test.zst | awk '{print $5}'
done

# Compare decompression speeds
for f in your_data.gz your_data.br your_data.zst; do
  echo -n "$f: "
  time zcat $f > /dev/null 2>&1  # adjust command per format
done

# The lzbench tool benchmarks many algorithms at once:
lzbench -t16,16 your_data.bin
```

**lzbench** (https://github.com/inikep/lzbench) is the standard tool for this — it benchmarks 70+ compression algorithms and levels on any file you provide.

### 2. Compress Once, Decompress Many

The compression/decompression speed asymmetry matters enormously for economics. If a file will be downloaded 100,000 times:

```
Economic calculation for a 100MB file downloaded 100K times:

Option A: gzip-9 (slow compress, good ratio)
  Compress time: 200 seconds (one-time cost)
  Compressed size: 30MB → save 70MB per download
  Total bandwidth saved: 70MB × 100K = 7TB
  At $0.09/GB: $630 saved

Option B: gzip-6 (faster compress, slightly worse ratio)
  Compress time: 40 seconds (one-time cost)
  Compressed size: 32MB → save 68MB per download
  Total bandwidth saved: 68MB × 100K = 6.8TB
  At $0.09/GB: $612 saved

The 2MB per download difference is worth it for popular content.
For rarely-accessed files: faster compression is better economics.
```

### 3. Know When Compatibility Is Non-Negotiable

If you control both the writer and the reader, you can use any format you want. If the reader is a third party (a browser, a legacy system, a standards-mandated format), you're constrained.

```
Format compatibility grid (a reminder):

Format    gzip    br     zstd   lzma   zlib
──────────────────────────────────────────────
curl        ✓      ✓      ✓      —      —
Chrome      ✓      ✓      ✓      —      —
Safari      ✓      ✓      ✓      —      —
Python      ✓      ✓      ✓      ✓      ✓
Java (JDK)  ✓      —      —      —      ✓
Node.js     ✓      ✓      ✓      —      ✓
Most Unix   ✓      —      ✓      ✓      —
Windows     ✓      —      —      —      —
```

For files that must be openable by non-technical users without installing software: ZIP or GZIP. For everything else: Zstd.

### 4. Pre-Compress When You Can

Pre-compressing at high levels (and storing the result) always beats on-the-fly compression:
- Better ratio (more CPU time available)
- No latency penalty when serving
- Consistent performance under load

Static site generators, CDN pipelines, and build systems should always pre-compress at the highest practical level.

### 5. Don't Compress Already-Compressed Data

This deserves repeating because it's a common mistake. Adding a gzip step after a JPEG, MP4, or ZIP step is pure waste. The result will be approximately the same size, and you've wasted CPU.

Signs you're doing this wrong:
- Your archive contains `.jpg`, `.mp4`, or `.pdf` files and they compress to ~100% of their original size
- Your gzip ratio on "all files" is 1.01:1

### 6. Test Extreme Cases

Whatever algorithm you pick, test what happens when it gets input that's worst-case for that algorithm:
- Zstd on already-compressed data (should be ~1.0 ratio, not expand)
- RLE on random data (should not expand catastrophically)
- LZMA on a 10GB file (memory usage OK?)
- LZ4 on a 1-byte file (what's the overhead?)

## Algorithm Reference Matrix

```
Algorithm      Best for                Ratio    Speed(c)  Speed(d)  Memory
────────────────────────────────────────────────────────────────────────────
PLAIN/None     Random, already cmp     1.0×     ∞         ∞         0
RLE            Highly repetitive       10-100×  Very fast Very fast  Tiny
Delta          Time-series, sorted     2-20×    Very fast Very fast  Tiny
BWT+MTF        Text (bzip2)            3-4×     Slow      Slow       8MB
LZ4            In-memory, database     2-3×     700 MB/s  4000 MB/s  KB
Snappy         Database (Hadoop)       1.7-2×   500 MB/s  1800 MB/s  KB
Zstd-1         Fast general purpose    3-4×     500 MB/s  1200 MB/s  1MB
Zstd-3         General purpose         3-5×     350 MB/s  1200 MB/s  4MB
Zstd-9         High ratio + speed      4-6×     50 MB/s   1200 MB/s  16MB
Zstd-19        Near-max lossless       4-7×     8 MB/s    1200 MB/s  64MB
gzip-6         Universal compat        2.5-4×   25 MB/s   200 MB/s   256KB
Brotli-6       Web content             3-5×     15 MB/s   200 MB/s   8MB
Brotli-11      Pre-compressed web      3.5-5.5× 1 MB/s    200 MB/s   8MB
xz-6 (LZMA)   Archival, distrib.      4-8×     4 MB/s    50 MB/s    97MB
xz-9 (LZMA)   Maximum lossless        5-9×     1 MB/s    50 MB/s    683MB
PNG            Lossless images         1.5-3×   Medium    Fast       Medium
JPEG-90        Photographs             5-10×    Fast      Fast       Tiny
WebP-lossy     Web photographs         8-15×    Medium    Fast       Tiny
AVIF           Modern web images       15-30×   Very slow Fast       Medium
FLAC           Lossless audio          ~2×      Medium    Fast       Small
MP3-192        Lossy audio             7×       Medium    Fast       Small
Opus-128       Lossy audio (modern)    10×      Fast      Fast       Small
H.264          Video (universal)       100-200× Slow      Fast       Medium
H.265          Video (high eff.)       150-300× Very slow Fast       Medium
AV1            Video (open, modern)    200-400× Very slow Fast       Large
```

*Speed values are order-of-magnitude estimates on commodity hardware; actual speeds vary significantly by data type, CPU, and implementation.*

## The Compression Ratio vs. Speed Tradeoff Curve

One of the most useful mental models: every algorithm lives somewhere on a ratio-vs-speed tradeoff curve. Faster algorithms sacrifice ratio; better-ratio algorithms sacrifice speed.

```
Compression ratio vs. speed landscape:

  Ratio
   9× │                           ·xz-9
      │                      ·xz-6
   7× │                ·Zstd-19
      │            ·Brotli-11
   5× │       ·Brotli-6  ·Zstd-9
      │    ·gzip-6
   3× │  ·Zstd-3         (Modern sweet spot:
      │ ·Zstd-1           ·Zstd-3)
   2× │·Snappy
      │·LZ4
   1× │·None
      └────────────────────────────────────────── Speed
      0.1 MB/s    10 MB/s    100 MB/s   1000 MB/s
      (xz-9)      (Brotli)   (gzip)     (LZ4)

The "efficient frontier": no algorithm is simultaneously better
at ratio AND speed than the frontier algorithms.
Choosing an algorithm is choosing where on this curve you live.
```

Zstd's genius is that it offers points along this curve with a single tool: `-1` for LZ4-comparable speed, `-19` for xz-comparable ratio, and everything in between with the same binary and API.

## A Worked Example: Designing a Log Storage System

Problem: Store application logs. 50GB/day written, kept for 90 days (4.5TB). Queries: recent logs accessed frequently, old logs accessed rarely. Must compress to keep storage costs under $50/month.

**Step 1: Characterize the data**

Logs are structured text (JSON or key=value), with high repetition of field names and common values. Very compressible.

**Step 2: Define the constraints**

- Write speed: must handle ~600KB/s sustained write rate
- Read speed: recent logs must be queryable with <100ms latency
- Cost: must fit in ~5TB storage (4.5TB uncompressed → compress to < 1.5TB)
- Compatibility: in-house tools, no compatibility constraint

**Step 3: Tiered approach**

```
Log storage tiered compression:

Hot tier (< 24 hours): Zstd level 1
  - 600KB/s write → easily within Zstd-1 speed (500+ MB/s)
  - Ratio: ~4× → 12.5GB/day stored
  - Fast decompression for recent queries

Warm tier (1-7 days): Zstd level 6
  - Recompressed during off-peak (batch job)
  - Better ratio: ~5× → 10GB/day stored
  - Still fast decompress for query latency

Cold tier (> 7 days): Zstd level 19
  - Recompressed during off-peak weekly
  - Best ratio: ~6× → 8.3GB/day stored
  - Slower decompress acceptable for historical queries

Total storage:
  Day 1:    12.5GB (hot)
  Days 2-7:  10GB/day × 6 = 60GB (warm)
  Days 8-90: 8.3GB/day × 83 = 689GB (cold)
  Total: ~762GB for 90 days (vs 4.5TB uncompressed = 83% savings)

Cost at $0.02/GB/month: ~$15/month. ✓
```

**Step 4: Validate on real data**

Run `lzbench` on a sample of actual log files. Adjust tier boundaries based on actual ratios and speeds.

## Summary

- The four dimensions of every compression decision: speed, ratio, compatibility, and data characteristics.
- **Measure on your actual data.** Benchmarks on synthetic data are guides, not guarantees.
- **Compress once, decompress many**: justify expensive compression for frequently-accessed data.
- **Pre-compress when possible**: build-time compression beats run-time compression in almost every case.
- **Never compress already-compressed formats**: waste of CPU, rarely any size savings.
- Zstd is the modern general-purpose default: one tool, configurable from LZ4-speed to xz-ratio.
- Brotli for static web assets, gzip for universal compatibility, LZ4/Snappy for in-memory.
- LZMA/7-Zip when maximum compression ratio is the only goal and time doesn't matter.
- Tiered compression (different algorithms for hot/warm/cold data) is the practical answer for large-scale systems.

The next (and final) chapter is something different: a look at the argument that large language models are, at their core, a very sophisticated form of lossy compression.
