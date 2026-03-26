# Phase Vocoder Algorithms: Deep Implementation Reference

## 1. Phase Vocoder Fundamentals: The STFT Analysis/Synthesis Pipeline

### Overview

The phase vocoder operates on the Short-Time Fourier Transform (STFT). Audio is chopped into overlapping windowed frames, each frame is FFT'd to get magnitude and phase per frequency bin, modifications are made in the frequency domain, and the inverse FFT + overlap-add reconstructs the output.

### STFT Analysis

Given input signal `x[n]`, window function `w[n]` of length `N`, and analysis hop size `H_a`:

```
X(m, k) = sum_{n=0}^{N-1} x[n + m*H_a] * w[n] * exp(-j * 2*pi*k*n / N)
```

where:
- `m` = frame index
- `k` = frequency bin index (0 to N-1)
- `X(m, k)` is a complex value encoding magnitude and phase

Extract magnitude and phase:
```
magnitude(m, k) = |X(m, k)|
phase(m, k)     = angle(X(m, k)) = atan2(imag(X(m,k)), real(X(m,k)))
```

### Expected Phase Increment Per Bin Per Hop

This is the critical formula that makes the phase vocoder work. For bin `k`, if the signal in that bin were exactly at the bin's center frequency, the phase would advance by a predictable amount each hop:

```
omega_k = 2 * pi * k * H_a / N
```

where:
- `k` = bin index
- `H_a` = analysis hop size (in samples)
- `N` = FFT size

This is the **nominal phase advance** -- the phase increment you'd expect if bin `k` contained a pure sinusoid at exactly `k * fs / N` Hz.

In vector form (for all bins simultaneously):
```python
omega = 2 * np.pi * np.arange(N) * H_a / N
```

Or equivalently, as in librosa:
```python
phi_advance = hop_length * fft_frequencies(sr=2*pi, n_fft=n_fft)
# which gives: phi_advance[k] = hop_length * (2*pi*k / n_fft) = 2*pi*k*hop/N
```

### Phase Difference and Unwrapping

The measured phase difference between consecutive frames for bin `k`:
```
delta_phi(m, k) = phase(m, k) - phase(m-1, k)
```

The deviation from the expected phase advance:
```
delta_phi_wrapped(m, k) = princarg(delta_phi(m, k) - omega_k)
```

where `princarg` wraps to [-pi, pi]:
```python
def princarg(phase_in):
    """Wrap phase to [-pi, pi]"""
    return phase_in - 2*np.pi * np.round(phase_in / (2*np.pi))
```

This wrapped deviation `delta_phi_wrapped` represents the **true frequency deviation** of the signal in bin `k` from the bin's center frequency.

### Instantaneous Frequency Estimation

The instantaneous frequency (in radians/sample) for bin `k` at frame `m`:
```
inst_freq(m, k) = (omega_k + delta_phi_wrapped(m, k)) / H_a
```

This tells you the actual frequency of the energy in each bin, not just the bin center.

### Synthesis Phase Accumulation for Time Stretching

For time stretching by factor `alpha = H_s / H_a` (where `H_s` is the synthesis hop):

**Classic phase vocoder** -- accumulate the synthesis phase:
```
phase_synth(m, k) = phase_synth(m-1, k) + H_s * inst_freq(m, k)
```

Equivalently:
```
phase_synth(m, k) = phase_synth(m-1, k) + omega_k * (H_s / H_a) + delta_phi_wrapped(m, k) * (H_s / H_a)
```

Or more simply:
```
phase_synth(m, k) = phase_synth(m-1, k) + phi_advance_k + delta_phi_wrapped(m, k)
```
where `phi_advance_k = 2*pi*k*H_s/N` (using synthesis hop now).

### Complete Time-Stretch Algorithm (Pseudocode)

