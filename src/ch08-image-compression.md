# Chapter 8: Image Compression

Images are where compression theory meets human perception in the most visible way. A JPEG artifact, a PNG that's larger than the original bitmap, a WebP that's visually indistinguishable from the original at a third the size — these are compression algorithms at work in a domain where "quality" is defined by what humans can see.

Image compression splits cleanly along the lossless/lossy divide. PNG and related formats preserve every pixel exactly. JPEG, WebP in lossy mode, and AVIF discard information your visual system is unlikely to notice. Understanding which approach to use — and why — requires understanding a bit about both the algorithms and the human visual system.

## The Raw Numbers: Why Image Compression Matters

A 4000×3000 pixel photo, 24 bits per pixel (8 bits each R, G, B):

```
Uncompressed: 4000 × 3000 × 3 bytes = 36,000,000 bytes ≈ 36MB

PNG (lossless):      ~18MB   (2× compression on typical photos)
JPEG quality 90:     ~4MB    (9× compression)
JPEG quality 75:     ~2MB    (18× compression, still good quality)
AVIF quality 80:     ~1MB    (36× compression, excellent quality)
```

Modern smartphone photos are 10–20MB because they're stored as JPEG. Without compression, your photo library would be terabytes.

## PNG: Lossless Compression for Images

PNG (Portable Network Graphics) was created in 1995 to replace GIF (which had LZW patent issues). It's lossless: decompressing a PNG gives you the exact original pixel values.

### PNG's Filter Stage

PNG applies a *filter* to each row of pixels before compression. Filters exploit spatial correlation — adjacent pixels tend to be similar — by predicting each pixel from its neighbors and encoding the prediction error instead of the raw value.

```
PNG filter types (applied per row):

None:     Px = raw value
Sub:      Px = raw - left
Up:       Px = raw - above
Average:  Px = raw - floor((left + above) / 2)
Paeth:    Px = raw - paeth_predictor(left, above, upper_left)

Paeth predictor (Philippe Paeth, 1991):
  a = left pixel, b = above pixel, c = upper-left pixel
  p = a + b - c
  pa = |p - a|, pb = |p - b|, pc = |p - c|
  if pa ≤ pb and pa ≤ pc: return a
  elif pb ≤ pc: return b
  else: return c
```

The PNG encoder typically tries all five filter types for each row and picks the one that produces the smallest values (easiest to compress). This adaptive filtering is one reason PNG compresses reasonably well despite using generic DEFLATE.

```
Filter example (a gradient row):

Raw pixels:  100, 102, 104, 106, 108, 110

Sub filter (subtract left):
  100, 2, 2, 2, 2, 2

DEFLATE will easily compress the run of 2s.
Without filtering, DEFLATE has to match the gradually-changing values.
```

### PNG Internals

```
PNG file structure:

  ┌─────────────────────────────────────────┐
  │ PNG Signature: \x89PNG\r\n\x1a\n        │
  ├─────────────────────────────────────────┤
  │ IHDR chunk (13 bytes)                   │
  │  - Width, height (4 bytes each)         │
  │  - Bit depth, color type                │
  │  - Compression method (always 0=DEFLATE)│
  │  - Filter method, interlace method      │
  ├─────────────────────────────────────────┤
  │ Optional chunks: tEXt, gAMA, cHRM, etc.│
  ├─────────────────────────────────────────┤
  │ IDAT chunk(s) — compressed image data  │
  │  - Filtered rows, DEFLATE compressed   │
  │  - May be split across multiple IDATs   │
  ├─────────────────────────────────────────┤
  │ IEND chunk — marks end of file         │
  └─────────────────────────────────────────┘
```

All PNG chunks have a 4-byte CRC. This makes PNG moderately resilient to corruption — a bad chunk can be detected (though not corrected).

### Optimizing PNGs

Because PNG uses DEFLATE, the compressor's settings matter. Default PNG encoders often don't use the best DEFLATE settings. Tools like `oxipng` and `pngcrush` can reduce PNG file sizes by 20–50% without any quality loss by:
- Trying all filter combinations and picking the best per-row
- Using maximum DEFLATE settings
- Removing unnecessary metadata chunks (GPS, camera info)

```bash
# Optimize in place, removing unnecessary metadata
oxipng --opt 4 --strip all image.png

# Typical improvement
Before: 1.2MB
After:  800KB   (33% smaller, identical pixels)
```

### When to Use PNG

