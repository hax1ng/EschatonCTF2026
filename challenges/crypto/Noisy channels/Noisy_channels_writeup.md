# Noisy Channels — Writeup

**CTF:** Eschaton CTF 2026
**Category:** Crypto
**Flag:** `esch{t1m1ng_0r4cl3s_4r3_d4ng3r0us}`

---

## What We're Given

A single file: `intercept.json` (~787KB). Cracking it open we find:

| Field | What it is |
|---|---|
| `modulus` | `12289` — a prime number |
| `dimension` | `200` |
| `samples` | `320` |
| `public_matrix` | A big 320×200 matrix of numbers |
| `ciphertext` | 320 numbers |
| `timing_ns` | 320 timing measurements (nanoseconds) |
| `power_trace` | 320 power consumption readings |
| `encrypted_flag` | 34 bytes — the prize |

## So What Is This?

This is a textbook **Learning With Errors (LWE)** setup — a cornerstone of post-quantum cryptography. The idea is dead simple:

> Pick a secret vector **s**. For each sample, compute **A·s**, then add a tiny random error **e**. Publish **A** and the noisy results **b**. Good luck getting **s** back.

In math terms:

```
b = A·s + e   (mod 12289)
```

The matrix **A** and the vector **b** are public. The secret **s** and the errors **e** are hidden. Normally, recovering **s** from this is *hard* — like, "we're betting the future of encryption on it" hard. The small errors are what make it impossible to just solve the linear system directly.

But... the challenge is called **Noisy Channels**, and we got some extra goodies: `timing_ns` and `power_trace`. That's a hint that we're supposed to use **side-channel attacks**.

## The Side-Channel Leak

Side-channel attacks exploit physical information that leaks out of a system — how long an operation takes, how much power it draws, etc. Here, each of the 320 samples came with a timing value and a power reading.

Plotting out the distributions, two obvious clusters jump out:

**Timing:**
- ~243 samples clustered around **1197–1203 ns** (fast)
- ~77 samples clustered around **1247–1253 ns** (slow)

**Power:**
- Most fast samples had power **94–106** (low)
- The slow samples had power **119–131** (medium)
- A handful (~16) of fast samples had weirdly high power **144–156** (outliers)

This screams "the side-channel is leaking info about the error term." Bigger errors = more computation = more time and power.

## The Attack Plan

If we can figure out which samples have **error = 0**, then for those samples we get:

```
b = A·s + 0   (mod 12289)
b = A·s        (mod 12289)
```

That's just a plain linear system with no noise! And if we have at least 200 such samples (matching our dimension), we can solve it directly.

**Classification:**
- **Fast timing + low power** → probably `e = 0` → **227 samples**
- Everything else → nonzero error

227 > 200. We're in business.

## Solving It

Using SageMath, we:

1. Grabbed the 227 "clean" rows from A and b
2. Built the system over GF(12289) (a finite field)
3. Checked the rank — full rank 200, so the system has a unique solution
4. Solved `A_clean · s = b_clean`

To verify we got the right answer, we computed the errors for *all* 320 samples:

| Error | Count |
|---|---|
| -2 | 9 |
| -1 | 41 |
| **0** | **227** |
| +1 | 36 |
| +2 | 7 |

All 227 "clean" samples truly had zero error. The side-channel classification was perfect.

## Decrypting the Flag

With the secret **s** recovered, we needed to figure out how the 34-byte `encrypted_flag` was encrypted. After trying a few methods, the one that worked:

1. Convert **s** to a comma-separated string: `"7587,2064,4753,..."`
2. SHA256 hash that string to get a 32-byte key
3. XOR the key (cycling) against `encrypted_flag`

Out pops:

```
esch{t1m1ng_0r4cl3s_4r3_d4ng3r0us}
```

Timing oracles *are* dangerous indeed.

## Solve Script

```python
# solve.sage — run with: sage solve.sage
import json
import hashlib

with open('intercept.json') as f:
    data = json.load(f)

q = data['modulus']
n = data['dimension']
A = data['public_matrix']
b = data['ciphertext']
timing = data['timing_ns']
power = data['power_trace']
enc_flag = data['encrypted_flag']

# Side-channel filter: fast timing + low power = zero error
clean = [i for i in range(len(b)) if timing[i] <= 1203 and power[i] <= 106]

# Solve noiseless linear system mod q
F = GF(q)
A_clean = matrix(F, [A[i] for i in clean])
b_clean = vector(F, [b[i] for i in clean])
s = A_clean.solve_right(b_clean)

# Decrypt flag
s_int = [int(x) for x in s]
key = hashlib.sha256(','.join(str(x) for x in s_int).encode()).digest()
flag = bytes([enc_flag[i] ^^ key[i % len(key)] for i in range(len(enc_flag))])
print(flag.decode())
```

## TL;DR

LWE is supposed to be hard because of the random noise. But if a system leaks *which* samples are noisy (via timing and power side-channels), you can just throw away the noisy ones and solve a clean linear system. Game over.
