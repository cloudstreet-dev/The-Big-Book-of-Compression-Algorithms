# Chapter 9: Audio Compression

Audio compression has a split personality that mirrors the lossless/lossy divide more starkly than almost any other domain. On one side: FLAC, where audiophiles insist on bit-perfect fidelity, and every sample is preserved exactly. On the other: MP3, which threw away half the bits and convinced the world it sounded fine.

Both are right, for different contexts. Understanding audio compression means understanding *what the ear actually hears* — and that turns out to be a surprisingly limited subset of what a microphone records.

## The Physics of Audio Data

Digital audio is a sequence of samples, each representing the amplitude of a sound wave at a point in time. CD quality is:
- 44,100 samples per second (44.1 kHz)
- 16 bits per sample
- 2 channels (stereo)

That's `44100 × 16 × 2 = 1,411,200 bits per second`, or about 10MB per minute uncompressed. A typical song is ~50MB of raw PCM data.

```
Audio signal visualization (1ms of a sine wave at 440 Hz):

Amplitude
  +32767 ┤                 ╭─╮
         ┤              ╭──╯ ╰──╮
         ┤            ╭╯        ╰╮
         ┤           ╭╯          ╰╮
       0 ┤──────────╯──────────────╰──────────
         ┤                          ╭╯  ╰╮
         ┤                        ╭╯     ╰╮
         ┤                      ╭─╯       ╰─╮
  -32768 ┤                    ╭─╯           ╰─╮
         └──────────────────────────────────────→
         0                                    1ms
         (44.1 samples per ms of audio)
```

## FLAC: Lossless Audio Compression

FLAC (Free Lossless Audio Codec, 2001) is the standard for lossless audio. It compresses CD-quality audio to roughly 50-60% of the original size without losing a single sample. You can decompress FLAC and compare it byte-by-byte to the original PCM — they're identical.

### FLAC's Prediction + Residual Architecture

Audio signals are predictable. Given a few past samples, you can estimate the current sample quite accurately. FLAC exploits this through linear prediction:

```
Linear prediction:

Given past samples: x[n-1], x[n-2], ..., x[n-p]
Predict current:    x̂[n] = a₁x[n-1] + a₂x[n-2] + ... + aₚx[n-p]

The residual (prediction error):
    e[n] = x[n] - x̂[n]

For audio signals, residuals are much smaller than samples.
They compress much better.

Example: gradual sine wave
  Samples:   1000, 1050, 1095, 1134, 1165, ...
  Predicted: 1000, 1050, 1095, 1133, 1163, ... (from coefficients)
  Residuals:    0,    0,    0,    1,    2,  ... (tiny errors)
```

FLAC uses *Linear Predictive Coding (LPC)* with:
- Up to 32 predictor coefficients
- Coefficients quantized to integers (for exact arithmetic)
- Multiple sub-frame types (constant, verbatim, fixed predictor, LPC)

### Rice Coding for Residuals

Residuals cluster around zero — they're much more likely to be small than large. FLAC compresses them using *Rice coding*, a variant of Golomb coding:

```
Rice coding with parameter k:

Every integer n is encoded as:
  quotient q = floor(n / 2^k)   → encoded in unary (q zeros, then a 1)
  remainder r = n mod 2^k       → encoded in k bits (binary)

For FLAC (using mapped signed integers):
  0  → 0           (0 ones → unary "1", remainder "")
  1  → 10
  -1 → 11
  2  → 010
  -2 → 011

With k=0: small numbers get short codes.
With k=3: works better when residuals are larger.

FLAC chooses k adaptively per block to minimize bits used.
```

### FLAC Frame Structure

```
FLAC file:

  ┌───────────────────────────────────────────┐
  │ fLaC marker (4 bytes: 0x66 0x4C 0x61 0x43)│
  ├───────────────────────────────────────────┤
  │ Metadata blocks:                          │
  │  STREAMINFO (mandatory, always first)     │
  │  - Sample rate, channels, bit depth       │
  │  - Total samples, MD5 of uncompressed data│
  │  SEEKTABLE (optional, enables fast seek)  │
  │  VORBIS_COMMENT (tags: artist, title...) │
  │  PICTURE (embedded album art)             │
  ├───────────────────────────────────────────┤
  │ Audio frames                              │
  │  Frame header: sample count, channels,    │
  │                blocking strategy          │
  │  Subframes (one per channel):             │
  │    Type: CONSTANT, VERBATIM, FIXED, LPC  │
  │    LPC coefficients (if LPC type)         │
  │    Rice-coded residuals                   │
  │  Frame footer: CRC-16                     │
  └───────────────────────────────────────────┘
```

```
Typical FLAC compression ratios:

Genre                  Ratio
─────────────────────────────────
Classical (sparse)     45% of original size
Speech                 48%
Rock/pop               52%
Electronic/electronic  55%
Already-lossy re-encode 90%+  (can't reclaim what's gone)
```

FLAC is also streamable — each frame is independently decodable. This enables gapless playback, random seeking, and robust error handling.

## MP3: Psychoacoustic Compression

