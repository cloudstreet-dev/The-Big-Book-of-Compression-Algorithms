# Chapter 11: Database and Columnar Compression

Databases have a compression problem that's different from the problems we've seen so far. A video compressor can take minutes to encode a single frame. A database compressor runs on every write, every read, potentially millions of times per second. The constraint shifts from "best possible ratio" to "best ratio achievable in microseconds."

But databases have an advantage too: they *know something about their data*. A database knows that a column contains timestamps, or integers, or strings of bounded length. It knows the data type, the sort order, the cardinality. This domain knowledge enables compression techniques that generic algorithms can't match.

## Two Storage Models

Before compression, we need to understand the fundamental distinction between row-oriented and column-oriented storage, because it completely determines which compression techniques work.

```
The same data, stored two ways:

Logical table:
  user_id  country  age  plan
  ─────────────────────────────
  1001     US       34   pro
  1002     UK       27   free
  1003     US       41   pro
  1004     DE       33   free
  1005     US       29   pro

Row-oriented (traditional OLTP):
  [1001,US,34,pro][1002,UK,27,free][1003,US,41,pro][1004,DE,33,free]...

Column-oriented (OLAP / analytics):
  user_id:  [1001][1002][1003][1004][1005]...
  country:  [US][UK][US][DE][US]...
  age:      [34][27][41][33][29]...
  plan:     [pro][free][pro][free][pro]...
```

### Why Columns Compress Better

When data is stored in columns, each column contains only one type of data — and real-world data has low *cardinality* (few distinct values) and high *correlation* within columns:

- All `country` values are strings from a 195-element set
- All `age` values are integers in [0, 150]
- All `plan` values are one of {free, pro, enterprise}
- Adjacent rows in a time-sorted table have similar timestamps

A generic compressor on row data sees: `1001`, `US`, `34`, `pro`, `1002`, `UK`, `27`, `free` — a heterogeneous mix. A columnar compressor on the `plan` column sees: `pro`, `free`, `pro`, `free`, `pro` — highly compressible.

## Encoding Schemes for Columnar Data

### Dictionary Encoding

Replace string values with integer codes. Store the code → string mapping once.

```
Dictionary encoding of the 'country' column:

Dictionary: {0: "US", 1: "UK", 2: "DE", 3: "FR", ...}

Raw strings:   "US", "UK", "US", "DE", "US", "US", "FR", "US"
Encoded:        0,    1,    0,    2,    0,    0,    3,    0

Savings:
  Raw: 8 × 2 bytes = 16 bytes (average country code is 2 chars)
  Dict: 8 × 1 byte + 4 × 2 bytes (dict) ≈ 16 bytes
  → No savings for small cardinality strings!

But for longer strings ("United States", "United Kingdom"):
  Raw: 8 × 13 bytes average = 104 bytes
  Dict: 8 × 1 byte + 195 × 13 bytes dict = 8 + 2535...

The real win: dict encoding enables faster comparisons and aggregations.
Filter "country = 'US'" becomes "code = 0" — integer comparison on
compact integer arrays. No string hashing or comparison needed.
```

Dictionary encoding is the foundation of most columnar formats. It enables:
- Compact representation when cardinality is low
- Fast filtering without decoding (compare integers to integer codes)
- Late materialization (operate on codes, decode only when outputting results)

### Run-Length Encoding (Columnar Style)

After sorting, columns often have long runs of identical values. RLE stores (value, count) pairs:

```
RLE on sorted 'plan' column:

Sorted data: free, free, free, ...(10000×)..., pro, pro, ...(5000×)..., enterprise, ...

RLE: [(free, 10000), (pro, 5000), (enterprise, 500)]

Query "count where plan = 'pro'": → read single RLE entry → 5000
No decompression needed!

This is "predicate pushdown into the encoding":
The compression format IS the index.
```

Parquet, ORC, and most columnar databases use RLE on sorted data extensively.

### Bit-Packing