```python
def phase_vocoder_time_stretch(x, stretch_factor, N=2048, H_a=512):
    H_s = int(H_a * stretch_factor)
    window = hann(N)

    # Analysis: compute STFT
    num_frames = (len(x) - N) // H_a + 1
    X = np.zeros((N, num_frames), dtype=complex)
    for m in range(num_frames):
        frame = x[m*H_a : m*H_a + N] * window
        X[:, m] = np.fft.fft(frame)

    magnitudes = np.abs(X)
    phases = np.angle(X)

    # Expected phase advance per bin per analysis hop
    omega = 2 * np.pi * np.arange(N) * H_a / N

    # Initialize synthesis phases from first analysis frame
    synth_phases = phases[:, 0].copy()

    # Synthesis
    output_length = (num_frames - 1) * H_s + N
    output = np.zeros(output_length)
    win_sum = np.zeros(output_length)  # for normalization

    for m in range(num_frames):
        if m > 0:
            # Phase difference between consecutive analysis frames
            dphi = phases[:, m] - phases[:, m-1]

            # Remove expected phase advance, wrap to [-pi, pi]
            dphi_wrapped = dphi - omega
            dphi_wrapped -= 2*np.pi * np.round(dphi_wrapped / (2*np.pi))

            # Instantaneous frequency (rad/sample)
            inst_freq = (omega + dphi_wrapped) / H_a

            # Advance synthesis phase by synthesis hop * inst_freq
            synth_phases += H_s * inst_freq

        # Reconstruct frame with original magnitude, new phase
        Y = magnitudes[:, m] * np.exp(1j * synth_phases)
        frame = np.real(np.fft.ifft(Y)) * window

        # Overlap-add
        start = m * H_s
        output[start : start + N] += frame
        win_sum[start : start + N] += window ** 2

    # Normalize by sum of squared windows
    win_sum[win_sum < 1e-8] = 1e-8
    output /= win_sum

    return output
```

### Librosa's Implementation (Reference)

From the actual librosa source code (`spectrum.py`):

```python
# Expected phase advance in each bin per frame
phi_advance = hop_length * fft_frequencies(sr=2*np.pi, n_fft=n_fft)

# Phase accumulator; initialize to the first sample
phase_acc = np.angle(D[..., 0])

for t, step in enumerate(time_steps):
    columns = D[..., int(step) : int(step + 2)]
    alpha = np.mod(step, 1.0)
    mag = (1.0 - alpha) * np.abs(columns[..., 0]) + alpha * np.abs(columns[..., 1])

    d_stretch[..., t] = phasor(phase_acc, mag=mag)

    # Compute phase advance
    dphase = np.angle(columns[..., 1]) - np.angle(columns[..., 0]) - phi_advance

    # Wrap to -pi:pi range
    dphase = dphase - 2.0 * np.pi * np.round(dphase / (2.0 * np.pi))

    # Accumulate phase
    phase_acc += phi_advance + dphase
```

Notable: librosa interpolates between frames (using fractional `step` indices and `alpha` blending of magnitudes) rather than reading every frame sequentially. It also works directly on the STFT matrix rather than on raw audio.


---

## 2. Phase Locking Techniques

### The Problem Phase Locking Solves

The classic phase vocoder computes synthesis phase independently for each bin. This preserves **horizontal coherence** (phase continuity across time for each bin) but destroys **vertical coherence** (phase relationships between adjacent bins within a single frame).

In the original signal, a single sinusoidal partial spreads across several FFT bins due to windowing (the main lobe of the window). The phase relationships between these bins encode the temporal position of the partial within the window. When the phase vocoder adjusts each bin independently, these relationships break, producing the characteristic "phasiness."

### Peak Detection

Both identity and scaled phase locking begin by finding spectral peaks. A simple peak detection algorithm:

```python
def find_spectral_peaks(magnitudes):
    """Find local maxima in the magnitude spectrum.
    A bin k is a peak if mag[k] > mag[k-1] AND mag[k] > mag[k+1].
    """
    peaks = []
    for k in range(1, len(magnitudes) - 1):
        if magnitudes[k] > magnitudes[k-1] and magnitudes[k] >= magnitudes[k+1]:
            peaks.append(k)
    return peaks
```

More robust approaches use a minimum prominence threshold or require the peak to be the maximum within a window of several bins.

### Region of Influence

Each peak "owns" the bins surrounding it, up to the midpoint between adjacent peaks. For peaks at bins `p_i` and `p_{i+1}`:

```
region_of_influence(p_i) = [ (p_{i-1} + p_i) / 2, (p_i + p_{i+1}) / 2 )
```

For the first peak, the region extends down to bin 0. For the last peak, it extends up to N/2.