- **Icons, logos, UI elements**: Exact colors matter; flat regions with solid colors compress well.
- **Screenshots**: Text is sharp-edged; JPEG would produce visible ringing artifacts.
- **Images with transparency**: PNG supports alpha channels; JPEG does not.
- **Workflow/intermediate images**: When you'll process the image further, preserve all information.
- **Synthetic images**: Computer-generated graphics often have exact flat regions where PNG excels.

**When not to use PNG**: Photographs. A typical photo in PNG is 3–5× larger than a good JPEG at perceptually equivalent quality. The DEFLATE compression engine isn't designed for the statistical properties of photographic data.

## JPEG: Lossy DCT Compression

JPEG (Joint Photographic Experts Group, 1992) is the format that stores almost every photograph in existence. It achieves dramatic compression by:
1. Converting to a perceptually-motivated color space
2. Applying the Discrete Cosine Transform (DCT) to 8×8 blocks
3. Quantizing the DCT coefficients (this is where information is lost)
4. Entropy coding the quantized coefficients

### Step 1: Color Space Conversion (RGB → YCbCr)

The human visual system is much more sensitive to brightness (luminance) changes than to color (chrominance) changes. JPEG exploits this by separating luminance (Y) from two chrominance channels (Cb = blue-yellow, Cr = red-green).

```
RGB to YCbCr conversion:

Y  =  0.299R + 0.587G + 0.114B
Cb = -0.168R - 0.331G + 0.500B + 128
Cr =  0.500R - 0.418G - 0.081B + 128

Why this matters:
- Y channel: human eye is highly sensitive → compress carefully
- Cb, Cr channels: human eye is less sensitive → compress aggressively

Chroma subsampling (4:2:0): Keep all Y values, but only keep
1 Cb and 1 Cr value per 2×2 block of pixels.

  ┌───┬───┬───┬───┐    Y: full resolution
  │ Y │ Y │ Y │ Y │
  ├───┼───┼───┼───┤    Cb, Cr: half resolution in each direction
  │ Y │ Y │ Y │ Y │    (one value per 4 pixels)
  └───┴───┴───┴───┘

This halves chroma data before any further compression.
The result is usually invisible at normal viewing distances.
```

### Step 2: The Discrete Cosine Transform

Each 8×8 pixel block is transformed using the DCT — related to the Fourier transform, but working with cosines only (which avoids the phase issues of the full Fourier transform for real-valued data).

The DCT converts 64 spatial values (pixel intensities) into 64 frequency coefficients. The top-left coefficient (DC) represents the average brightness of the block. The remaining coefficients (AC) represent progressively higher spatial frequencies.

```
An 8×8 pixel block (luminance values):

  139 144 149 153 155 155 155 155
  144 151 153 156 159 156 156 156
  150 155 160 163 158 156 156 156
  159 161 162 160 160 159 159 159
  159 160 161 162 162 155 155 155
  161 161 161 161 160 157 157 157
  162 162 161 163 162 157 157 157
  162 162 161 163 163 158 158 158

After DCT (simplified, showing the low-frequency dominance):

  1260  -1  -12   -5    2   -2    0    0   ← DC + low freq
    -23  -17    -6  -3    2    0    0    0
    -11   -9    -2   2    0    1    0    0
     -7   -2     0   1    0    0    0    0
      2    0     0   0    0    0    0    0
      0    0     0   0    0    0    0    0   ← high freq
      0    0     0   0    0    0    0    0
      0    0     0   0    0    0    0    0

Most energy is concentrated in the top-left (low-frequency) coefficients.
High-frequency coefficients are near zero for natural images.
```

### Step 3: Quantization (The Lossy Step)

Each DCT coefficient is divided by a quantization value from a *quantization table*, then rounded to the nearest integer. This is where information is permanently lost.

```
Quantization table (JPEG standard luminance, quality 50):

  16  11  10  16  24  40  51  61
  12  12  14  19  26  58  60  55
  14  13  16  24  40  57  69  56
  14  17  22  29  51  87  80  62
  18  22  37  56  68 109 103  77
  24  35  55  64  81 104 113  92
  49  64  78  87 103 121 120 101
  72  92  95  98 112 100 103  99

Each DCT coefficient is divided by the corresponding table value and rounded.

Large quantization values → coarse rounding → more information lost (smaller file).
Small quantization values → fine rounding → less information lost (larger file).

"Quality" in JPEG is just a scalar applied to this table:
  Quality 90: divide table by ~3 (fine quantization)
  Quality 50: use table as-is (moderate quantization)
  Quality 10: multiply table by ~8 (coarse quantization)
```

