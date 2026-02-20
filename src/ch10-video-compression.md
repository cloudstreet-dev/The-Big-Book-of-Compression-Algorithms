# Chapter 10: Video Compression

Video is where the stakes are highest and the algorithms are most complex. A single minute of uncompressed 1080p video at 30fps is about 6GB. Streaming services deliver that minute in roughly 50MB at good quality — a 120:1 compression ratio — in real time, to billions of devices.

Understanding video compression means understanding everything from the previous chapters, applied to a new dimension: time. Video has spatial redundancy (within a frame, like images) and temporal redundancy (between frames, since most of the scene doesn't move much between consecutive frames). The second kind is where the dramatic compression ratios come from.

## The Scale of the Problem

```
Uncompressed video data rates:

Resolution    FPS    Bit depth    Data rate
──────────────────────────────────────────────
720p         30      8-bit        777 Mbps
1080p        30      8-bit        1.5 Gbps
1080p        60      8-bit        3.0 Gbps
4K UHD       30      8-bit        5.7 Gbps
4K UHD       60      10-bit      17.6 Gbps
8K UHD       60      10-bit      71.6 Gbps

Target streaming bitrates (compressed):
1080p at good quality: 4–8 Mbps (H.264)
                       2–4 Mbps (H.265/HEVC)
                       1.5–3 Mbps (AV1)
4K at good quality:   15–25 Mbps (H.264, rarely used)
                       7–12 Mbps (H.265)
                       4–8 Mbps (AV1)
```

## Spatial Compression: Intra-Frame Coding

Each individual video frame can be compressed like an image — using DCT, quantization, and entropy coding. This is called *intra-frame* coding (I-frames in MPEG terminology).

Modern video codecs apply intra-prediction before the transform: they predict each block from its neighbors within the same frame, then transform and quantize the prediction error (residual).

```
Intra-prediction directions (H.264/H.265):

For each block, try predicting from:
  ┌─────────────────────────────────────────┐
  │  ←─ left ─→  ↑─ above ─↑  ↖diagonal   │
  │  ↙ down-left ↘ down-right              │
  │  DC (flat, average of neighbors)        │
  │  + many more in H.265 (33 directions)  │
  │  + even more in AV1 (56 directions)    │
  └─────────────────────────────────────────┘

Pick the prediction that leaves the smallest residual.
Encode: prediction mode + residual.
```

This gives modern codecs a significant advantage over JPEG even on still images — JPEG has no intra-prediction.

## Temporal Compression: Inter-Frame Coding

The big win. Between consecutive video frames, most pixels are the same or have just moved slightly. Instead of encoding each frame independently, encode only the *difference* from a previous frame.

```
Temporal redundancy visualization:

Frame N:      Frame N+1 (person moves slightly right):

┌──────────────────────┐   ┌──────────────────────┐
│░░░░░░░░░░░░░░░░░░░░░░│   │░░░░░░░░░░░░░░░░░░░░░░│
│░░░░░[PERSON]░░░░░░░░░│   │░░░░░░░[PERSON]░░░░░░░│
│░░░░░░░░░░░░░░░░░░░░░░│   │░░░░░░░░░░░░░░░░░░░░░░│
│░░░░░░░░░░░░░░░░░░░░░░│   │░░░░░░░░░░░░░░░░░░░░░░│
└──────────────────────┘   └──────────────────────┘

Background: identical (background compression is essentially free)
Person: same pixels, shifted 2 pixels right
        → encode as "motion vector (2,0), copy region"

The difference frame: almost entirely zero.
```

### Motion Estimation and Compensation

**Motion estimation**: Find where each block of the current frame came from in the reference frame.
**Motion compensation**: Use the found match to predict the current block.

The encoder tries to find, for each block, a motion vector `(dx, dy)` such that the block in the reference frame at `(x+dx, y+dy)` is as similar as possible to the current block.

```
Motion estimation search pattern:

For block at current position (cx, cy),
search reference frame within search range:

Reference frame search window:
  ┌─────────────────────────────┐
  │  . . . . . . . . . . . .   │  Each '.' is a candidate
  │  . . . . . . . . . . . .   │  position to check.
  │  . . . . ┌─────┐ . . . .   │
  │  . . . . │BLOCK│ . . . .   │  Find the candidate with
  │  . . . . └─────┘ . . . .   │  smallest difference (SAD,
  │  . . . . . . . . . . . .   │  Sum of Absolute Differences)
  │  . . . . . . . . . . . .   │
  └─────────────────────────────┘

Exhaustive search: O(search_range²) per block position → too slow
Fast search algorithms:
  Three-step search: O(log(search_range)) candidates
  Diamond search: starts large, zooms in
  Hierarchical (multi-resolution): search at coarse scale first
```

After motion compensation, the *residual* (actual block minus predicted block) is typically small — it can be transformed, quantized, and encoded like an intra-frame block, but with fewer bits needed because the values are close to zero.

### Sub-pixel Motion

Real motion isn't always aligned to integer pixel boundaries. A block might have moved 2.5 pixels to the right. Modern codecs support sub-pixel motion vectors:

- **Half-pixel precision**: Intermediate values computed by bilinear interpolation of adjacent pixels
- **Quarter-pixel precision**: Bicubic or similar interpolation (H.264 and later)
- **1/8 pixel precision**: Used in AV1

Better sub-pixel accuracy → better motion compensation → smaller residuals → better compression. Cost: more computation for both encoder and decoder.

## Frame Types

Video streams use three types of frames:

```
GOP (Group of Pictures) structure:

I  B  B  P  B  B  P  B  B  P  B  B  I
│  │  │  │  │  │  │                  │
│  │  │  └──┘  └──┘  └──┘            └── next I-frame (keyframe)
│  │  │  │
│  └──┴──┘
│
└── Intra-coded frame (keyframe)
    Self-contained; decoder can start here.
    Largest frames (no inter-prediction).

P-frame (Predicted):
  Predicted from ONE past reference frame.
  Contains motion vectors + residuals.
  Much smaller than I-frames.

B-frame (Bi-directional):
  Predicted from BOTH past AND future frames.
  Smallest of the three — uses best of both references.
  Requires decoder to buffer future frames → adds latency.
```

**I-frames** (keyframes) are where seeking can land. Every I-frame is fully self-contained. Streaming services insert I-frames every few seconds to allow seeks and fast channel-changing.

**P-frames** reference one past frame. A decoder must have correctly decoded the reference frame to decode a P-frame.

**B-frames** reference frames in both directions. Better compression, but adds decode complexity and latency. Live streaming often avoids B-frames; stored video uses them extensively.

## H.264 / AVC: The Universal Codec

H.264 (Advanced Video Coding, 2003) is the codec that made HD video streaming practical. It's in every smartphone, streaming service, browser, and camera. Even as newer codecs have emerged, H.264 remains dominant because it has 15+ years of hardware decoder support.

### H.264 Key Features

**Entropy coding**: Two options:
- **CAVLC** (Context-Adaptive Variable-Length Coding): Simpler, faster, baseline profile
- **CABAC** (Context-Adaptive Binary Arithmetic Coding): Better compression (~10-15%), requires more computation, used in main/high profiles

**Reference frames**: Can reference up to 16 past frames (not just the immediately previous one), enabling better motion compensation for complex motion.

**Deblocking filter**: Applied after reconstruction to reduce 8×8 blocking artifacts. H.264's deblocking filter is in-loop — it runs before reference frames are stored, so future frames use deblocked reference frames.

**Profiles and levels**: H.264 defines multiple profiles (Baseline, Main, High) with increasing feature sets, and levels controlling maximum resolution, bitrate, and buffer sizes.

```
H.264 profiles:

Baseline: No B-frames, CAVLC only. Low-power decoders.
          Used for: video calling, older devices
Main:     B-frames, CABAC. Better compression.
          Used for: standard SD/HD video, some streaming
High:     8×8 intra DCT, adaptive quantization, CABAC.
          Best compression. Used for: Blu-ray, streaming HD
High 10:  10-bit depth. HDR content.
High 4:2:2, High 4:4:4: Professional production.
```

### H.264 Encoder Presets (x264)

The x264 encoder has presets from `ultrafast` to `veryslow`. The slower the preset, the more encoding time is spent looking for better compression, with diminishing returns.

```
x264 preset comparison (1080p at CRF 23):

Preset       Encode speed    File size    PSNR
──────────────────────────────────────────────
ultrafast     ~1500 fps       Large        Low
fast           ~350 fps       Medium       Good
medium         ~150 fps       Medium       Good
slow            ~70 fps       Small        Better
veryslow        ~20 fps       Smaller      Best

CRF (Constant Rate Factor) = quality target.
Lower CRF = higher quality, larger file.
CRF 23 = default; CRF 18 ≈ visually lossless.
```

## H.265 / HEVC: Twice the Compression

H.265 (High Efficiency Video Coding, 2013) targets the same quality as H.264 at half the bitrate, or better quality at the same bitrate.

### Key Differences from H.264

**CTUs instead of macroblocks**: H.264 uses 16×16 macroblocks. H.265 uses Coding Tree Units (CTUs) of up to 64×64 pixels, subdivided recursively using a quadtree structure.

```
CTU quadtree structure:

  64×64 CTU:
  ┌──────────────────────────────────────┐
  │                  │         │CU│CU│   │
  │  Single 32×32 CU │ 16×16   ├──┼──┤   │
  │  (flat region)   │ CU      │CU│CU│   │
  │                  │         ├──┴──┤   │
  │                  ├─────────┤  8×8│   │
  │                  │ 16×16   │     │   │
  │                  │ CU      │     │   │
  └──────────────────┴─────────┴─────┴───┘

Large blocks for uniform regions (efficient).
Small blocks for detailed/complex regions (accurate).
```

**More intra prediction modes**: 35 directions (vs 9 in H.264 high profile).

**SAO (Sample Adaptive Offset)**: An additional in-loop filter that reduces banding and ringing artifacts using lookup tables.

**Better motion compensation**: Asymmetric partition shapes, more reference frames.

### H.265 Licensing Problem

H.265 achieves excellent compression but has a complex patent licensing situation with multiple patent pools. This fragmentation discouraged browser support and pushed Google, Mozilla, Cisco, and others to form the Alliance for Open Media to develop AV1 as a royalty-free alternative.

## AV1: The Open Codec

AV1 (2018) was designed by the Alliance for Open Media specifically to be:
- Royalty-free (no per-unit licensing fees)
- ~30% better compression than H.264 (similar to H.265, sometimes better)
- Better than VP9 (its predecessor from Google)

### AV1 Innovations

**Larger, more flexible block partitions**: Up to 128×128, with 10 different split patterns (including T-shaped and L-shaped).

**More intra prediction modes**: 56 directional angles, plus "smooth" predictors, "palette mode" for flat-colored regions, and "CfL" (Chroma from Luma).

**Frame-level film grain synthesis**: Instead of encoding film grain in each frame (expensive), AV1 can encode grain parameters and resynthesize plausible grain on decode. Saves bits, preserves perceptual "texture."

**Loop Restoration filter**: More advanced post-processing than H.265's SAO.

**Constrained Directional Enhancement Filter (CDEF)**: Reduces ringing artifacts along directional edges.

```
Codec comparison at equivalent quality:

  Bitrate for comparable quality (normalized to H.264):

  H.264:   100%  (baseline)
  VP9:      65%  (~35% savings)
  H.265:    55%  (~45% savings)
  AV1:      50%  (~50% savings)

  At 4K/60fps content:
  H.264 needs ~50 Mbps
  H.265 needs ~25 Mbps
  AV1   needs ~18 Mbps

  Encoding speed (software, 1080p):
  H.264 (x264 fast):    ~600 fps
  H.265 (x265 fast):    ~150 fps
  AV1 (libaom fast):      ~8 fps  ← Very slow software encoder
  AV1 (SVT-AV1 fast):   ~120 fps ← Production-grade fast encoder
```

AV1 hardware encoders (in Intel Arc, NVIDIA RTX 40xx, AMD RDNA3) bring encoding speed to real-time. Hardware decoders are in smartphones (since 2021), smart TVs, and recent Intel/AMD/NVIDIA graphics cards. Netflix, YouTube, and Twitch all use AV1.

## The Complete Video Compression Pipeline

```
Video codec encoding pipeline (H.264/H.265/AV1):

  Raw video frames
        │
        ▼
  Scene change detection
  → Insert I-frame at scene changes
        │
        ▼
  For each frame, for each block:
  ┌──────────────────────────────────────────────┐
  │  1. INTRA PREDICTION or INTER PREDICTION     │
  │     Intra: predict from neighbors in frame   │
  │     Inter: motion estimation against refs,   │
  │            find best motion vector           │
  │                                              │
  │  2. TRANSFORM (DCT or integer approximation)│
  │     Convert residual to frequency domain     │
  │                                              │
  │  3. QUANTIZATION                             │
  │     Divide by quantization step (lossy!)     │
  │     Rate-distortion optimization decides     │
  │     which blocks get more/fewer bits         │
  │                                              │
  │  4. IN-LOOP FILTERING                        │
  │     Deblocking, SAO (H.265), CDEF (AV1)    │
  │                                              │
  │  5. ENTROPY CODING                           │
  │     CABAC (H.264/H.265) or ANS (AV1)       │
  └──────────────────────────────────────────────┘
        │
        ▼
  Video bitstream (NAL units / OBUs for AV1)
        │
        ▼
  Container: MP4 (H.264/H.265), WebM (AV1/VP9), MKV, TS, ...
```

## Rate Control: The Invisible Algorithm

Rate control decides how many bits to allocate per frame. It's as important as the codec itself for real-world quality.

```
Rate control modes:

CRF (Constant Rate Factor): Target a quality level, let bitrate vary.
  Best for: archiving, local storage
  Downside: unknown output file size

CBR (Constant Bit Rate): Exactly N bits per second, always.
  Best for: live streaming, broadcast with fixed bandwidth
  Downside: wastes bits on simple scenes, degrades quality on complex ones

ABR (Average Bit Rate): Target average, let it vary.
  Middle ground. Used by most streaming services.

CQ (Constant Quantizer): Same QP for every frame.
  Simple, rarely optimal. Educational use mainly.

2-pass encoding:
  Pass 1: Analyze video, estimate complexity per scene.
  Pass 2: Allocate bits based on analysis.
  Best quality at target filesize. 2× encoding time.
```

## Why B-Frames Hurt Live Streaming

B-frames reference future frames — which means the encoder can't output a B-frame until it has seen the future frames it references, and the decoder can't display it until it has received those future frames.

```
B-frame latency:

Without B-frames (low latency):
  Capture → Encode → Transmit → Decode → Display
  Latency: ~1–2 frames

With B-frames (better compression):
  Capture → [wait for future frames] → Encode → Transmit
         → [buffer future frames] → Decode → Display
  Latency: 3+ frames minimum

For interactive video (gaming, video calling):
  B-frames are disabled. Latency matters more than compression.

For stored video / VOD streaming:
  B-frames are used. Viewers don't notice 2-second buffer.
```

## Codec Support Matrix

```
Codec support in major contexts (as of 2025):

Context           H.264    H.265    AV1    VP9
────────────────────────────────────────────────────
Chrome/Firefox      ✓        ✓       ✓      ✓
Safari              ✓        ✓       ✓      —
Mobile (iOS)        ✓        ✓      ✓(hw)   —
Mobile (Android)    ✓        ✓      ✓(hw)   ✓
Smart TVs           ✓        ✓      Varies  —
YouTube             ✓        —       ✓      ✓
Netflix             ✓        ✓       ✓      —
Blu-ray             ✓        ✓       —      —
FFmpeg encode       ✓        ✓       ✓      ✓
```

## Summary

- Video compression combines spatial (intra-frame, like images) and temporal (inter-frame) compression.
- Motion estimation finds where blocks moved between frames; motion compensation predicts current blocks from past frames.
- I-frames are keyframes (self-contained), P-frames predict from past, B-frames predict from past and future.
- **H.264** is the universal standard — hardware support everywhere, reasonable compression.
- **H.265/HEVC** is ~50% more efficient than H.264, hampered by patent complexity.
- **AV1** is royalty-free, ~50% more efficient than H.264, hardware acceleration arriving. The future for streaming.
- Rate control and encoder presets are as important as codec choice for real-world quality.
- Live streaming avoids B-frames to minimize latency; stored video uses them for compression.

Next: how databases compress data, where the same LZ and Huffman ideas run at a completely different scale — and where column-oriented storage changes everything.