```python
def compute_regions(peaks, num_bins):
    """Compute region of influence for each peak."""
    regions = []
    for i, peak in enumerate(peaks):
        if i == 0:
            start = 0
        else:
            start = (peaks[i-1] + peak) // 2 + 1

        if i == len(peaks) - 1:
            end = num_bins
        else:
            end = (peak + peaks[i+1]) // 2 + 1

        regions.append((start, end, peak))
    return regions
```

### Identity Phase Locking

Identity phase locking (Laroche & Dolson, 1999) preserves the exact phase relationships between bins within a region of influence from the original (analysis) STFT.

**Algorithm:**

1. Detect peaks in the current frame's magnitude spectrum.
2. For each peak bin `p`, compute the synthesis phase using the standard phase vocoder formula:
   ```
   theta_s(m, p) = theta_s(m-1, p) + H_s * inst_freq(m, p)
   ```
3. Compute the rotation angle at the peak:
   ```
   rotation(p) = theta_s(m, p) - theta_a(m, p)
   ```
   where `theta_a(m, p)` is the analysis phase of the peak bin.
4. For all bins `k` in the region of influence of peak `p`, set:
   ```
   theta_s(m, k) = theta_a(m, k) + rotation(p)
   ```

In other words: the entire region is rotated by the same angle, preserving the phase differences between bins exactly as they appear in the original analysis.

**Why "identity"?** Because for a stretch factor of 1.0 (no stretching), the output phases are identical to the input phases -- the transform is the identity.

```python
def identity_phase_locking(mag, phase_a, prev_synth_phase, omega, H_a, H_s):
    num_bins = len(mag)
    synth_phase = np.zeros(num_bins)

    # 1. Find peaks
    peaks = find_spectral_peaks(mag)
    if len(peaks) == 0:
        peaks = [np.argmax(mag)]

    # 2. Compute synthesis phase at peaks using standard PV formula
    dphi = phase_a - prev_analysis_phase  # (need prev analysis phase too)
    dphi_unwrapped = dphi - omega
    dphi_unwrapped -= 2*np.pi * np.round(dphi_unwrapped / (2*np.pi))
    inst_freq = (omega + dphi_unwrapped) / H_a

    peak_synth_phase = np.zeros(num_bins)
    for p in peaks:
        peak_synth_phase[p] = prev_synth_phase[p] + H_s * inst_freq[p]

    # 3. Compute rotation angles at peaks
    rotation = peak_synth_phase[peaks] - phase_a[peaks]

    # 4. Propagate to regions of influence
    regions = compute_regions(peaks, num_bins)
    for (start, end, peak), rot in zip(regions, rotation):
        for k in range(start, end):
            synth_phase[k] = phase_a[k] + rot

    return synth_phase
```

### Scaled Phase Locking

Scaled phase locking also locks regions to peaks, but instead of preserving the original analysis phases, it scales the phase deviation from the peak:

```
theta_s(m, k) = theta_s(m, p) + [theta_a(m, k) - theta_a(m, p)] * (H_s / H_a)
```

This means the phase relationship between a bin and its peak is scaled by the same time-stretch ratio. Scaled phase locking tends to sound somewhat more "phasey" than identity phase locking but can be more stable for large stretch factors.

### Comparison

| Technique | Vertical Coherence | Identity at ratio=1 | Quality |
|---|---|---|---|
| Classic PV | None | No | Phasey, metallic |
| Identity PL | Preserved from analysis | Yes | Good, natural |
| Scaled PL | Scaled approximation | Yes | Good, slightly more artifacts |

### Implementation Note

Both phase locking techniques require only:
- One cosine and one sine evaluation per peak (which can be tabulated)
- One complex multiply per FFT bin

This makes them computationally cheap -- the overhead beyond a classic phase vocoder is minimal.


---

## 3. Transient Detection and Preservation

### Why Transients Sound Bad in a Naive Phase Vocoder

Transients (drum hits, plucks, consonants) have broadband energy concentrated in a very short time window. The phase vocoder assumes signals are locally stationary (well-described by sinusoids within each analysis window). Transients violate this assumption:

1. **Transient smearing**: When time-stretching, the phase vocoder spreads the transient over `stretch_factor` frames. A snare hit that occupied 5ms gets stretched to 10ms or more, losing its sharp attack.

2. **Pre-echo / "reverb" artifact**: The analysis window captures energy from the transient even in frames before the attack. During synthesis, this energy gets spread out, creating a ghost of the transient before the actual hit.

3. **Phase scrambling**: Transients have complex phase relationships across all bins (they need coherent phases to constructively interfere at one point in time). The phase vocoder disrupts these relationships, turning a sharp click into a diffuse "splash."

### Spectral Flux Transient Detection

Spectral flux measures how rapidly the spectrum is changing. High spectral flux = likely transient.

**Standard spectral flux:**
```
SF(m) = sum_k (|X(m,k)| - |X(m-1,k)|)^2
```

**Half-wave rectified spectral flux** (better for onset detection -- only counts increases in energy):
```
SF+(m) = sum_k (H(|X(m,k)| - |X(m-1,k)|))^2
```

where `H(x) = max(0, x)` is the half-wave rectifier (only positive differences count).

```python
def spectral_flux_hwr(magnitudes):
    """Half-wave rectified spectral flux for transient detection."""
    num_frames = magnitudes.shape[1]
    flux = np.zeros(num_frames)

    for m in range(1, num_frames):
        diff = magnitudes[:, m] - magnitudes[:, m-1]
        diff_positive = np.maximum(0, diff)  # half-wave rectify
        flux[m] = np.sum(diff_positive ** 2)

    return flux

def detect_transients(flux, threshold_multiplier=1.5):
    """Detect transients using adaptive thresholding."""
    # Use a running median as the adaptive threshold
    median_window = 10  # frames
    transients = []

    for m in range(median_window, len(flux)):
        local_median = np.median(flux[m-median_window:m])
        local_mean = np.mean(flux[m-median_window:m])
        threshold = local_median + threshold_multiplier * (local_mean - local_median)

        if flux[m] > threshold and flux[m] > flux[m-1]:
            transients.append(m)

    return transients
```

### Group Delay Method (Robel 2003)

An alternative approach uses group delay to estimate the temporal position of energy within each analysis window:

```
COG_k(m) = -d(phase(m,k)) / d(omega)  (group delay at bin k)
```

When the center of gravity (COG) of spectral peaks is close to the center of the analysis window, it indicates a transient at that position. This method works at the level of individual spectral peaks, so stationary components are unaffected.

### What To Do When You Find a Transient

**Strategy 1: Phase Reset**

At detected transient frames, reset the synthesis phase to the analysis phase:
```python
if is_transient[m]:
    synth_phase[:, m] = analysis_phase[:, m]  # direct copy
else:
    synth_phase[:, m] = synth_phase[:, m-1] + H_s * inst_freq[:, m]  # normal PV
```

The phase reset must be synchronized across all bins belonging to the same transient to prevent the perceived attack from disintegrating. This means: when you detect a transient, reset phases for ALL bins in that frame (or at least all bins whose spectral peaks are associated with the transient).

After a reset, the phase accumulator picks up from the analysis phase and continues normally.

**Strategy 2: Reduced Stretch at Transients**

Force the local stretch factor to 1.0 at transient frames (no stretching), and compensate by stretching the steady-state portions more:

```python
def adjust_time_map(transient_frames, total_frames, target_stretch):
    """Create a non-uniform time map that preserves transients."""
    time_map = np.zeros(total_frames)

    # Mark transient regions (e.g., +/- 2 frames around each transient)
    is_transient = np.zeros(total_frames, dtype=bool)
    for t in transient_frames:
        is_transient[max(0, t-2):min(total_frames, t+3)] = True

    # Transient frames map 1:1; steady-state frames absorb extra stretch
    num_transient = np.sum(is_transient)
    num_steady = total_frames - num_transient

    # Total output frames needed
    total_output = int(total_frames * target_stretch)
    # Transient frames contribute 1:1
    steady_stretch = (total_output - num_transient) / max(1, num_steady)

    output_pos = 0
    for m in range(total_frames):
        time_map[m] = output_pos
        if is_transient[m]:
            output_pos += 1  # no stretch
        else:
            output_pos += steady_stretch

    return time_map
```

