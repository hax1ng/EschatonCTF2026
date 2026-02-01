# Intercepted Transmission - Writeup

**Category:** Crypto
**Points:** 340
**Flag:** `esch{random-oracles-are-idealized-viper-phantom-2876}`

---

## The Challenge

We're given a web service that displays an "intercepted transmission" containing RSA-encrypted data:

```json
{
  "n": "0x84831ce998a8edd479bd67b7cedf327fc737500d680a61f26289e63d19ade46f...",
  "e": 65537,
  "c": "0x197396bf38048be280f4fd817d3a79282469d1af9fcbeb9e0186b52ee7575ecd..."
}
```

The hint says the encryption was "implemented hastily" - so there's a weakness somewhere.

---

## Quick RSA Refresher

RSA encryption works like this:
- Pick two large prime numbers, **p** and **q**
- Multiply them to get **n = p × q** (the public modulus)
- Pick a public exponent **e** (usually 65537)
- The private key **d** is calculated from p, q, and e
- Encrypt: `ciphertext = message^e mod n`
- Decrypt: `message = ciphertext^d mod n`

The security of RSA depends on the fact that factoring **n** back into **p** and **q** is really hard... usually.

---

## Finding the Weakness

The modulus n is 2048 bits - that's normally way too big to factor. But the hint about "hasty implementation" made me think: what if p and q were generated poorly?

One classic mistake is choosing p and q that are too close together. If they're close, there's an attack called **Fermat Factorization** that can break them.

### How Fermat Factorization Works

If p and q are close, then n = p × q is close to a perfect square. We can write:
- n = a² - b² = (a-b)(a+b)
- So p = a-b and q = a+b

We just need to find `a` such that `a² - n` is a perfect square!

```python
def fermat_factor(n):
    a = isqrt(n)  # Start near √n
    while True:
        a += 1
        b2 = a*a - n
        b = isqrt(b2)
        if b*b == b2:  # Found it!
            return a-b, a+b
```

---

## The Attack

I ran Fermat factorization on n and it worked almost instantly:

```
p = 12933719672094134598842467489461187573189312968535843701336354723052823280222...
q = 12933719672094134598842467489461187573189312968535843701336354723052823280222...
```

Look at those numbers - they're almost identical! The difference between p and q is only about 300,000. For 1024-bit primes, that's incredibly close - like picking two "random" numbers between 1 and a trillion and getting numbers that differ by 0.0000003.

---

## Decrypting the Message

With p and q in hand, decryption is straightforward:

```python
# Calculate phi(n) = (p-1)(q-1)
phi = (p - 1) * (q - 1)

# Calculate private exponent: d = e^(-1) mod phi
d = pow(e, -1, phi)

# Decrypt: message = ciphertext^d mod n
message = pow(c, d, n)

# Convert to text
plaintext = message.to_bytes(...).decode('utf-8')
```

**Result:**
```
esch{random-oracles-are-idealized-viper-phantom-2876}
```

---

## The Vulnerability Explained

When generating RSA keys, you need two large random primes. The key word is **random**.

This implementation probably did something like:
```python
p = next_prime(random_1024_bit_number())
q = next_prime(p)  # WRONG! Just gets the next prime after p
```

Instead of generating two independent random primes, they generated one and then found the next prime after it. This makes p and q extremely close, breaking RSA's security.

**The fix:** Generate p and q independently, and verify that |p - q| is large (at least 2^(n/2 - 100) for an n-bit modulus).

---

## Solve Script

```python
import math

n = 0x84831ce998a8edd479bd67b7cedf327fc737500d680a61f26289e63d19ade46f3e8438bae1c9cb07d5509f41af2fb4fbbbb062b766f81b615845e2fb84aed2f912accf649cbe22d9f1a612125e289f1b135218b42a6b1698f900d047f4bc728ab42bb0f3ccaba63fe9dfc2b58f213763121ece984851d27d616d05b566e8362e2017f3a3e69d9da58914c6cdff92511e5479ba97319facfd3e48e8473c38624b5eb111bf90a31bba0bf1534f341b9cd5c87d8d3b0707a7da31ae60f3bd2e1e44010ab4da3d2155a0e69aa5419b41507522054daa882f6c9ad4ba64e7431ec5ea2cc5b9a1a8d88269b36d85a1ef017ff19a773b8052df1d9404d66247fd433057
e = 65537
c = 0x197396bf38048be280f4fd817d3a79282469d1af9fcbeb9e0186b52ee7575ecdbad49c098282aabeb13c7a47b4df5ac1bb1469b9fa89add7f1a81fe11dff2484e8a0d9bdb12c64d74124c64e65c8a70aa06253f4c7b897e24cffc616007cd7356a33a100a4795a24f253a14254d30eb3662bf88abd090ea6c12dd494008a60942b5e6a62b0a3667b646ad6d6d0de361b59df307427458aa88b8ce8e7946feb7aa963bb498a4de306d6611e6775f258d298ad3c6c2b967e8f18d8d98a59cc89bbf24c16678a1f62f9a5fcc558b73590c486452e87365791919a58efd659999e0bce8df607a073cd653a223996038ec19442985b066586405623d07bb24926a67f

# Fermat factorization
def fermat_factor(n):
    a = math.isqrt(n)
    while True:
        a += 1
        b2 = a*a - n
        b = math.isqrt(b2)
        if b*b == b2:
            return a-b, a+b

p, q = fermat_factor(n)
phi = (p-1) * (q-1)
d = pow(e, -1, phi)
m = pow(c, d, n)
print(m.to_bytes((m.bit_length()+7)//8, 'big').decode())
```

---

## TL;DR

1. RSA with 2048-bit modulus - normally secure
2. But p and q were generated too close together
3. Fermat factorization cracks it in milliseconds
4. Calculate private key, decrypt, get flag

---

*Flag: `esch{random-oracles-are-idealized-viper-phantom-2876}`*