The visual effect of quantization is visible as "blocking" — 8×8 pixel blocks with slightly wrong colors that become obvious at low quality settings.

### Step 4: Zigzag Scan and Entropy Coding

After quantization, the 64 coefficients are read out in a zigzag pattern that puts low-frequency (large) coefficients first and high-frequency (near-zero) coefficients last:

```
Zigzag scan order:

  ┌──┬──┬──┬──┬──┬──┬──┬──┐
  │ 0│ 1│ 5│ 6│14│15│27│28│
  ├──┼──┼──┼──┼──┼──┼──┼──┤
  │ 2│ 4│ 7│13│16│26│29│42│
  ├──┼──┼──┼──┼──┼──┼──┼──┤
  │ 3│ 8│12│17│25│30│41│43│
  ├──┼──┼──┼──┼──┼──┼──┼──┤
  │ 9│11│18│24│31│40│44│53│
  ├──┼──┼──┼──┼──┼──┼──┼──┤
  │10│19│23│32│39│45│52│54│
  ├──┼──┼──┼──┼──┼──┼──┼──┤
  │20│22│33│38│46│51│55│60│
  ├──┼──┼──┼──┼──┼──┼──┼──┤
  │21│34│37│47│50│56│59│61│
  ├──┼──┼──┼──┼──┼──┼──┼──┤
  │35│36│48│49│57│58│62│63│
  └──┴──┴──┴──┴──┴──┴──┴──┘

Reading zigzag order from quantized coefficients:
  1260, -2, -23, -11, -17, -12, -7, ...  0, 0, 0, 0, 0, 0, 0, 0

The long run of zeros at the end is RLE-encoded with a special
end-of-block (EOB) symbol. Then Huffman coding compresses everything.
```

### JPEG Artifacts

The DCT-based approach produces characteristic artifacts:
- **Blocking**: 8×8 grid visible, especially at edges and in smooth gradients
- **Ringing**: Dark halos around sharp edges (Gibbs phenomenon from truncating high-frequency content)
- **Mosquito noise**: Temporal noise in areas of fine texture in videos
- **Color bleeding**: Chroma subsampling creates color fringing at high-contrast edges

```
Where you'll see JPEG artifacts:

  Sharp text on colored background → heavy ringing
  ──────────────────────────────────────────────
  │  HELLO  │  ← Dark halos around the letters
  ──────────────────────────────────────────────

  Flat blue sky → visible 8×8 banding
  Fine diagonal lines → "staircase" pattern

  JPEG is optimized for natural photographic content, not:
  - Text or diagrams with sharp edges
  - Flat synthetic colors
  - Images with text overlaid on photos
```

## WebP: VP8 Intra-Frame Coding

WebP was developed by Google (from the VP8 video codec) and released in 2010. It supports both lossless and lossy modes.

**Lossy WebP** uses VP8's intra-frame prediction with smaller blocks than JPEG (4×4 in addition to 16×16), better chroma handling, and an arithmetic coder instead of Huffman — getting roughly 25–35% better compression than JPEG at equivalent perceptual quality.

**Lossless WebP** is more interesting: it uses a completely different algorithm from PNG, with:
- Spatial prediction (like PNG filters but more sophisticated, including multiple predictors per block)
- Color transform (subtract color correlation between channels)
- Subtract green (G channel before R and B)
- Palette coding (for images with limited colors)
- LZ77 + Huffman on the resulting values

```
WebP lossless vs. PNG on typical web images:

Image type           PNG size    WebP lossless    Savings
──────────────────────────────────────────────────────────
Icon/logo              24KB         18KB            25%
Screenshot            450KB        320KB            29%
Illustration          1.2MB        820KB            32%
Photo (no alpha)     18MB (rare)   15MB             17%
```

WebP lossless is generally 26% smaller than equivalent PNG, with slightly slower encoding and comparable decoding speed.

## AVIF: The Modern Challenger

AVIF (AV1 Image File Format) uses the AV1 video codec's intra-frame encoding to compress still images. Released in 2019 by the Alliance for Open Media (Apple, Google, Netflix, Amazon, et al.), it's the current state of the art for image compression.

### What Makes AVIF Better