**Strategy 3: Time-Domain Splicing**

Extract transient segments from the original audio (using time-domain windowing), apply phase vocoder only to the steady-state portions, then splice the original transients back in at the correct positions in the stretched output.

**Strategy 4: Harmonic-Percussive Source Separation (HPS)**

Separate the signal into harmonic and percussive components using median filtering on the spectrogram:
- Harmonic: median filter along time axis (horizontal)
- Percussive: median filter along frequency axis (vertical)

Apply phase vocoder to the harmonic component and OLA (no phase modification) to the percussive component, then sum:

```python
def harmonic_percussive_stretch(x, stretch_factor, N=2048, H=512, median_size=31):
    S = stft(x, N, H)
    mag = np.abs(S)

    # Median filtering for separation
    from scipy.ndimage import median_filter
    mag_harmonic = median_filter(mag, size=(1, median_size))  # smooth in time
    mag_percussive = median_filter(mag, size=(median_size, 1))  # smooth in freq

    # Create soft masks
    mask_h = mag_harmonic / (mag_harmonic + mag_percussive + 1e-10)
    mask_p = 1 - mask_h

    S_harmonic = S * mask_h
    S_percussive = S * mask_p

    # Phase vocoder for harmonic part
    S_h_stretched = phase_vocoder(S_harmonic, stretch_factor)

    # OLA (simple time-domain stretch) for percussive part
    x_percussive = istft(S_percussive)
    x_p_stretched = ola_stretch(x_percussive, stretch_factor)

    return istft(S_h_stretched) + x_p_stretched
```


---

## 4. The "Phasiness" Problem

### What It Sounds Like

Phasiness manifests as:
- **Metallic, chorused, or "underwater" timbre** -- especially on sustained sounds
- **Loss of presence** -- the audio sounds distant, as if recorded in a reverberant room
- **Smeared attacks** -- transients lose their sharpness
- **"Echo" artifacts** -- subtle repetitions or pre-echoes of transient events

### Root Cause: Loss of Vertical Phase Coherence

A single sinusoidal partial at frequency `f` spreads across multiple FFT bins due to the analysis window's spectral leakage (the main lobe of the window function). In the original signal, the phases of these bins have a specific relationship that encodes **where in time** the partial sits within the analysis window.

The classic phase vocoder computes a new phase for each bin independently. Even though each bin's phase is individually correct for its instantaneous frequency, the **relative phases between bins** are no longer consistent. When the inverse FFT reconstructs the time-domain signal, the partial's energy no longer constructively interferes at a single point -- it gets spread out, creating the diffuse, reverberant quality.

**Mathematical explanation:**

For a sinusoid at frequency `f_0` centered at time `t_0` within the analysis window, the STFT phase at bin `k` is approximately:

```
phase(k) = 2*pi*f_0*t_0 + phase_of_window_transform(k - k_0)
```

where `k_0 = f_0 * N / fs` is the fractional bin index. The second term captures the window's contribution. The classic phase vocoder modifies each `phase(k)` independently, breaking the relationship imposed by the window transform term.

### Horizontal vs Vertical Coherence

- **Horizontal coherence** (across time): The phase at bin `k` in frame `m+1` should be the phase at bin `k` in frame `m` plus the expected phase advance for the actual frequency in that bin. The classic phase vocoder preserves this -- that is its core operation.

- **Vertical coherence** (across frequency): The phases of bins within the main lobe of a spectral peak should have a consistent relationship determined by the analysis window and the partial's time offset. The classic phase vocoder destroys this.

Phasiness arises primarily from loss of **vertical** coherence. The classic PV was designed (Flanagan, 1966) to preserve horizontal coherence but did not address vertical coherence until Laroche & Dolson (1999).

### Techniques to Fix Phasiness

**1. Phase Locking (Identity or Scaled)** -- See Section 2 above. This is the primary fix. By locking the phases within a peak's region of influence, vertical coherence is preserved.

**2. Phase Gradient Heap Integration (Prusha & Holighaus, 2017)**

A more recent approach that automatically enforces both horizontal and vertical coherence without explicit peak picking. The algorithm:
- Sorts all time-frequency bins by magnitude (using a max-heap)
- Starting from the highest-magnitude bin, propagates phase outward to neighbors
- Phase is propagated in the direction of the phase gradient
- This naturally handles both sinusoidal and transient components

