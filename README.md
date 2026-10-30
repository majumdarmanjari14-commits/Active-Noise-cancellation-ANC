# ANC from Scratch

Building active noise cancellation step by step: from two sine waves cancelling, to an FxLMS adaptive filter in Python, to a hardware prototype.

Each step is a short notebook with one idea, one experiment and what I learned from it.

## Roadmap

| # | Step | Status |
|---|------|--------|
| 01 | Superposition and phase error: why timing decides everything | Done |
| 02 | FIR filters: delay, flip and reshape with weights | Done |
| 03 | Delay as phase shift across frequencies | To do |
| 04 | LMS system identification | To do |
| 05 | Adaptive noise canceller (Widrow) | To do |
| 06 | Secondary path: why plain LMS fails | To do |
| 07 | FxLMS with perfect and imperfect path estimates | To do |
| 08 | Secondary-path identification | To do |
| 09 | Real recorded noise, dB reduction vs frequency | To do |
| 10 | Hardware: 1-D duct prototype | To do |
| 11 | Hardware: ear cup | To do |

---

## Step 01 – Sine-wave superposition and phase error

📓 [`notebooks/01_superposition_phase_error.ipynb`](notebooks/01_superposition_phase_error.ipynb)

**Question:** If anti-noise is a flipped copy of the noise, they cancel. How much cancellation survives if the anti-noise arrives slightly late?

**Setup:** A 200 Hz tone sampled at 8 kHz. The anti-noise is the same tone, inverted, with a phase error φ.

### Perfect timing vs a 20° phase error

![Noise, anti-noise and residual at 0° and 20° phase error](figures/01_waves.png)

- **0°:** the noise (blue) and anti-noise (orange) are exact mirror images. They add to zero at every instant, so the residual (black) is flat.
- **20°:** the anti-noise is about 0.28 ms late. The residual is a smaller wave at the same 200 Hz, with amplitude ≈ 0.35 instead of 1.

### Cancellation vs phase error

For two equal-amplitude waves, the leftover amplitude is $2\sin(\varphi/2)$.

![Noise change in dB against phase error](figures/01_cancellation_vs_phase.png)

| Phase error | Noise change | Meaning |
|---|---|---|
| ~1° | about −40 dB | almost silence |
| 20° | about −9 dB | clearly quieter, not silent |
| 60° | 0 dB | no benefit at all |
| > 60° | positive | ANC makes the noise **louder** |

**Takeaway:** cancellation needs very precise timing. A fraction of a millisecond decides between silence and extra noise. Keeping the phase error near zero, continuously, is the whole job of the adaptive filter.

---

## Step 02 – FIR filters

📓 [`notebooks/02_fir_filters.ipynb`](notebooks/02_fir_filters.ipynb)

**Question:** What is the tool that turns a noise signal into anti-noise?

An FIR filter is a weighted sum of the most recent input samples:

$$y[n] = w_0\,x[n] + w_1\,x[n-1] + \dots + w_{L-1}\,x[n-L+1]$$

Only the weights change between the experiments below.

### Part 1 – Same input, four different weight lists

![Pass-through and 10-sample delay](figures/02_pass_and_delay.png)

![Flip and 8-sample average](figures/02_flip_and_average.png)

| Weights | Effect seen |
|---|---|
| `[1]` | output identical to input |
| `[0]*10 + [1]` | shifted right by 10 samples = 1.25 ms at 8 kHz |
| `[-1]` | flipped upside down: this is anti-noise |
| `[1/8]*8` | slightly smaller (≈ 0.93) and slightly late (3.5 samples ≈ 0.44 ms) |

The averager both reshapes **and** delays. Any real filter in the ANC chain adds phase shift like this, which is exactly the problem from Step 01.

### Part 2 – The averager as a low-pass filter

![200 Hz plus 2000 Hz before and after averaging](figures/02_lowpass.png)

The 2000 Hz zigzag disappears completely, and the 200 Hz tone survives almost untouched. Plain averaging is a low-pass filter.

### Part 3 – Gain and phase at every frequency

![Frequency response of the 8-tap averager](figures/02_freq_response.png)

- **Notches at 1000, 2000 and 3000 Hz.** At these frequencies, 8 samples hold an exact whole number of periods (one period of 1000 Hz is 8 samples at 8 kHz). The positive and negative halves cancel in the average, so the output is zero. That's why the 2000 Hz tone in Part 2 vanished.
- **Straight-line phase.** The phase falls steadily with frequency, which means a constant time delay of 3.5 samples: 360° × 1000 × 3.5 / 8000 = 157.5° at 1000 Hz. The +180° jumps happen at the notches, where the output changes sign.

### My predictions vs results

1. *Weights for anti-noise that is flipped and 3 samples late:* `[0, 0, 0, -1]`.
2. *Where are the notches?* At multiples of fs / 8 = 1000 Hz.

**Takeaway:** one FIR filter can delay, flip and reshape a signal using nothing but a list of weights. In ANC the right weights are unknown and keep changing, so the next step is an algorithm that learns them: LMS.

---

## Repo layout

```
notebooks/   one notebook per step
figures/     plots from the notebooks, used in this README
LOG.md       learning log: what I tried, what surprised me
```

## Running it

```bash
pip install -r requirements.txt
jupyter notebook
```

Or open any notebook in Google Colab.