If a column's values fit in fewer bits than the storage type, pack them tightly.

```
Bit-packing a column of values 0–15 (4 bits needed, stored in 8):

Naive storage (8 bits each):
  0x03 0x07 0x0B 0x0F 0x01 0x09 0x02 0x0E
  = 8 bytes for 8 values

Bit-packed (4 bits each):
  0x37 0xBF 0x19 0x2E
  = 4 bytes for 8 values (50% savings)

Vectorized unpacking:
  Modern CPUs can unpack bit-packed integers using SIMD instructions
  at rates exceeding 1 billion integers per second.
```

**PFOR (Patched Frame of Reference)** is a sophisticated variant: most values in a frame fit in B bits, with occasional outliers ("exceptions"). Store the base values in B bits, exceptions separately.

### Delta Encoding for Sorted Numerics

Sorted ID or timestamp columns benefit enormously from delta encoding:

```
User ID column (sorted, sequential):
  Raw:    10000001, 10000002, 10000003, ..., 10000000+N
  Delta:  10000001, 1, 1, 1, ..., 1

  After delta: bit-pack the 1s into 1 bit each.
  Savings: 32-bit integers → effectively 1 bit each.

Timestamp column (milliseconds, sorted):
  Raw:    1700000000000, 1700000000060, 1700000000120...
  Delta:  1700000000000, 60, 60, 60...

  After delta: values fit in 6 bits (60 < 64).
  Savings: 64-bit timestamps → 6 bits each.
```

## Parquet: Columnar Storage Done Right

Apache Parquet (2013) is the de facto standard for columnar data at rest in the Hadoop ecosystem. It's used by Spark, Hive, Athena, Redshift Spectrum, BigQuery (via external tables), and most modern data lake tools.

### Parquet File Structure

```
Parquet file layout:

  ┌──────────────────────────────────────────────┐
  │  Magic: PAR1 (4 bytes)                      │
  ├──────────────────────────────────────────────┤
  │  Row Group 1 (e.g., 128MB of row data)      │
  │  ┌─────────────────────────────────────────┐│
  │  │ Column Chunk: user_id                   ││
  │  │  Page 1 (Dictionary page, if any)       ││
  │  │  Page 2 (Data pages)                    ││
  │  │  Page 3 ...                             ││
  │  ├─────────────────────────────────────────┤│
  │  │ Column Chunk: country                   ││
  │  │  Dictionary page + data pages           ││
  │  ├─────────────────────────────────────────┤│
  │  │ Column Chunk: age                       ││
  │  │  Data pages (RLE/bit-packed)            ││
  │  └─────────────────────────────────────────┘│
  ├──────────────────────────────────────────────┤
  │  Row Group 2...                             │
  ├──────────────────────────────────────────────┤
  │  File Footer (Thrift-encoded metadata)      │
  │  - Schema (column names, types)             │
  │  - Row group offsets and sizes              │
  │  - Column statistics (min, max, null count) │
  │  - Encoding types used per column           │
  ├──────────────────────────────────────────────┤
  │  Footer length (4 bytes)                    │
  │  Magic: PAR1 (4 bytes)                      │
  └──────────────────────────────────────────────┘
```

### Parquet Encodings

Parquet supports multiple encoding schemes per column chunk, chosen by the writer:

```
Parquet encoding types:

PLAIN:              Raw values, no encoding. Fallback.
DICTIONARY:         Dict page + integer codes. Default for strings.
RLE_DICTIONARY:     RLE on top of dict codes. Sorted low-cardinality.
PLAIN_DICTIONARY:   Deprecated (same as RLE_DICTIONARY).
RLE:                Run-length encoding.
BIT_PACKED:         Bit-packing. Used for repetition/definition levels.
DELTA_BINARY_PACKED: Delta + bit-packing for sorted integers.
DELTA_LENGTH_BYTE_ARRAY: Delta coding of string lengths.
DELTA_BYTE_ARRAY:   Delta coding of byte arrays (prefix compression for sorted strings).
BYTE_STREAM_SPLIT:  Split float bytes across separate streams for better GZIP compression.
```