The key insight: phase should be propagated from high-energy bins to low-energy bins, because high-energy bins have more reliable phase estimates.

**3. PVSOLA (Phase Vocoder with Synchronized Overlap-Add)**

Combines the phase vocoder with waveform-similarity OLA:
- Use the phase vocoder for frequency-domain processing
- Use waveform similarity (like WSOLA) to find optimal overlap positions
- This preserves both phase coherence and transient sharpness

**4. Using Longer Windows / Multi-Resolution**

Using different FFT sizes for different frequency ranges:
- Long windows for low frequencies (better frequency resolution)
- Short windows for high frequencies (better time resolution)

This reduces phasiness because each frequency range is analyzed at an appropriate resolution.

**5. Noise Handling**

For noisy signals, the phase vocoder can introduce correlations at the analysis window rate, making noise sound "tonal" or "bubbly." Fix: add random phase perturbation to bins identified as noise (low magnitude, no clear peak).


---

## 5. Overlap-Add Reconstruction

### The Basic OLA Process

After modifying each STFT frame and taking the inverse FFT, frames are recombined:

```python
output = np.zeros(output_length)
for m in range(num_frames):
    frame = ifft(modified_spectrum[m]) * synthesis_window
    start = m * H_s
    output[start : start + N] += frame
```

### The COLA (Constant Overlap-Add) Constraint

For perfect reconstruction (unmodified spectrum in, identical signal out), the analysis/synthesis window system must satisfy:

```
sum_m w[n - m*H] = C   for all n
```

where `C` is a constant (typically 1). This means: at every sample position, the sum of all overlapping window values equals a constant.

**Weak COLA:** The window's DFT has zeros at harmonics of the frame rate: `W(2*pi*k/R) = 0` for `k = 1, 2, ..., R-1`. Allows aliasing but relies on cancellation.

**Strong COLA:** The window is bandlimited to `|omega| < pi/R`. No aliasing at all -- more robust for modified spectra (which is what the phase vocoder produces).

### Window Choices and Their COLA Properties

**Hann Window:**
- COLA-compliant at 50% overlap (hop = N/2), 75% overlap (hop = N/4), and generally any overlap of `k/(k+1)` for integer `k`
- The most commonly used window for phase vocoders
- With 75% overlap: excellent frequency resolution and robust COLA compliance
- Sum of squared Hann windows at 75% overlap = constant (important for WOLA)

```python
# Hann window, N=2048, 75% overlap
N = 2048
H = N // 4  # = 512
window = 0.5 * (1 - np.cos(2 * np.pi * np.arange(N) / N))  # periodic Hann
```

**Important:** Use the **periodic** Hann window (divide by `N`, not `N-1`) for FFT-based processing. The symmetric version (divide by `N-1`) does not have the same clean COLA properties.

**Kaiser Window:**
- More flexible: the beta parameter controls the sidelobe level
- Not inherently COLA at standard overlaps -- requires careful tuning
- Beta=8, length 33, hop=6 gives ~58 dB stopband rejection
- Useful when you need better sidelobe suppression than Hann provides

**Blackman Family:**
- COLA at 2/3 overlap (hop = N/3)
- Better sidelobe suppression than Hann

**Rectangular Window:**
- COLA at 0% overlap (trivially)
- Never used for phase vocoders (terrible frequency resolution, spectral leakage)

### Common COLA Configurations for Phase Vocoders

| Window | Overlap | Hop Size | COLA Sum | Notes |
|--------|---------|----------|----------|-------|
| Hann | 50% | N/2 | 1.0 | Minimum viable overlap |
| Hann | 75% | N/4 | 1.5 | Most common choice |
| Hann | 87.5% | N/8 | -- | Very smooth, high quality |
| Blackman | 66.7% | N/3 | const | Good sidelobe suppression |

### Normalization: WOLA (Weighted Overlap-Add)

In a phase vocoder, the synthesis hop `H_s` differs from the analysis hop `H_a`, so the standard COLA sum may not hold. The solution is **WOLA normalization**: divide by the sum of squared windows.

