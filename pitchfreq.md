# pitchfreq

## What This Is

A pro-quality audio time-stretch and pitch-shift engine written in Zig 0.16. The long-term vision is an ARA plugin for Pro Tools that replaces Serato Pitch 'n Time with a modern, Apple Silicon-native tool featuring drawable pitch and tempo envelopes. Right now we're building the core DSP engine as a CLI tool.

## Zig 0.16 — Critical Notes

This project targets **Zig 0.16.0-dev**. **IMPORTANT: Before writing ANY Zig code, read `~/dev/contexts/zig.md` for the full 0.16 API reference.** Most LLM training data has outdated Zig (0.11-0.13 era) that will not compile. Key things to remember:

- **New I/O**: `pub fn main(init: std.process.Init) !void` — use `init.io`, `init.gpa`, `init.arena`
- **stdout**: `std.Io.File.stdout().writer(io, &.{})` then `&writer.interface`
- **Files**: `std.Io.Dir.cwd()`, pass `io` to all file ops
- **ArrayList**: `.empty` init, pass allocator per-operation: `list.append(allocator, val)`
- **Builtins**: single-arg casts with inferred types: `const x: u8 = @intCast(val)`
- **@typeInfo**: use quoted identifiers: `@typeInfo(T).@"struct"`
- **GPA renamed**: `std.heap.GeneralPurposeAllocator` → `std.heap.DebugAllocator` (prefer `init.gpa`)
- **Async**: `async`/`await` restored in 0.16 via `std.Io` (old `suspend`/`resume`/`nosuspend` still gone). Use `io.async(fn, .{args})` / `future.await(io)`
- **No** `std.io.getStdOut()`, `std.fs.cwd()`, `std.time.sleep()`, `std.os.*`, `usingnamespace` — all gone/moved

## Architecture Principles

### Design for Variable Processing from Day One
The engine must support **per-frame variable** stretch and pitch ratios, even though v1 only does static values. All processing functions accept curve/envelope parameters, not single scalar ratios.

```
// Even for static stretch, we pass a curve (with one constant point)
engine.process(input, output, &stretch_curve, &pitch_curve);
```

### Core Data Model
```
StretchCurve / PitchCurve → array of (time, value) points with interpolation
ProcessContext → input buffer + output buffer + curves + config
Engine → FFT state, phase vocoder, window, scratch buffers
```

### Processing Pipeline (Phase Vocoder)
1. Window input frame (Hann, later Kaiser)
2. Forward FFT
3. Magnitude/phase extraction
4. Phase advancement with **per-frame stretch ratio** from curve
5. Phase locking (identity phase locking around spectral peaks)
6. Inverse FFT
7. Overlap-add to output

### Pitch Shifting = Stretch + Resample
`pitch_shift(audio, ratio) = time_stretch(audio, 1/ratio) + resample(audio, ratio)`
Use high-quality sinc interpolation for resampling.

### Quality Targets (Offline, Not Real-Time)
- FFT size: 4096 (start here, support 8192)
- Hop size: FFT/8 = 512 (high overlap for quality)
- Phase locking: identity phase locking around spectral peaks
- Transient detection: spectral flux, treat transients differently (time-domain or reduced stretch)
- Stereo: Mid/Side processing to preserve stereo image
- Resampling: bandlimited sinc interpolation

## Build Order

### Phase 1: CLI Time Stretch (current)
- Load WAV file
- Apply static time stretch via phase vocoder
- Write output WAV
- `./pitchfreq --input in.wav --output out.wav --stretch 1.25`

### Phase 2: Add Pitch Shifting
- Implement sinc resampler
- Combine stretch + resample for pitch shift
- `./pitchfreq --input in.wav --output out.wav --pitch 1.2`

### Phase 3: Quality Upgrades
- Phase locking (identity phase locking)
- Transient detection + hybrid processing (FFT for harmonic, time-domain for transients)
- Mid/Side stereo
- Kaiser window option

### Phase 4: Variable Curves
- Accept JSON curve definitions for pitch/tempo envelopes
- Partial recompute (only re-render affected regions)
- Frame caching

### Phase 5: Plugin (far future)
- C-compatible API wrapper around Zig engine
- C++ for AAX/ARA integration and UI
- Pro Tools SDK integration

## Build & Run

```sh
zig build              # build
zig build run          # run executable
zig build test         # run all tests
```

## Project Structure

```
src/
  main.zig       — CLI entry point
  root.zig       — library root (public API)
build.zig        — build configuration
build.zig.zon    — package manifest
```

## FFT Library Decision

**pffft** for v1 (BSD-licensed, SIMD-optimized, single C file). FFTW is GPL — not viable for commercial plugin without a commercial license. Production target: Apple Accelerate (vDSP) on macOS for best Apple Silicon performance, pffft as cross-platform fallback.

## DSP Reference Documents

Before writing any DSP code, read these:
- **`~/dev/contexts/pitch-shifting-algorithms.md`** — Algorithm reference: phase vocoder, phase locking, transient detection, WSOLA, hybrid methods, resampling, FFT choices, implementation phases
- **`~/dev/contexts/phase-vocoder-deep-dive.md`** — Deep dive: complete formulas, pseudocode, Python reference implementation of phase-locked vocoder with transient detection

## Key Quality Differentiators (Why This Beats Pitch 'n Time)
- Non-destructive edit model with pitch/tempo envelopes
- Native Apple Silicon (ARM NEON SIMD via Accelerate/vDSP)
- Modern phase vocoder with identity phase locking
- Hybrid transient handling (PV for tonal, time-domain for transients)
- Mid/Side stereo processing (preserves stereo image)
- Region-based partial recompute for responsive editing
