# Forgemaster's Folly - Writeup

**Category:** Crypto
**Points:** 500
**Flag:** `esch{c0pp3r5m1th_l4tt1c3_4tt4ck_ftw}`

---

## The Challenge

We're presented with an RSA encryption challenge, but with an unusual twist - we're given some extra information:

```
N = 0x60ed7e156891376276774d4da4b01b23...  (2048-bit modulus)
e = 65537
A = 0xa37924228b4cda400c4758954a778072...  (768-bit value)
k = 256
c = 0x2ff1da2da238ce79ae6c76224c53ba4b...  (ciphertext)
```

The hint mentions that one prime was "drawn from the void" (random) while the other was "shaped with care, built upon an ancient foundation" (has structure).

---

## Understanding the Vulnerability

The extra parameters `A` and `k` are the key. They're telling us that one of the primes has the form:

```
p = A × 2^k + x
```

Where:
- `A` is the "ancient foundation" (768 bits)
- `k = 256` is a shift amount
- `x` is a small unknown value (less than 2^256)

This is a **partial information attack** - we know most of p, but not the lower 256 bits.

---

## Coppersmith's Attack

When you know part of a prime factor, you can use **Coppersmith's small roots method** to recover the rest. This is a lattice-based attack that can find small solutions to polynomial equations modulo a factor of N.

Here's how it works:
1. We construct a polynomial `f(x) = A × 2^k + x`
2. We know that `f(x) ≡ 0 (mod p)` for some small x
3. Coppersmith's method uses lattice reduction (LLL algorithm) to find this x

The attack works when:
- x is small enough (bounded by `X = 2^k`)
- p is a large enough factor of N (at least N^0.5)

Both conditions are met here since p is roughly half of N (1024 bits out of 2048).

---

## The Solve Script

Using SageMath for the lattice computations:

```python
# solve.sage
N = 0x60ed7e156891376276774d4da4b01b23...
e = 65537
A = 0xa37924228b4cda400c4758954a778072...
k = 256
c = 0x2ff1da2da238ce79ae6c76224c53ba4b...

# Define polynomial ring over Z/NZ
P.<x> = PolynomialRing(Zmod(N))
f = A * 2^k + x

# Find small roots using Coppersmith's method
# X = upper bound on x, beta = p/N ratio (0.5 = sqrt)
roots = f.small_roots(X=2^k, beta=0.5, epsilon=0.02)

if roots:
    x_val = int(roots[0])
    p = A * 2^k + x_val
    q = N // p

    # Standard RSA decryption
    phi = (p-1) * (q-1)
    d = inverse_mod(e, phi)
    m = power_mod(c, d, N)

    print(int(m).to_bytes((int(m).bit_length()+7)//8, 'big').decode())
```

Running this (via Docker since SageMath is heavy):
```bash
docker run --rm -v "$(pwd):/work" -w /work sagemath/sagemath sage solve.sage
```

**Result:**
```
Roots: [841317791189840027537221288996547890667351175402245006728795]
Flag: esch{c0pp3r5m1th_l4tt1c3_4tt4ck_ftw}
```

---

## What We Recovered

The small root x turned out to be about 240 bits - well within the 256-bit bound:
```
x = 841317791189840027537221288996547890667351175402245006728795
```

This gave us the full factorization:
```
p = A × 2^256 + x = 114794790266444972259100125680416229025809529758151726314071...
q = N / p = 106590108940469096672319695392793196057776887859464411036237...
```

---

## The Vulnerability Explained

The challenge author generated RSA primes like this (approximately):

```python
# BAD: Structured prime generation
A = random_768_bit_number()
x = random_256_bit_number()
p = A * 2^256 + x  # Vulnerable!
q = random_1024_bit_prime()  # This one is fine
```

The "ancient foundation" A leaked too much information about p. A properly generated prime should be completely random, with no known structure.

---

## Why This Matters

This attack was discovered by Don Coppersmith in 1996 and is still relevant today. It shows that:

1. **Partial key exposure is dangerous** - knowing ~75% of a prime is enough to recover the rest
2. **Structure in primes is bad** - primes should be uniformly random
3. **Lattice attacks are powerful** - the LLL algorithm can find "small" solutions to polynomial equations

---

## TL;DR

1. Given: RSA with extra info A and k
2. Recognize: p = A × 2^k + x structure
3. Apply: Coppersmith's small roots attack
4. Solve: Use SageMath's `small_roots()` function
5. Win: Decrypt with recovered p and q

---

*Flag: `esch{c0pp3r5m1th_l4tt1c3_4tt4ck_ftw}`*