The most important: **DELTA_BINARY_PACKED** for timestamp columns. A column of 1 billion sorted timestamps takes roughly 8GB naive; with delta + bit-packing it can compress to under 1GB before any additional gzip/Snappy compression.

### Parquet + Compression Codec

After columnar encoding, Parquet applies a general-purpose compressor to each page:

```
Encoding pipeline for a string column:

  "US", "UK", "US", "US", "DE", ...
       │
       ▼ Dictionary encoding
  Dict: {0:"US", 1:"UK", 2:"DE"}
  Codes: 0, 1, 0, 0, 2, ...
       │
       ▼ RLE_DICTIONARY
  (run-length encode the codes)
  (0, 3), (1, 1), (0, 47), (2, 2), ...
       │
       ▼ Snappy / GZIP / Zstd / LZ4
  Generic compression on the encoded bytes
       │
       ▼ Parquet page bytes
```

Choosing the Parquet compression codec:

```
Parquet codec comparison:

Codec       Compress speed    Decompress    Ratio    Use case
───────────────────────────────────────────────────────────────
UNCOMPRESSED  —              Fastest       1.0×     Development/debug
SNAPPY      Very fast        Very fast     1.5×     General Hadoop
LZ4_RAW     Very fast        Fastest       1.5×     Performance-critical
GZIP        Slow             Fast          2.0×     Cold storage, S3
ZSTD        Fast             Very fast     2.0×     Modern default choice
BROTLI      Very slow        Fast          2.2×     Static web data
```

Zstd has become the recommended default for new Parquet files in 2023+: better compression than Snappy, much faster than GZIP, good decompression speed.

## LZ4: Speed Above All Else

LZ4 was designed by Yann Collet (who also wrote Zstandard) with a single goal: be the fastest possible LZ-family compressor. Not the best ratio — the fastest.

```
LZ4 design philosophy:

"The minimum information needed to decompress is:
  - A stream of literal bytes (copy these literally)
  - A stream of (offset, length) match references

Encode these as simply as possible.
No Huffman. No arithmetic coding. Just token headers."

LZ4 token format:
  ┌──────────────────────────────────────────────────┐
  │  1 byte token:                                   │
  │  [literal_len: 4 bits][match_len: 4 bits]        │
  │                                                  │
  │  If literal_len == 15: read more bytes (255+)    │
  │  Then: literal_len literal bytes                 │
  │  Then: 2-byte offset (little-endian)             │
  │  If match_len == 15: read more bytes (255+4)     │
  └──────────────────────────────────────────────────┘

No entropy coding = decompressor does almost nothing.
Modern CPUs can decompress LZ4 at 4+ GB/s.
```

LZ4 compresses roughly 2:1 on typical data — much less than Zstd or gzip. But its decompression speed is so high that using LZ4 to compress RAM can actually *increase* effective memory bandwidth:

```
LZ4 in-memory caching:

Without compression:
  Read 100MB from disk → store 100MB in RAM → serve 100MB
  Network → 100MB/s → Cache hit reads: 100MB/s

With LZ4 compression:
  Read 100MB from disk → compress to 50MB → store 50MB in RAM
  Network → 100MB/s → Cache hit reads: 50MB at 4GB/s decompression
  Effective cache size: 2× (same RAM, more data)
  Latency: slightly higher (decompress time ~12ms for 50MB)
```

This is why Redis, Memcached, and many in-memory databases offer LZ4 as an optional compression layer.

## Snappy: Google's In-Memory Compressor

Snappy (originally Zippy, by Google, 2011) is similar to LZ4 — designed for speed over ratio. It's the default codec in Hadoop, Cassandra, and Leveldb/RocksDB.

