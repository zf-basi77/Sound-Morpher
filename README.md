# Audio Syllable Morphing with Dynamic Time Warping

This project explores two ways to generate intermediate audio signals between two recordings. It is implemented as a Jupyter notebook and is intended for short sounds, such as spoken or animal-vocalization syllables, where timing and spectral structure may differ between recordings.

The workflow combines **dynamic time warping (DTW)** for temporal alignment with two alternative approaches to audio reconstruction:

1. **WORLD-based morphing** — interpolates pitch and source-filter features before resynthesizing the waveform.
2. **Magnitude-spectrogram morphing** — interpolates aligned spectrograms and reconstructs audio with the Griffin–Lim algorithm.

The notebook also includes visual and spectral checks to help assess whether an intermediate sound behaves more like a smooth frequency transition or a cross-fade between two spectral components.

## Overview

Given recordings A and B, the notebook:

1. Loads both recordings and resamples them to a shared sampling rate.
2. Extracts pitch and spectral features and checks the input analyses.
3. Aligns the recordings using DTW.
4. Generates a sequence of intermediate sounds controlled by a morphing coefficient, `p`.
5. Normalizes output RMS levels and applies short fades at the waveform boundaries.
6. Plots spectrograms and provides notebook audio players for listening.
7. Saves the generated WAV files.
8. Runs a diagnostic focused on spectral ridges to help characterize the transition between the source sounds.

The coefficient `p` ranges from 0 to 1:

- `p = 0`: the A-side representation is selected.
- `p = 1`: the B-side representation is selected.
- Intermediate values blend the aligned representations.

The resynthesized endpoints should be treated as **reconstructions**, not guaranteed bit-for-bit copies of the original recordings.

## Methods

### Method 1: WORLD-based morphing

WORLD separates the audio into features used for speech analysis and synthesis:

- **Fundamental frequency (`F0`)** — the estimated pitch contour.
- **Spectral envelope** — the distribution of spectral energy across frequency.
- **Aperiodicity** — an estimate of the noise-like component of the signal.

DTW is applied to MFCC features derived from the log spectral envelopes to find corresponding time frames. The aligned features are interpolated between A and B. Spectral envelopes and voiced pitch values are interpolated in log space, while aperiodicity is interpolated linearly. WORLD then synthesizes the resulting features into audio.

This method is useful when pitch and source-filter structure are meaningful parts of the transformation. Its quality depends on the accuracy of the pitch estimates and the suitability of the WORLD source model for the sounds being analyzed.

### Method 2: Magnitude-spectrogram morphing

The second method does not use an explicit pitch track or WORLD source model. It:

1. Computes STFT magnitude spectrograms.
2. Uses MFCCs from mel-scaled power spectrograms to calculate a DTW alignment.
3. Interpolates the aligned magnitude spectrograms.
4. Uses Griffin–Lim to estimate phase and reconstruct a waveform.

This provides a comparison with the WORLD approach and can be useful for signals that are noisy or contain multiple components.

**Known limitation:** when A and B contain harmonics or spectral ridges at different frequencies, linear magnitude interpolation can preserve both components at intermediate steps. The result may sound like a cross-fade rather than a single component gliding between frequencies. Griffin–Lim also estimates phase rather than recovering the original phase exactly.

## Diagnostic plots

The notebook includes several checks:

- **Input spectrograms and F0 tracks:** inspect pitch estimates before generating morphs. Incorrect F0 tracking can compromise the WORLD results.
- **DTW path:** a roughly diagonal path suggests similar timing; long horizontal or vertical runs indicate that one recording may be stretched substantially relative to the other.
- **Morph spectrograms:** compare how spectral structure changes across the sequence.
- **Spectral-ridge diagnostic:** examine a selected time window and frequency band. The strongest-peak trajectory, energy in endpoint/intermediate bands, and presence of two distinct peaks provide evidence consistent with a glide-like transition or a cross-fade-like transition.

The ridge diagnostic is a targeted check, not a universal perceptual classifier. Its interpretation depends on choosing a time window and frequency band that contain the same relevant spectral component in both source recordings.

## Requirements

The notebook uses Python and the following packages:

- NumPy
- librosa
- PyWORLD (`pyworld`)
- SoundFile (`soundfile`)
- Matplotlib
- SciPy
- IPython / Jupyter

Install the dependencies in your environment, for example:

```bash
pip install numpy librosa pyworld soundfile matplotlib scipy ipython jupyter
```

PyWORLD installation availability can vary by Python version and operating system. If installation fails, consult the package's installation instructions and use a compatible Python environment.

## Running the notebook

1. Open `morph_syllables.ipynb` in Jupyter Notebook, JupyterLab, or a compatible notebook environment.
2. Set `PATH_A` and `PATH_B` to the two audio files you want to compare.
3. Review the analysis settings below.
4. Run the notebook cells from top to bottom.
5. Listen to the outputs and inspect the plots before interpreting the morph sequence.

The notebook currently contains example path assignments. Replace them with paths that exist on your machine. If multiple example assignments to `PATH_A` and `PATH_B` remain in the settings cell, remember that Python uses the last assignment executed.

## Main settings

| Setting | Purpose |
|---|---|
| `PATH_A`, `PATH_B` | Input audio file paths |
| `FS` | Shared sampling rate; both files are loaded at this rate |
| `N_STEPS` | Number of morph outputs generated, including the two endpoints |
| `FRAME_MS` | WORLD analysis frame period in milliseconds |
| `F0_RANGE_A`, `F0_RANGE_B` | Expected minimum and maximum F0 values for each recording, in Hz |
| `NFFT` | FFT size used for WORLD spectral analysis |
| `SMOOTH_F0` | Optional median-filter length for smoothing the F0 contour; set to `0` to disable |
| `FMAX_PLOT` | Maximum frequency displayed in the main spectrogram plots |
| `N_FFT`, `HOP` | STFT parameters for Method 2 |
| `GL_ITERS` | Number of Griffin–Lim iterations for Method 2 |
| `WIN_MS`, `BAND_HZ` | Time window and frequency band used by the ridge diagnostic |

The notebook's example settings use a sampling rate of 44.1 kHz and seven morph outputs. These are starting points, not universal settings. In particular, choose F0 ranges based on the actual fundamentals visible in the recordings. An overly broad or poorly chosen range can lead to subharmonic or octave errors.

## Output files

By default, the notebook writes WAV files to two local directories:

- `morphs/` — Method 1 (WORLD-based) outputs.
- `morphs_spec/` — Method 2 (spectrogram-based) outputs.

The output filenames include the morph index and percentage. Change the output-directory variables in the notebook if you want to save elsewhere.

## Practical considerations

- **Check the input analysis first.** Review the F0 overlays and aperiodicity plots before trusting WORLD-generated intermediates.
- **Compare both methods.** Differences between the methods can help identify whether an artifact may be associated with pitch/source modeling or with magnitude-spectrogram interpolation and phase reconstruction.
- **Listen as well as inspect.** Spectrograms and peak trajectories are useful diagnostics, but they do not fully characterize perceived sound quality.
- **Interpret endpoint checks realistically.** WORLD resynthesis and Griffin–Lim reconstruction are not expected to reproduce the original waveform sample-for-sample.
- **Use appropriate source recordings.** DTW can align sequences of different durations, but a path with extensive stretching may indicate that the recordings do not share a straightforward frame-by-frame structure.

## Scope

This is an exploratory audio-analysis and synthesis workflow rather than a packaged command-line application. The notebook exposes the analysis choices and diagnostic plots so that users can inspect intermediate results and adapt the parameters to different pairs of recordings.
