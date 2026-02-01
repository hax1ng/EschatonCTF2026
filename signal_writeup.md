# Signal - Hardware Challenge Writeup

**Category:** Hardware/RF
**Author:** @solvz

## Challenge Description

> Our ground station captured a transmission from a Low Earth Orbit (LEO) amateur satellite during its 10-minute overhead pass.
>
> Your mission is to analyze the signal, and decode the hidden message.

We're given a file: `satellite_signal.cf32`

## TL;DR

The signal is a **QPSK-modulated** satellite beacon at **4800 baud**. After correcting for Doppler shift and demodulating, we recover a repeating message containing the flag.

**Flag:** `esch{d0ppl3r_efffffect_0n_orb1t}`

---

## Step 1: Understanding the File Format

The `.cf32` extension tells us this is a **complex float32** IQ sample file - a standard format used in Software Defined Radio (SDR). Each sample consists of two 32-bit floats representing the In-phase (I) and Quadrature (Q) components of the signal.

```python
import numpy as np
data = np.fromfile('satellite_signal.cf32', dtype=np.complex64)
print(f'Samples: {len(data):,}')  # 28,800,000 samples
```

At 48,000 Hz sample rate, this gives us exactly **600 seconds (10 minutes)** of data - matching the "10-minute overhead pass" mentioned in the challenge!

## Step 2: Identifying the Doppler Shift

LEO satellites move fast - like, really fast. As the satellite approaches, the signal frequency shifts higher (like a car horn approaching). As it recedes, it shifts lower. This is the **Doppler effect**.

By looking at the instantaneous frequency over time, we can see this classic pattern:

```
t=  0s: +3269 Hz  (satellite approaching)
t=300s:    +10 Hz  (closest approach - overhead)
t=600s: -3106 Hz  (satellite receding)
```

The frequency swings from about +3.3 kHz to -3.1 kHz over the pass. Classic LEO Doppler signature!

## Step 3: Finding the Modulation Type

Initially, I thought this was **FSK** (Frequency Shift Keying) because the FM-demodulated signal showed two clear frequency peaks around ±4000 Hz. But the key insight came from looking at the **phase constellation**.

When I plotted the signal phases after frequency correction, I saw **four distinct clusters** - the telltale sign of **QPSK** (Quadrature Phase Shift Keying).

The strong 4800 Hz component in the spectrum also told us the **baud rate: 4800 symbols/second**.

## Step 4: QPSK Demodulation

QPSK encodes 2 bits per symbol using four phase states (typically at 45°, 135°, 225°, 315°). Here's how I demodulated it:

### 4a. Frequency Correction (Per-Second Windows)

Since the Doppler shift changes continuously, I corrected it in 1-second windows:

```python
for window_start in range(0, len(data), sample_rate):
    chunk = data[window_start:window_start + sample_rate]

    # Estimate frequency offset from phase slope
    phase = np.unwrap(np.angle(chunk))
    coeffs = np.polyfit(np.arange(len(phase)), phase, 1)
    freq_offset = coeffs[0] * sample_rate / (2 * np.pi)

    # Apply correction
    t = np.arange(len(chunk)) / sample_rate
    correction = np.exp(-1j * 2 * np.pi * freq_offset * t)
    chunk_corrected = chunk * correction
```

### 4b. Symbol Sampling

With 48000 Hz sample rate and 4800 baud, we get **10 samples per symbol**. I sampled at the middle of each symbol period:

```python
samples_per_symbol = sample_rate // baud_rate  # 10
for i in range(0, len(chunk), samples_per_symbol):
    mid = i + samples_per_symbol // 2
    symbols.append(chunk_corrected[mid])
```

### 4c. Phase-to-Dibit Mapping

After trying several mappings, the one that worked was:

| Quadrant | Phase Range | Dibit |
|----------|-------------|-------|
| Q1 | 0° - 90° | 11 |
| Q2 | 90° - 180° | 10 |
| Q3 | 180° - 270° | 00 |
| Q4 | 270° - 360° | 01 |