```python
# During overlap-add synthesis:
output = np.zeros(output_length)
window_sum = np.zeros(output_length)

for m in range(num_frames):
    frame = np.real(np.fft.ifft(Y[:, m])) * window
    start = m * H_s
    output[start : start + N] += frame
    window_sum[start : start + N] += window ** 2

# Normalize
window_sum = np.maximum(window_sum, 1e-8)  # avoid division by zero
output /= window_sum
```

**Why squared windows?** If the same window is used for both analysis and synthesis, each sample sees the window applied twice: once during analysis windowing, once during synthesis windowing. The effective window is `w[n]^2`, so the normalization must divide by the sum of `w[n]^2` across overlapping frames.

The Hann window at 75% overlap has a constant sum of squared windows (= 1.5), which is why it works so well for phase vocoders.

**Root-window approach:** Any positive COLA window can be split into analysis and synthesis windows by taking its square root:
```
w_analysis[n] = sqrt(w[n])
w_synthesis[n] = sqrt(w[n])
```
Then `w_analysis * w_synthesis = w`, and if `w` is COLA, perfect reconstruction is achieved.

### Handling the Edges

At the beginning and end of the audio, not all overlapping frames are present. This causes:
- Reduced amplitude at edges (fewer frames contributing)
- The normalization denominator (window sum) drops toward zero

**Solutions:**

1. **Zero-pad the input** before analysis: add at least `N` zeros at the beginning and `N` at the end. This ensures all frames have full context, and you can trim the output afterward.

```python
# Pad before STFT analysis
pad_length = N
x_padded = np.concatenate([np.zeros(pad_length), x, np.zeros(pad_length)])
# ... process ...
# Trim output
output = output[int(pad_length * stretch_factor) : int((pad_length + len(x)) * stretch_factor)]
```

2. **Fade in/out**: Apply a short linear or raised-cosine fade at the edges of the output to avoid clicks from the incomplete overlap region.

3. **Clamp the normalization denominator**: When the window sum is too small (edge regions), clamp it to avoid amplifying noise:

```python
window_sum = np.maximum(window_sum, threshold)
# where threshold is e.g. max(window_sum) * 0.01
```

4. **Librosa's approach**: Pad the STFT matrix with zero columns at the end so interpolation at the boundary doesn't go out of bounds:
```python
padding[-1] = (0, 2)
D = np.pad(D, padding, mode="constant")
```


---

## Appendix: Complete Phase-Locked Vocoder Implementation