MP3 (MPEG-1 Audio Layer 3, 1993) achieved something remarkable: it convinced millions of people that 128kbps audio sounded "good enough" — a 90% reduction from CD quality. It did this by understanding what the human auditory system actually *processes*.

### The Psychoacoustic Model

Human hearing has several exploitable limitations:

**Frequency masking**: A loud tone at one frequency masks quieter tones at nearby frequencies. If a 1 kHz tone is playing at 80dB, you can't hear a 1.1 kHz tone at 60dB — the louder one masks it.

```
Frequency masking:

Sound     ╭─────────────╮
level(dB) │   Loud       │
          │   tone at    │  ← Audible
          │   1kHz       │
          ╰──────────────╯
          Masking         ╭──────╮  ← Also audible
          threshold       │Quiet │
                          │tone  │
          Hidden by mask: ╰──────╯
                               ↕
                          Would need to be
                          HERE to be heard
          ─────────────────────────────────→ frequency
```

**Temporal masking**: A loud sound masks quieter sounds that occur *slightly before and after* it. The ear needs time to recover from loud stimuli.
- Pre-masking: ~5ms before the loud sound
- Post-masking: ~100ms after the loud sound

**Absolute threshold of hearing**: Some frequencies can't be heard at low volumes regardless (humans can't hear 20kHz at 20dB SPL, for example).

**Critical bands**: The cochlea divides the frequency spectrum into roughly 24 overlapping bands. Masking effects are strongest within the same critical band.

### MP3 Encoding Pipeline

```
MP3 encoding pipeline:

  PCM input (44100 Hz, 16-bit stereo)
       │
       ▼
  Filter bank (32 sub-bands) AND psychoacoustic model run in parallel:
  ┌────────────────────┐   ┌──────────────────────────────────┐
  │ Polyphase filter   │   │ FFT-based psychoacoustic analysis │
  │ bank (32 sub-bands)│   │ - Compute masking thresholds     │
  │                    │   │ - Determine allowed noise per band│
  └────────────────────┘   └──────────────────────────────────┘
       │                         │
       └─────────────┬───────────┘
                     ▼
  MDCT (Modified Discrete Cosine Transform)
  - 576 frequency coefficients per granule
  - Better frequency resolution than polyphase alone
       │
       ▼
  Quantization
  - Each coefficient divided by step size
  - Step sizes chosen so quantization noise stays below masking threshold
  - Bit allocation: louder/more-masked bands get fewer bits
       │
       ▼
  Huffman coding
  - Quantized coefficients coded using Huffman tables
  - Quad-point, pair, and singleton encoders for different value ranges
       │
       ▼
  Bit reservoir management
  - Smooth out bit usage across frames
  - Complex sections use bits saved from simple sections
       │
       ▼
  MP3 frame output (each ~26ms of audio)
```

### The MDCT: Frequency Analysis Over Time

The Modified Discrete Cosine Transform is a variant of the DCT designed for audio:
- Processes overlapping blocks (half-overlap to avoid blocking artifacts)
- Converts 1152 time-domain samples to 576 frequency coefficients
- "Long blocks" for steady-state content, "short blocks" for transients

```
MDCT overlapping blocks:

Block N:    ├────────────────────┤
Block N+1:            ├────────────────────┤
Block N+2:                        ├────────────────────┤
                50% overlap

This overlap-add approach avoids blocking artifacts at the
boundaries of analysis windows.
```

### MP3 Quality at Different Bitrates

```
MP3 bitrate guide:

 64 kbps: Acceptable for speech, AM-radio quality audio
          Ratio: ~22:1 vs CD
128 kbps: "Transparent" for most listeners on most content
          Ratio: ~11:1 vs CD
          Long considered the minimum for music
192 kbps: High quality; differences from original audible only
          on extreme material (cymbals, sustained strings)
          Ratio: ~7:1 vs CD
256 kbps: Very high quality; audibly transparent on almost
          all content for most listeners
320 kbps: Maximum MP3 bitrate; effectively transparent
          Ratio: ~4:1 vs CD
VBR V0:   Variable bitrate, targeting ~245 kbps average;
          better quality than 256 kbps CBR at similar size
```

The "128kbps is transparent" belief is now considered outdated — controlled blind tests show many listeners can distinguish 128kbps from lossless on quality headphones. 192kbps or VBR V2 (~190 kbps) is the current pragmatic minimum for music.

## AAC: The MP3 Successor

Advanced Audio Coding (AAC, 1997) was developed as the successor to MP3. It achieves better audio quality than MP3 at the same bitrate, or equal quality at lower bitrates.

Key improvements over MP3:
- **Longer MDCT windows**: 2048 coefficients (vs 576), better frequency resolution
- **More Huffman codebooks**: MP3 has 32, AAC has 13+
- **Temporal noise shaping (TNS)**: Shapes quantization noise in time, reducing pre-echo artifacts
- **Perceptual noise substitution**: For bands where noise is fully masked, replace the signal with synthesized noise (cheaper to encode)
- **Better stereo handling**: Mid/Side stereo coding, intensity stereo