```python
mapping = [0b11, 0b10, 0b00, 0b01]  # For quadrants 0,1,2,3

for sym in symbols:
    phase = np.angle(sym)
    deg = np.degrees(phase) % 360
    quadrant = int(deg // 90)
    dibits.append(mapping[quadrant])
```

### 4d. Dibits to Bytes

Four dibits make one byte (MSB first):

```python
for i in range(0, len(dibits) - 3, 4):
    byte = (dibits[i] << 6) | (dibits[i+1] << 4) | (dibits[i+2] << 2) | dibits[i+3]
    bytes_data.append(byte)
```

## Step 5: Finding the Flag

After demodulation, searching the byte stream revealed the flag pattern:

```
...ect_0n_orb1t} | esch{d0ppl3r_efffffect_0n_orb1t} | esch{d0ppl3r_...
```

The satellite beacon was transmitting the message on repeat! The `|` separator shows where one transmission ends and the next begins.

The flag: **`esch{d0ppl3r_efffffect_0n_orb1t}`**

## Why "efffffect"?

You might wonder about the five f's. At first I thought it was a decoding error, but voting across multiple decoded instances confirmed it's intentional. The flag is a play on:

- **d0ppl3r** = Doppler (leetspeak)
- **efffffect** = effect (with extra f's for style)
- **0n** = on
- **orb1t** = orbit (leetspeak)

The theme perfectly matches the challenge - we literally had to deal with Doppler effect caused by the satellite's orbit!

## Full Solution Script

```python
import numpy as np

data = np.fromfile('satellite_signal.cf32', dtype=np.complex64)
sample_rate = 48000
baud_rate = 4800
samples_per_symbol = sample_rate // baud_rate

all_dibits = []

# Process in 1-second windows with Doppler correction
for window_start in range(0, len(data) - sample_rate, sample_rate):
    chunk = data[window_start:window_start + sample_rate]

    # Correct Doppler shift
    phase = np.unwrap(np.angle(chunk))
    coeffs = np.polyfit(np.arange(len(phase)), phase, 1)
    freq_offset = coeffs[0] * sample_rate / (2 * np.pi)
    t = np.arange(len(chunk)) / sample_rate
    correction = np.exp(-1j * 2 * np.pi * freq_offset * t)
    chunk_corrected = chunk * correction

    # QPSK demodulation
    mapping = [0b11, 0b10, 0b00, 0b01]
    for i in range(0, len(chunk_corrected) - samples_per_symbol, samples_per_symbol):
        mid = i + samples_per_symbol // 2
        sym = chunk_corrected[mid]
        phase = np.angle(sym)
        deg = np.degrees(phase) % 360
        quadrant = int(deg // 90)
        all_dibits.append(mapping[quadrant])

# Convert to bytes
bytes_data = bytearray()
for i in range(0, len(all_dibits) - 3, 4):
    byte = (all_dibits[i] << 6) | (all_dibits[i+1] << 4) | (all_dibits[i+2] << 2) | all_dibits[i+3]
    bytes_data.append(byte)

# Find flag
pos = bytes_data.find(b'esch{')
if pos != -1:
    # Find closing brace
    end = bytes_data.find(b'}', pos)
    flag = bytes_data[pos:end+1]
    print(f"Flag: {flag.decode()}")
```

## Lessons Learned

1. **LEO satellites = Doppler shift** - Any real satellite signal will have significant frequency drift that needs correction
2. **Sample rate matters** - The 48 kHz rate and 10-minute duration were big hints
3. **QPSK vs FSK** - Don't assume! Check the constellation
4. **Windowed processing** - For time-varying effects, process in small windows
5. **The flag is always thematic** - "Doppler effect on orbit" was hiding in plain sight

## Flag

```
esch{d0ppl3r_efffffect_0n_orb1t}
```