AVIF inherits AV1's modern encoding toolkit:
- **Variable block sizes**: 4×4 to 128×128 (vs JPEG's fixed 8×8)
- **More sophisticated intra-prediction**: 56 directional modes + DC and smooth predictors
- **Better chroma subsampling**: 4:4:4, 4:2:2, and 4:2:0, all handled natively
- **Film grain synthesis**: Rather than preserving noise in the bitstream, AVIF can store grain parameters and resynthethize natural-looking film grain on decode
- **HDR and wide gamut**: Native 10-bit and 12-bit support
- **Alpha channel**: In the lossy bitstream, not a separate lossless channel

```
Compression comparison at equivalent quality (DSSIM ≈ 0.001):

Format      File size    Encoding time    Decoding time
──────────────────────────────────────────────────────────
JPEG (MozJPEG)  100%       1×               1×
WebP lossy       72%       1.5×             1×
AVIF             45%      50×              0.5×  (hardware decode)
                          (software encode is slow)
```

AVIF encodes at roughly half the size of JPEG for equivalent perceptual quality. The catch: encoding is currently slow in software (AV1 was designed with hardware in mind). Browser support arrived in 2021 (Chrome, Firefox), with Safari following in 2023.

### AVIF Deployment in Practice

```python
# Using Pillow to encode AVIF
from PIL import Image

img = Image.open('photo.jpg')

# AVIF with quality control
img.save('photo.avif', quality=80)
# quality=100: visually lossless, ~50% smaller than JPEG 95
# quality=80: good quality, ~50% size of JPEG 80
# quality=50: noticeable artifacts, very small

# For web serving: offer AVIF with JPEG fallback
# <picture>
#   <source srcset="photo.avif" type="image/avif">
#   <source srcset="photo.webp" type="image/webp">
#   <img src="photo.jpg">
# </picture>
```

## Format Selection Guide

```
Image format decision tree:

Does the image require transparency (alpha)?
├─ Yes, lossless alpha required → PNG (or WebP lossless)
├─ Yes, lossy acceptable → WebP lossy or AVIF
└─ No → continue

Is it a photograph or complex scene?
├─ No (icon, logo, diagram, screenshot) → PNG or WebP lossless
└─ Yes → continue

Is encoding speed critical?
├─ Yes (dynamic generation, on-the-fly) → JPEG or WebP lossy
└─ No (batch encode, CDN delivery) → AVIF

Is browser support a constraint?
├─ Must support IE/old browsers → JPEG/PNG only
├─ Modern browsers (2021+) → AVIF with WebP/JPEG fallback
└─ Latest browsers only → AVIF

File size priority?
├─ Maximum compression → AVIF
├─ Good compression, fast encode → WebP lossy
└─ Compatibility → JPEG quality 75-85
```

## The Quality-Size Curve

Every lossy format has a quality dial. Understanding how quality translates to visual fidelity helps set appropriate targets:

```
JPEG quality guidelines:

Quality 95+: Visually lossless for most humans. Use for source masters.
             Typical size: 4-5× the AVIF equivalent.

Quality 85:  High quality. Artifacts invisible at normal viewing distance.
             Good for photos where quality is a priority.

Quality 75:  Good quality. Slight artifacts visible at 1:1 pixel.
             Standard web quality for photographs.

Quality 60:  Moderate quality. Visible artifacts on close inspection.
             OK for thumbnails, preview images.

Quality 40:  Low quality. Obvious blocking on smooth gradients.
             Only appropriate for very small sizes or previews.

Quality 20:  Very low. Clear blocking, ringing, color errors.
             Rarely appropriate.
```

## Summary

- **PNG** uses predictive filtering (Paeth, Sub, etc.) + DEFLATE. Lossless, ideal for graphics with flat regions, transparency, and text. Too large for photographs.
- **JPEG** converts to YCbCr, applies DCT to 8×8 blocks, quantizes (lossily), then Huffman-codes. The standard for photos; characteristic blocking and ringing artifacts at low quality.
- **WebP** offers both lossless (26% better than PNG) and lossy (25% better than JPEG) modes. Good choice when broad support is available.
- **AVIF** uses AV1 intra-frame coding. ~50% smaller than JPEG at equivalent quality. Slow to encode in software; the future of web image compression.
- The human visual system cares more about luminance than chrominance — all lossy formats exploit this.

Next: audio compression, where the same DCT-based approach runs on *time*, and psychoacoustics determines what we can safely throw away.