```
Quality comparison (bitrate for transparent quality):

Format    Bitrate for transparency
──────────────────────────────────
MP3         ~192 kbps
AAC         ~128 kbps    (AAC-LC, "Low Complexity" profile)
AAC-HE      ~64 kbps     (High Efficiency AAC, adds SBR)
AAC-HEv2    ~48 kbps     (adds PS: Parametric Stereo)
Opus        ~96 kbps     (see below)
```

AAC is the standard for Apple Music, YouTube, streaming services, smartphones (AAC over Bluetooth), and broadcast television (AC-3/Dolby Digital is a related format).

### Spectral Band Replication (HE-AAC)

AAC-HE adds *Spectral Band Replication (SBR)* — a clever trick for encoding at very low bitrates:
- Encode only the low-frequency half of the spectrum at full quality
- Encode a compact "envelope" describing the high-frequency energy distribution
- On decode, extrapolate high frequencies from the low-frequency signal using the envelope

At 48kbps, HE-AAC produces reasonable quality by letting the decoder reconstruct the "feel" of the high frequencies without transmitting them explicitly.

## Opus: The Modern All-Purpose Codec

Opus (2012, standardized in RFC 6716) is the newest audio codec and in many ways the most impressive. It was designed for internet use — specifically WebRTC and streaming — and it outperforms every other audio codec across the bitrate range.

```
Codec performance by use case:

Low bitrate (8–24 kbps): Voice, narrow-band
  Winner: Opus (uses SILK mode, speech-optimized)
  Runner-up: AMR-WB

Medium bitrate (32–96 kbps): Music, wideband voice
  Winner: Opus (uses CELT mode, MDCT-based)
  Runner-up: AAC-HE

High bitrate (96+ kbps): High-quality music
  Winner: Opus / AAC-LC (comparable)
  Lossless: FLAC

For voice/teleconference: Always Opus
For streaming music: AAC-LC or Opus
For downloaded music: FLAC (lossless) or AAC-LC (lossy)
```

Opus works by combining two underlying codecs:
- **SILK**: Originally developed by Skype for voice, optimized for speech signals
- **CELT**: A high-quality audio codec based on MDCT with perceptual coding

Opus dynamically switches between them based on content type, and can also use a hybrid mode. This makes it uniquely suited for mixed voice-and-music content (like a podcast with intro music).

```python
# Encoding audio with Opus in Python (using opuslib)
import opuslib

# Create encoder: 48000 Hz, 2 channels, audio application
enc = opuslib.Encoder(48000, 2, opuslib.APPLICATION_AUDIO)
enc.bitrate = 128000  # 128 kbps

# Encode a 20ms frame (960 samples at 48kHz)
pcm_frame = get_pcm_samples(960)  # your audio source
encoded = enc.encode(pcm_frame, 960)

# Decode
dec = opuslib.Decoder(48000, 2)
decoded = dec.decode(encoded, 960)
```

## The Container vs. Codec Distinction

A common confusion: "MP3" is both a codec and a container. But "AAC" is just a codec — it can be stored in several containers: `.aac` (raw), `.m4a` (MPEG-4 container), `.mp4` (video container), `.ogg` (Ogg container, uncommon).

```
Container / Codec combinations:

Container  Extension   Common codecs
─────────────────────────────────────────────
MPEG-4 Audio  .m4a    AAC-LC, HE-AAC
Ogg           .ogg    Vorbis, Opus, FLAC
WebM          .webm   Vorbis, Opus
MPEG-3        .mp3    MP3 only (format = codec)
FLAC          .flac   FLAC only
WAV           .wav    PCM (uncompressed), sometimes others
AIFF          .aiff   PCM (uncompressed), Apple LPCM
```

The container matters for:
- Chapter markers and metadata
- Gapless playback (requires container support)
- Video synchronization
- DRM integration

## What Happens to Transcoded Audio

A critical warning: transcoding between lossy formats (e.g., MP3 → AAC) does not improve quality. It makes it worse.

```
Transcoding degradation:

Original PCM  →  MP3 128kbps  →  re-encoded AAC 128kbps
Quality drop:     step 1           step 2 (additional loss)
Total loss = loss1 + loss2  >  loss1

The artifacts from MP3 become inputs to the AAC encoder,
which can't distinguish "this artifact was deliberately encoded"
from "this is the original signal" and may make different
(equally wrong) decisions about what to discard.
```

Always transcode from the highest-quality source available. For music production: record lossless, distribute lossy.

## Summary

- **FLAC** uses LPC prediction + Rice coding for bit-perfect audio compression at ~50% of original size.
- **MP3** uses psychoacoustic masking models to discard frequencies the human ear can't hear, achieving 10:1+ compression.
- **AAC** improves on MP3 with better MDCT windows, temporal noise shaping, and spectral band replication in HE-AAC variants.
- **Opus** is the modern all-purpose codec — better than everything else across all bitrates, especially for voice.
- Never transcode between lossy formats; always use a lossless source when re-encoding.
- Psychoacoustic models are the key to lossy audio: they define what can be safely discarded by modelling the limits of human perception.

Next: video compression, where everything from this chapter applies, plus an entirely new dimension — time.