```
Snappy vs. LZ4:

Metric              Snappy       LZ4
─────────────────────────────────────
Compression speed   ~500 MB/s    ~700 MB/s
Decompression       ~1800 MB/s   ~4000 MB/s
Compression ratio   ~1.7×        ~2.1×
Framing format      Yes (spec)   Flexible
```

LZ4 has mostly superseded Snappy in new systems — it's faster and compresses better. Snappy is still dominant in existing deployments due to ecosystem inertia.

## Columnar Compression in Practice

### Reading Only What You Need

The biggest benefit of columnar storage isn't compression ratio — it's I/O efficiency:

```
Query: SELECT avg(age) FROM users WHERE country = 'US'

Row-oriented storage:
  Must read ALL bytes of ALL rows (user_id, country, age, plan, ...)
  to extract country and age for each row.
  For 10M rows, 4 columns, 8 bytes each: 320MB read.

Column-oriented storage:
  Read ONLY the 'country' column and 'age' column.
  For 10M rows, 2 columns, 4 bytes each: 80MB read.
  4× less I/O before even considering compression.

With compression:
  Country column (dictionary): ~5MB (dictionary codes are tiny)
  Age column (bit-packed): ~20MB (values 0-150 fit in 8 bits)
  Total: 25MB read — 13× less than row-oriented.
```

### Predicate Pushdown and Bloom Filters

Parquet stores column statistics (min, max, null count) and optional Bloom filter data in the row group footer. Query engines use this to skip entire row groups without reading the column data:

```
Row group pruning example:

Query: SELECT * FROM orders WHERE order_date = '2024-01-15'

Row group 1:  date range [2024-01-01, 2024-01-10] → SKIP
Row group 2:  date range [2024-01-11, 2024-01-20] → READ
Row group 3:  date range [2024-01-21, 2024-01-31] → SKIP
Row group 4:  date range [2024-02-01, 2024-02-15] → SKIP

Result: read 25% of the data, skip 75%.
Combined with columnar projection, potentially read 3-5% of bytes.
```

Bloom filters provide set membership tests ("is this order_id in this row group?") without false negatives, at the cost of ~1% false positives. Combined with statistics, they enable very high skip rates for point queries.

## RocksDB and LSM-Tree Compression

RocksDB (Facebook, 2012) is a key-value store used by Cassandra, MySQL (MyRocks), Kafka (for log compaction), and many others. It uses an LSM (Log-Structured Merge) tree, which has interesting compression characteristics.

```
RocksDB LSM structure:

  Level 0: Small SSTables (sorted string tables)
           Written by memtable flush, not sorted between files
  Level 1: Larger SSTables, sorted, non-overlapping
  Level 2: 10× larger, same rules
  ...

Compression per level (typical config):
  L0, L1: Snappy (fast, writes happen often, frequent compaction)
  L2+:    Zstd or ZLIB (better ratio, these levels change less often)

Data size at each level:
  L0: 256MB
  L1: 256MB
  L2: 2.5GB
  L3: 25GB
  L4+: most of the data

Better compression on lower levels = smaller footprint.
```

The tiered compression strategy makes sense: levels that are frequently compacted (rewritten) benefit from fast compression; levels that are rarely compacted benefit from better ratio.

## Summary

- Columnar storage puts values of the same type together, making them dramatically more compressible.
- **Dictionary encoding** replaces string values with integer codes; enables fast filtering without decoding.
- **RLE** on sorted columnar data is the most powerful technique — can skip entire operations.
- **Bit-packing** and **delta encoding** exploit bounded ranges and sorted numeric columns.
- **Parquet** stacks columnar encodings + general-purpose compression (GZIP, Snappy, Zstd) per column chunk.
- **LZ4** and **Snappy** prioritize decompression speed for in-memory and real-time use cases.
- Predicate pushdown and statistics-based row group pruning multiply the benefit of columnar formats beyond compression ratio alone.

Next: network compression — where we add the constraint that both ends of the wire must agree on the algorithm, and examine when not to compress at all.
