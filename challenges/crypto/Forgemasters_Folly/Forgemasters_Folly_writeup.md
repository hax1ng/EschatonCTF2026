# Forgemaster's Folly Writeup

**Category:** Crypto
**Points:** 380
**Author:** @myst
**Flag:** `esch{primitives-compose-poorly-nova-ember-6932}`

---

## The Challenge

We're given an HTTP endpoint that serves up some cryptographic "artifacts" with a fantasy blacksmith theme. The story talks about a master craftsman who "shaped one number with care, built upon an ancient foundation" - classic CTF flavor text hinting that something about the key generation is flawed.

Here's what we get:

```
N = [2048-bit RSA modulus]
e = 65537
A = [768-bit number]
k = 256
c = [ciphertext]
```

The tagline "Structure breeds weakness" is the real hint here.

---

## What's Going On?

This is RSA, but with a twist. Normally in RSA:
- You pick two large random primes `p` and `q`
- Multiply them to get `N = p * q`
- The security relies on factoring `N` being hard

But here, the challenge is leaking something extra: `A` and `k`.

Looking at the numbers:
- `N` is 2048 bits (standard RSA modulus size)
- `A` is 768 bits
- `k` is 256

If you do the math: 768 + 256 = 1024 bits. That's exactly half of N, which is suspiciously close to the size of one prime factor.

**The vulnerability:** One of the primes was generated as:

```
p = A * 2^k + x
```

Where:
- `A` = the known 768-bit "high bits" of p
- `2^k` = shift A left by 256 bits
- `x` = unknown random 256-bit "low bits"

So we know most of `p`, just not the bottom 256 bits. That's only 25% of the prime that's unknown!

---

## The Attack: Coppersmith's Method

This is a classic scenario for **Coppersmith's small roots attack**. Here's the intuition:

Imagine you have a polynomial equation and you know there's a "small" solution. Coppersmith figured out that lattice reduction (fancy linear algebra) can find these small solutions efficiently, even when working modulo some unknown factor of a big number.

In our case:
- We want to solve `f(x) = A * 2^256 + x ≡ 0 (mod p)`
- We don't know `p`, but we know `p` divides `N`
- The solution `x` is "small" (only 256 bits compared to the 1024-bit prime)

SageMath has this attack built-in as `.small_roots()` on polynomials.

---

## The Solve

First, I tried the obvious parameters but got no results:

```python
f = A * 2^k + x
roots = f.small_roots(X=2^256, beta=0.5, epsilon=0.02)
# No roots found :(
```

The trick was tweaking `beta` slightly. The `beta` parameter tells the algorithm what fraction of N the factor is (p is about N^0.5). Using `beta=0.49` instead of exactly `0.5` made it work:

```python
roots = f.small_roots(X=2^256, beta=0.49, epsilon=0.05)
# Found it!
```

Once we have `x`, the rest is standard RSA:

```python
# Recover p
x_val = int(roots[0])
p = A * 2^256 + x_val

# Factor N
q = N // p

# Decrypt
phi = (p-1) * (q-1)
d = inverse_mod(e, phi)
m = pow(c, d, N)

# Convert to text
flag = int(m).to_bytes(length, 'big').decode()
```

---

## Full Solution Script

```python
# solve.sage - run with: sage solve.sage

N = 0x9c4df9dabd95f2affcfda98f1eaa166415110a0d8a4fc40607c50b8539bc41bd...
e = 65537
A = 0xb9da3a2f2404e95366c02ada6beae1afc9522679dfa1f2503b5cccd4c6652440...
k = 256
c = 0x49c2760b0d4956c38c1208d1b2fc74b24f675374eb34f41f736b4ac523765d16...

P.<x> = PolynomialRing(Zmod(N))
f = A * 2^k + x

# The magic - beta=0.49 instead of 0.5
roots = f.small_roots(X=2^k, beta=0.49, epsilon=0.05)

if roots:
    x_val = int(roots[0])
    p = A * 2^k + x_val
    q = N // p
    phi = (p-1)*(q-1)
    d = inverse_mod(e, phi)
    m = power_mod(c, d, N)
    m_bytes = int(m).to_bytes((int(m).bit_length()+7)//8, "big")
    print(m_bytes.decode())
```

---

## Lessons Learned

1. **Don't roll your own crypto** - The challenge name "Forgemaster's Folly" hints at this. The "forgemaster" tried to be clever with prime generation and it backfired.

2. **Random means random** - Primes should be fully random. Any structure or pattern (like "high bits are fixed") can be exploited.

3. **Coppersmith is powerful** - If you leak even a portion of a secret value, lattice attacks might recover the rest. This applies to:
   - Partial key exposure
   - Stereotyped messages (known plaintext prefix/suffix)
   - Weak random padding

4. **Parameter tuning matters** - The `beta=0.49` vs `beta=0.5` thing is a good reminder that crypto attacks often need experimentation. The math says it should work at 0.5, but numerical precision and algorithm details mean you sometimes need to fiddle with parameters.

---

## Tools Used

- **SageMath** (via Docker: `sagemath/sagemath`) - For the `.small_roots()` implementation
- **curl** - To grab the challenge data from the server

---

## References

- [Coppersmith's Attack (Wikipedia)](https://en.wikipedia.org/wiki/Coppersmith%27s_attack)
- [Twenty Years of Attacks on RSA](https://crypto.stanford.edu/~dabo/papers/RSA-survey.pdf) - Dan Boneh's classic survey
- [SageMath small_roots documentation](https://doc.sagemath.org/html/en/reference/polynomial_rings/sage/rings/polynomial/polynomial_modn_dense_ntl.html)