```python
import numpy as np
from scipy.signal import get_window

def phase_locked_vocoder(x, stretch_factor, n_fft=2048, hop_ratio=4,
                         phase_lock=True, transient_detection=True):
    """
    Phase vocoder with identity phase locking and transient detection.

    Args:
        x: input audio signal (mono, float)
        stretch_factor: >1 slows down, <1 speeds up
        n_fft: FFT size
        hop_ratio: window_size / hop_size (typically 4 for 75% overlap)
        phase_lock: enable identity phase locking
        transient_detection: enable spectral flux transient detection

    Returns:
        y: time-stretched audio signal
    """
    H_a = n_fft // hop_ratio          # analysis hop
    H_s = int(H_a * stretch_factor)   # synthesis hop

    window = get_window('hann', n_fft, fftbins=True)  # periodic Hann

    num_bins = n_fft // 2 + 1

    # Pad input
    x = np.concatenate([np.zeros(n_fft), x, np.zeros(n_fft)])

    # ---- STFT Analysis ----
    num_frames = (len(x) - n_fft) // H_a + 1
    X = np.zeros((num_bins, num_frames), dtype=complex)
    for m in range(num_frames):
        frame = x[m * H_a : m * H_a + n_fft] * window
        X[:, m] = np.fft.rfft(frame)

    magnitudes = np.abs(X)
    phases = np.angle(X)

    # Expected phase advance per analysis hop
    omega = 2 * np.pi * np.arange(num_bins) * H_a / n_fft

    # ---- Transient Detection ----
    is_transient = np.zeros(num_frames, dtype=bool)
    if transient_detection:
        flux = np.zeros(num_frames)
        for m in range(1, num_frames):
            diff = magnitudes[:, m] - magnitudes[:, m - 1]
            flux[m] = np.sum(np.maximum(0, diff) ** 2)

        # Adaptive threshold
        from scipy.ndimage import median_filter
        flux_median = median_filter(flux, size=15)
        flux_std = np.sqrt(median_filter((flux - flux_median)**2, size=15))
        threshold = flux_median + 2.0 * flux_std

        for m in range(1, num_frames):
            if flux[m] > threshold[m] and flux[m] > flux[m-1]:
                is_transient[m] = True

    # ---- Phase Vocoder Synthesis ----
    synth_phases = phases[:, 0].copy()
    prev_analysis_phase = phases[:, 0].copy()

    output_length = (num_frames - 1) * H_s + n_fft
    output = np.zeros(output_length)
    win_sum = np.zeros(output_length)

    for m in range(num_frames):
        if m > 0:
            # Phase difference
            dphi = phases[:, m] - prev_analysis_phase

            # Remove expected advance, wrap to [-pi, pi]
            dphi_dev = dphi - omega
            dphi_dev -= 2 * np.pi * np.round(dphi_dev / (2 * np.pi))

            # Instantaneous frequency (rad/sample)
            inst_freq = (omega + dphi_dev) / H_a

            if is_transient[m]:
                # Phase reset: use analysis phase directly
                synth_phases = phases[:, m].copy()
            elif phase_lock:
                # Identity phase locking
                # Step 1: compute PV phase at peak bins
                peaks = []
                mag = magnitudes[:, m]
                for k in range(1, num_bins - 1):
                    if mag[k] > mag[k-1] and mag[k] >= mag[k+1]:
                        peaks.append(k)
                if len(peaks) == 0:
                    peaks = [np.argmax(mag)]

                # Step 2: advance peaks normally
                pv_phase = synth_phases + H_s * inst_freq

                # Step 3: compute rotation angle at each peak
                rotations = pv_phase[peaks] - phases[peaks, m]

                # Step 4: propagate to regions
                new_synth = np.zeros(num_bins)
                for i, peak in enumerate(peaks):
                    # Determine region boundaries
                    if i == 0:
                        start = 0
                    else:
                        start = (peaks[i-1] + peak) // 2 + 1
                    if i == len(peaks) - 1:
                        end = num_bins
                    else:
                        end = (peak + peaks[i+1]) // 2 + 1

                    # Apply rotation to entire region
                    new_synth[start:end] = phases[start:end, m] + rotations[i]

                synth_phases = new_synth
            else:
                # Classic phase vocoder
                synth_phases = synth_phases + H_s * inst_freq

        prev_analysis_phase = phases[:, m].copy()

        # Reconstruct frame
        Y = magnitudes[:, m] * np.exp(1j * synth_phases)
        frame = np.fft.irfft(Y, n=n_fft) * window

        # Overlap-add
        start = m * H_s
        output[start : start + n_fft] += frame
        win_sum[start : start + n_fft] += window ** 2

    # Normalize
    win_sum = np.maximum(win_sum, np.max(win_sum) * 0.01)
    output /= win_sum

    # Trim padding
    pad_out = int(n_fft * stretch_factor)
    output = output[pad_out:]

    return output
```

---

## Key References

These are the foundational papers and resources on which this document is based:

- Flanagan, J.L. & Golden, R.M. (1966). "Phase Vocoder." Bell System Technical Journal. -- The original phase vocoder.
- Laroche, J. & Dolson, M. (1999). "Improved Phase Vocoder Time-Scale Modification of Audio." IEEE Trans. Speech and Audio Processing. -- Identity and scaled phase locking.
- Prusha, Z. & Holighaus, N. (2017). "Phase Vocoder Done Right." -- Phase gradient heap integration.
- Robel, A. (2003). "Transient Detection and Preservation in the Phase Vocoder." ICMC. -- Group delay transient detection.
- Roebel, A. (2003). "A New Approach to Transient Processing in the Phase Vocoder." DAFx. -- Spectral peak-level transient handling.
- Laroche, J. & Dolson, M. (1997). "About This Phasiness Business." ICMC. -- Analysis of phasiness artifacts.
- Griffin, D. & Lim, J. (1984). "Signal Estimation from Modified Short-Time Fourier Transform." IEEE Trans. ASSP. -- Griffin-Lim algorithm.
