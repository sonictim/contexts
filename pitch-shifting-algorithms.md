# Pitch Shifting & Time Stretching Algorithms Reference

> **Purpose**: Implementation-level reference for audio time-stretch and pitch-shift algorithms. Covers phase vocoder, WSOLA, hybrid methods, resampling, and FFT library choices. Written for the pitchfreq project (Zig 0.16, offline processing, commercial plugin target).

> **Last updated**: 2026-03-26

---

## Table of Contents

1. [Algorithm Overview & Comparison](#1-algorithm-overview--comparison)
2. [Phase Vocoder Core](#2-phase-vocoder-core)
3. [Phase Locking](#3-phase-locking)
4. [Transient Detection & Preservation](#4-transient-detection--preservation)
5. [The Phasiness Problem](#5-the-phasiness-problem)
6. [Overlap-Add & Windowing](#6-overlap-add--windowing)
7. [WSOLA (Time-Domain Alternative)](#7-wsola-time-domain-alternative)
8. [Hybrid Approaches (Pro-Quality)](#8-hybrid-approaches-pro-quality)
9. [Pitch Shifting via Stretch + Resample](#9-pitch-shifting-via-stretch--resample)
10. [High-Quality Resampling](#10-high-quality-resampling)
11. [Mid/Side Stereo Processing](#11-midside-stereo-processing)
12. [FFT Library Decision](#12-fft-library-decision)
13. [Implementation Phases](#13-implementation-phases)
14. [Key References](#14-key-references)

---

## 1. Algorithm Overview & Comparison

| Method | Best For | Weakness | Complexity |
|--------|----------|----------|------------|
| **Phase Vocoder** | Tonal/harmonic content, polyphonic | Smears transients, phasiness | Medium |
| **WSOLA** | Monophonic, speech, transients | Fails on polyphonic material | Low |
| **Sinusoidal Modeling (SMS)** | Clean tonal, monophonic pitch shift | Complex, expensive, polyphonic issues | High |
| **Hybrid (PV + WSOLA)** | Everything (pro quality) | Implementation complexity | High |

**For pitchfreq**: Phase vocoder with identity phase locking + transient detection + hybrid processing at transients. This is the path used by Rubber Band, Elastique, and Pitch 'n Time.

---

## 2. Phase Vocoder Core

### The Pipeline

```
Input audio
  → Window (Hann)
  → Forward FFT (real-to-complex)
  → Extract magnitude & phase per bin
  → Compute instantaneous frequency per bin
  → Accumulate synthesis phase (scaled by stretch ratio)
  → Recombine magnitude + new phase
  → Inverse FFT (complex-to-real)
  → Window again (synthesis window)
  → Overlap-add to output
  → WOLA normalization
```

### Critical Formulas

**Expected phase advance per bin per hop:**
```
omega_k = 2 * pi * k * H_a / N
```
where k = bin index, H_a = analysis hop, N = FFT size.

**Phase difference between frames:**
```
delta_phi(m, k) = phase(m, k) - phase(m-1, k)
```

**Deviation from expected (wrap to [-pi, pi]):**
```
delta_wrapped(m, k) = princarg(delta_phi(m, k) - omega_k)

princarg(x) = x - 2*pi * round(x / (2*pi))
```

**Instantaneous frequency (radians/sample):**
```
inst_freq(m, k) = (omega_k + delta_wrapped(m, k)) / H_a
```

**Synthesis phase accumulation:**
```
phase_synth(m, k) = phase_synth(m-1, k) + H_s * inst_freq(m, k)
```
where H_s = synthesis hop = H_a * stretch_factor.

**Reconstruct frame:**
```
Y(m, k) = magnitude(m, k) * exp(j * phase_synth(m, k))
frame = real(IFFT(Y)) * synthesis_window
```

### Key Parameters for Offline Quality

| Parameter | Value | Notes |
|-----------|-------|-------|
| FFT size | 4096 (start), 8192 (high quality) | Larger = better frequency resolution |
| Hop size | FFT/8 = 512 | 87.5% overlap, very smooth |
| Window | Periodic Hann | COLA-compliant, use `N` not `N-1` in denominator |
| Analysis hop (H_a) | Fixed at 512 | |
| Synthesis hop (H_s) | H_a * stretch_factor | Varies with stretch ratio |

### Half-Spectrum Optimization

For real-valued audio, the FFT is conjugate-symmetric. Only process bins 0 to N/2 (i.e., N/2+1 bins). Use `rfft` / `irfft` variants.

---

## 3. Phase Locking

### Why It Matters
The classic phase vocoder processes each bin independently, preserving **horizontal coherence** (phase continuity across time) but destroying **vertical coherence** (phase relationships between bins within a frame). This causes the characteristic metallic/phasey sound.

### Identity Phase Locking (Laroche & Dolson, 1999)

**The most important quality upgrade.** Algorithm:

1. **Detect spectral peaks** in current frame's magnitude spectrum:
   - Bin k is a peak if `mag[k] > mag[k-1]` AND `mag[k] >= mag[k+1]`

2. **Compute regions of influence** — each peak "owns" bins up to midpoints between adjacent peaks:
   - Region of peak p_i = `[(p_{i-1} + p_i) / 2, (p_i + p_{i+1}) / 2)`
   - First peak's region extends to bin 0, last peak's to N/2

3. **Compute synthesis phase at peaks** using standard PV formula:
   ```
   theta_s(m, p) = theta_s(m-1, p) + H_s * inst_freq(m, p)
   ```

4. **Compute rotation at each peak:**
   ```
   rotation(p) = theta_s(m, p) - theta_a(m, p)
   ```

5. **Apply same rotation to entire region:**
   ```
   theta_s(m, k) = theta_a(m, k) + rotation(p)   for all k in region(p)
   ```

This preserves the original phase relationships within each spectral peak's main lobe.

**Why "identity"?** At stretch_factor = 1.0, output phases exactly equal input phases.

### Scaled Phase Locking (Alternative)

Instead of preserving original phase offsets, scale them:
```
theta_s(m, k) = theta_s(m, p) + [theta_a(m, k) - theta_a(m, p)] * (H_s / H_a)
```
Slightly more artifacts than identity PL but can be more stable for extreme stretch factors.

### Computational Cost

Minimal — one complex multiply per bin, one sine/cosine per peak (can be tabled). Peak detection is O(N/2).

---

## 4. Transient Detection & Preservation

### Why Transients Sound Bad in Phase Vocoders

- Transients are broadband, short-duration events (drums, plucks, consonants)
- PV assumes local stationarity — transients violate this
- Results: smeared attacks, pre-echo artifacts, "splash" instead of "snap"
- This is the #1 complaint about naive phase vocoders

### Spectral Flux Detection

**Half-wave rectified spectral flux** (only counts energy increases):
```
SF+(m) = sum_k max(0, |X(m,k)| - |X(m-1,k)|)^2
```

**Adaptive thresholding:**
```
threshold(m) = median(SF, window=15 frames) + 2.0 * std(SF, window=15 frames)
is_transient(m) = SF+(m) > threshold(m) AND SF+(m) > SF+(m-1)
```

### Handling Strategies (in order of complexity)

**Strategy 1: Phase Reset** (simplest, good results)
At transient frames, reset synthesis phase to analysis phase:
```
if is_transient(m):
    synth_phase = analysis_phase  // direct copy
```
After reset, phase accumulator continues normally from the analysis phase.

**Strategy 2: Reduced Stretch at Transients**
Force stretch_factor = 1.0 at transient frames. Compensate by stretching steady-state regions more. Creates a non-uniform time map.

**Strategy 3: Time-Domain Splicing** (best quality)
Extract transient segments from original audio. Apply phase vocoder only to tonal regions. Splice original transients back at correct positions in stretched output.

**Strategy 4: Harmonic-Percussive Source Separation (HPS)**
Separate signal using median filtering on spectrogram:
- Harmonic mask: median filter along time axis (horizontal)
- Percussive mask: median filter along frequency axis (vertical)
Apply PV to harmonic part, OLA (no phase mod) to percussive part, sum.

---

## 5. The Phasiness Problem

### Symptoms
- Metallic, chorused, or "underwater" timbre
- Loss of presence / distant sound
- Smeared attacks
- Subtle echo/pre-echo artifacts

### Root Cause
A single sinusoidal partial spreads across multiple FFT bins (spectral leakage). The phases of these bins encode the temporal position of the partial within the window. Classic PV breaks these inter-bin phase relationships.

### Fixes (in order of effectiveness)
1. **Identity phase locking** — preserves vertical coherence (Section 3)
2. **Phase gradient heap integration** (Prusha & Holighaus 2017) — propagates phase from high-energy to low-energy bins, handles both sinusoidal and transient components automatically
3. **PVSOLA** — combines PV with waveform-similarity OLA for both phase coherence and transient sharpness
4. **Multi-resolution** — different FFT sizes for different frequency ranges
5. **Noise handling** — add random phase perturbation to noise bins (low magnitude, no peak) to prevent "bubbly" artifacts

---

## 6. Overlap-Add & Windowing

### COLA Constraint
For correct reconstruction: `sum_m w[n - m*H] = constant` for all n.

### Recommended Configuration

| Parameter | Value | Notes |
|-----------|-------|-------|
| Window | Periodic Hann | `w[n] = 0.5 * (1 - cos(2*pi*n/N))` — divide by N, not N-1 |
| Overlap | 87.5% (hop = N/8) | Very smooth, high quality for offline |
| Min overlap | 75% (hop = N/4) | Good balance, Hann is COLA here |

### WOLA Normalization (Critical for Phase Vocoder)

When synthesis hop differs from analysis hop (i.e., during stretching), normalize by sum of squared windows:
```
output[n] = sum_m (frame_m[n - m*H_s])
win_sum[n] = sum_m (window[n - m*H_s])^2
output[n] /= max(win_sum[n], threshold)
```

Threshold = `max(win_sum) * 0.01` to avoid amplifying noise at edges.

### Edge Handling
- Zero-pad input by at least N samples on each side before analysis
- Trim output accordingly: `output = output[pad*stretch : (pad+input_len)*stretch]`
- Apply short fade-in/fade-out to avoid clicks

---

## 7. WSOLA (Time-Domain Alternative)

Waveform Similarity Overlap-Add — a time-domain method.

### Algorithm
1. Read analysis frame at position `m * H_a`
2. Search within tolerance window `delta_max` (typically N/4) for the position that maximizes cross-correlation with the end of the previous output frame
3. Overlap-add at synthesis position `m * H_s`

### When to Use
- Monophonic material (speech, solo instruments)
- Transient-heavy material where PV fails
- As the time-domain component in a hybrid engine

### Limitations
- Fails on polyphonic content (can't find good correlation matches)
- Transient doubling/skipping at extreme ratios
- No frequency-domain control

---

## 8. Hybrid Approaches (Pro-Quality)

### The Professional Strategy

Decompose audio into components, process each optimally:

```
Input → Transient Detection
         ├── Transient frames  → Time-domain pass-through (WSOLA or direct copy)
         └── Tonal frames      → Phase vocoder with phase locking
                                  → Recombine
```

### What the Pros Do

**Rubber Band** (open source, v4.0):
- Phase vocoder with "phase lamination" (variant of phase locking)
- Transient detection → phase reset at transients
- R3 engine for highest quality
- Guide-frequency matching for pitch shifting

**Elastique** (commercial, Zplane):
- Psychoacoustic modeling
- Separate tonal/transient/noise processing
- Adaptive time-frequency resolution

**Melodyne** (Celemony):
- Sinusoidal modeling (partial tracking)
- Individual note/partial manipulation
- DNA (Direct Note Access) for polyphonic

### Recommended Path for pitchfreq

Phase 1: Phase vocoder + identity phase locking + spectral flux transient detection (phase reset)
Phase 2: Hybrid — add time-domain handling at transients
Phase 3: HPS (harmonic-percussive separation) for even better results

---

## 9. Pitch Shifting via Stretch + Resample

### The Method

```
pitch_shift(audio, semitones) =
    ratio = 2^(semitones/12)
    stretched = time_stretch(audio, 1/ratio)
    resampled = resample(stretched, ratio)
```

To pitch UP by ratio r:
1. Time-stretch by 1/r (make it shorter)
2. Resample by r (slow it back down, raising pitch)

To pitch DOWN by ratio r:
1. Time-stretch by 1/r (make it longer)
2. Resample by r (speed it back up, lowering pitch)

Result: same duration, different pitch.

### Why Not Just Resample?

Resampling alone changes both pitch AND duration. The stretch step compensates for the duration change.

---

## 10. High-Quality Resampling

### Windowed Sinc Interpolation

The ideal resampler uses a sinc kernel (infinite length). Practical resamplers window the sinc:

```
y[n] = sum_k x[k] * sinc((n*r - k)) * window((n*r - k) / L)
```
where L = half-length of the filter, r = resampling ratio.

### Key Parameters for Pro Audio

| Parameter | Value | Notes |
|-----------|-------|-------|
| Window | Kaiser, beta 10-14 | 100-130 dB stopband attenuation |
| Filter half-length | 32-64 taps | Longer = better, we're offline |
| Polyphase subfilters | 256-1024 phases | With linear interpolation between phases (Farrow structure) |

### Anti-Aliasing

When pitching UP (effectively downsampling): the anti-aliasing filter cutoff must be at the OUTPUT Nyquist, not input. Scale the sinc kernel by the ratio:
```
if ratio > 1:  // pitching up = downsampling
    cutoff = 1.0 / ratio
    sinc_arg *= ratio
```

### Polyphase Implementation

Precompute P subfilters (one per fractional phase). For arbitrary ratios, interpolate between the two nearest subfilters. This avoids recomputing the sinc kernel for every output sample.

---

## 11. Mid/Side Stereo Processing

### Why Not Process L/R Independently

Independent L/R processing causes:
- Phantom center drift (the PV makes different phase decisions per channel)
- Chorus/phaser artifacts on center-panned content
- Stereo image collapse or widening

### The M/S Approach

```
mid  = (L + R) / 2
side = (L - R) / 2
```

Process mid and side independently through the phase vocoder, then:

```
L = mid + side
R = mid - side
```

### Implementation Notes

- Share transient detection from the mid channel — don't detect independently
- The side channel is typically lower energy — can process more conservatively
- Phase locking should use peaks from the mid channel to guide the side channel
- This maintains mono compatibility and preserves the stereo image

---

## 12. FFT Library Decision

### Licensing Context

pitchfreq will be a **commercial Pro Tools plugin**. GPL libraries are not viable without purchasing a commercial license.

### Recommended Strategy

**Phase 1 (Prototype):** Use **pffft** as sole backend.
- License: BSD-like (from FFTPACK). Fully commercial-friendly.
- Performance: ~2x slower than FFTW, ~5x faster than KissFFT. SIMD: SSE, AVX, AVX2, NEON.
- Integration: Single C file + header. Trivially vendored.
- No memory allocation during execution.
- Power-of-2 sizes supported. The marton78 fork adds double precision.

**Phase 2 (Production):** Platform-specific backends.
- **macOS/iOS:** Apple Accelerate (vDSP) — hand-tuned for Apple Silicon, free, best performance.
- **Windows/Linux:** pffft — BSD-licensed, solid SIMD.
- Compile-time switch: `if (builtin.os.tag == .macos) use_vdsp() else use_pffft()`

### Library Comparison

| Library | License | Perf (n=512) | SIMD | Notes |
|---------|---------|-------------|------|-------|
| FFTW | GPL (commercial available) | 588 ns | All | Best perf, GPL blocks us |
| pffft | BSD-like | 1255 ns | SSE/AVX/NEON | Best commercial option |
| KissFFT | BSD | 6553 ns | None | Too slow for production |
| Apple vDSP | macOS SDK | ~FFTW | Apple Silicon | Best for Mac, not portable |
| muFFT | MIT | 719 ns | SSE only | No ARM/NEON, dead end |

### Zig Integration

**pffft:**
```zig
const pffft = @cImport({ @cInclude("pffft.h"); });
// build.zig: exe.addCSourceFile(.{ .file = b.path("vendor/pffft.c"), .flags = &.{} });
```

**Apple Accelerate:**
```zig
const accel = @cImport({ @cInclude("Accelerate/Accelerate.h"); });
// build.zig: exe.linkFramework("Accelerate");
```

### At Offline Sizes (4096-8192)

All libraries complete in sub-millisecond times. The 2x gap between pffft and FFTW translates to a few extra seconds over a full album — acceptable for offline processing.

---

## 13. Implementation Phases

### Phase 1: Minimal Time Stretch (CLI)
1. WAV I/O (parse headers, read/write PCM f32)
2. Audio buffer types (planar f32, sample rate, channels)
3. FFT wrapper (pffft via C interop)
4. Hann window generation
5. STFT analysis (windowing + forward FFT)
6. Classic phase vocoder (phase accumulation, no locking yet)
7. STFT synthesis (inverse FFT + overlap-add + WOLA normalization)
8. Stretch curve abstraction (constant value for v1)
9. CLI arg parsing (--input, --output, --stretch)

### Phase 2: Quality Upgrades
1. Identity phase locking (spectral peak detection + region propagation)
2. Spectral flux transient detection
3. Phase reset at transients
4. Edge handling (zero-padding, fade in/out)

### Phase 3: Pitch Shifting
1. High-quality sinc resampler (Kaiser windowed, polyphase)
2. Pitch curve abstraction
3. Pitch shift = stretch(1/ratio) + resample(ratio)
4. CLI: --pitch flag

### Phase 4: Pro Quality
1. Hybrid engine (time-domain at transients, PV for tonal)
2. Mid/Side stereo processing
3. HPS (harmonic-percussive separation) option
4. Kaiser window option
5. Multi-resolution option (different FFT sizes per frequency range)
6. Apple Accelerate backend for vDSP FFT

### Phase 5: Variable Curves
1. JSON curve input format
2. Per-frame interpolated stretch/pitch ratios
3. Partial recompute (only re-render affected regions)
4. Frame caching

---

## 14. Key References

### Papers
- Flanagan & Golden (1966). "Phase Vocoder." Bell System Technical Journal. — The original.
- **Laroche & Dolson (1999). "Improved Phase Vocoder Time-Scale Modification."** IEEE Trans. — Identity & scaled phase locking. THE key quality paper.
- Laroche & Dolson (1997). "About This Phasiness Business." ICMC. — Analysis of artifacts.
- Prusha & Holighaus (2017). "Phase Vocoder Done Right." — Phase gradient heap integration.
- Robel (2003). "Transient Detection and Preservation in the Phase Vocoder." ICMC.
- Griffin & Lim (1984). "Signal Estimation from Modified STFT." IEEE Trans.

### Open Source Reference Implementations
- **Rubber Band** (C++, GPL/commercial): https://breakfastquay.com/rubberband/ — State of the art open source.
- **librosa** (Python): `librosa.effects.time_stretch` / `pitch_shift` — Clean reference implementation.
- **SoundTouch** (C++, LGPL): https://www.surina.net/soundtouch/ — WSOLA-based, simpler.
- **paulstretch** (Python): Extreme time stretching, different approach.

### Detailed Implementation Reference
- `~/dev/contexts/phase-vocoder-deep-dive.md` — Full pseudocode, formulas, and complete Python reference implementation of phase-locked vocoder with transient detection.
