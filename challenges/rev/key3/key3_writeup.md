# Key 3 - Writeup

**Category:** Reversing | **Points:** 480 | **Author:** @solvz

> EnterpriseCore's premium licensing system uses a complex method for key generation. Reverse engineer the binary to build a keygen that works for any parameters.

**Flag:** `esch{conditions-drive-paths-shadow-shadow-8997}`

---

## TL;DR

Stripped Rust binary that validates license keys. The key derivation is: SHA-256 a blob of inputs, pull four 32-bit words out of the hash (little-endian!), CRC32 the hardware ID, then run everything through an 8-round Feistel-like mixing network. One endianness gotcha is the entire challenge.

---

## Recon

We get a single binary (`validator`) and a remote service that asks us to generate 5 valid keys in a row.

```
$ file validator
validator: ELF 64-bit LSB pie executable, x86-64, dynamically linked, stripped

$ ./validator
Usage: ./validator <username> <hwid> <timestamp> <tier> <key>
```

It's stripped, PIE, and compiled from Rust (visible in the embedded strings and panic paths referencing `src/validator.rs`). Running it with bad input gives `Invalid!`, and presumably a correct key gives `Valid!`.

The key format is visible in the strings:

```
KGIII-XXXXXXXX-XXXXXXXX-XXXXXXXX-XXXXXXXX-YYYY
```

Where `YYYY` is a tier suffix: `BRNZ` (tier 1), `SLVR` (tier 2), or `GOLD` (tier 3).

## Reversing the Validator

### Input Validation

Threw the binary into Binary Ninja and started decompiling. The main validation function lives at offset `0x402480`. It does a bunch of sanity checks first:

- **Username:** 4-32 printable ASCII characters
- **HWID:** Exactly 16 hex characters (uppercased internally, then parsed to 8 raw bytes)
- **Timestamp:** Must be within 86400 seconds (~24 hours) of the current system time
- **Tier:** 1, 2, or 3
- **Key:** Exactly 46 characters, must start with `KGIII-` and end with the right tier suffix

The key's four hex groups (8 chars each) are parsed into four 32-bit integers. These are the values the binary recomputes and checks against.

### The Hash

The binary builds a buffer and SHA-256 hashes it:

```
SHA-256( username_bytes || hwid_parsed_bytes || le64(timestamp) || u8(tier) )
```

Where `hwid_parsed_bytes` is the 8 raw bytes from parsing the hex string (not the ASCII hex string itself), and the timestamp is packed as a little-endian 64-bit integer.

From the first 16 bytes of the SHA-256 output, four 32-bit words are extracted:

```
h0 = sha256[0:4]  as little-endian u32
h1 = sha256[4:8]  as little-endian u32
h2 = sha256[8:12] as little-endian u32
h3 = sha256[12:16] as little-endian u32
```

### The CRC

A CRC32 (IEEE 802.3 polynomial) is computed over the 8 parsed HWID bytes. The binary uses `PCLMULQDQ`-accelerated CRC on hardware that supports it. Standard CRC32 result, nothing unusual here.

### The 8-Round Mixing Network

This is where the bulk of the work is. It's a Feistel-like structure that mixes the hash words and CRC through 8 rounds using:
- `rol32` (rotate left by N bits)
- XOR with a magic constant `0x5f8c2e7a`
- Additions and subtractions mod 2^32

Here's the full algorithm, round by round:

```python
MAGIC = 0x5f8c2e7a

# Setup
a = h0 ^ c            # c = CRC32 result
b = h1 + a

# Round 1
b4 = rol32(b, 4) ^ h2
x1 = b4 ^ MAGIC
c3 = rol32(c, 3)
y1 = x1 ^ c3
d  = ((h3 - x1) ^ a) + y1
d4 = rol32(d, 4) ^ a

# Round 2
x2 = d4 ^ MAGIC
c6 = rol32(c, 6)
y2 = x2 ^ c6
e  = ((rol32(b, 25) - x2) ^ y1) + y2

# Round 3
f1 = c3 ^ b4 ^ rol32(e, 4)
c9 = rol32(c, 9)
g1 = c9 ^ f1
h_val = ((rol32(d, 25) - f1) ^ y2) + g1

# Round 4
f2 = c6 ^ d4 ^ rol32(h_val, 4)
c12 = rol32(c, 12)
g2 = c12 ^ f2
j  = ((rol32(e, 25) - f2) ^ g1) + g2

# Round 5
j4 = rol32(j, 4) ^ g1
x5 = j4 ^ MAGIC
c15 = rol32(c, 15)
y5 = x5 ^ c15
k  = ((rol32(h_val, 25) - x5) ^ g2) + y5

# Round 6
k4 = rol32(k, 4) ^ g2
x6 = k4 ^ MAGIC
c18 = rol32(c, 18)
y6 = x6 ^ c18
m  = ((rol32(j, 25) - x6) ^ y5) + y6

# Round 7
f7 = c15 ^ j4 ^ rol32(m, 4)
c21 = rol32(c, 21)
n  = c21 ^ f7
p  = ((rol32(k, 25) - f7) ^ y6) + n

# Round 8
r_val = c18 ^ k4 ^ rol32(p, 4)

# Final key parts
key1 = r_val
key2 = (rol32(m, 25) - r_val) ^ n
key3 = n
key4 = rol32(p, 25)
```

The CRC value `c` gets rotated by incrementing multiples of 3 (3, 6, 9, 12, 15, 18, 21) throughout the rounds, acting as a round-dependent "tweak." The magic constant `0x5f8c2e7a` gets XORed in at specific points too. Each round feeds forward into the next, making the whole thing one big dependency chain.

## The Bug That Wasted My Time

I got the entire algorithm right on the first try... except for one thing. My initial keygen read the SHA-256 hash words as **big-endian**:

```python
h0 = struct.unpack('>I', sha256_hash[0:4])[0]  # WRONG
```

But the Rust binary reads them as **little-endian** (native x86 byte order):

```python
h0 = struct.unpack('<I', sha256_hash[0:4])[0]  # CORRECT
```

I caught this by setting GDB breakpoints right before the CRC32 call and comparing register values against my Python output:

| Word | Python (big-endian) | Binary (little-endian) |
|------|--------------------|-----------------------|
| h0   | `0x042defc2`       | `0xc2ef2d04`          |
| h1   | `0x678c5719`       | `0x19578c67`          |
| h2   | `0x36799186`       | `0x86917936`          |
| h3   | `0x9c202f2b`       | `0x2b2f209c`          |

Same bytes, opposite read order. The CRC32 result matched perfectly (0x9fb3d300 on both sides), which confirmed the HWID parsing was fine. Pure endianness issue.

Swapping `'>I'` to `'<I'` and suddenly:

```
$ ./validator testuser ABCDEF0123456789 1769889821 1 KGIII-85C83906-1118F893-E5166DD4-6E96FA6C-BRNZ
Valid!
```

## Getting the Flag

The remote service presents 5 challenges, each with a username, HWID, timestamp, and tier. You have to respond with a valid key for each one.

```
╔══════════════════════════════════════╗
║     key-3 Enterprise Licensing       ║
╠══════════════════════════════════════╣
║  Generate 5 valid keys to proceed   ║
╚══════════════════════════════════════╝

Challenge 1/5:
Username: subscriber_silver
HWID: DEAD5678FACE1234
Timestamp: 1769889863
Tier: 2 (SLVR)
Enter key:
```

Wrote a quick socket script that parses each challenge, runs the keygen, and sends the key back. All 5 accepted:

```
[+] Valid!
[+] Valid!
[+] Valid!
[+] Valid!
[+] Valid!

╔══════════════════════════════════════╗
║        Congratulations!              ║
╠══════════════════════════════════════╣
║  esch{conditions-drive-paths-shadow-shadow-8997}
╚══════════════════════════════════════╝
```

## Tools Used

- **Binary Ninja** - decompilation and disassembly of the stripped Rust binary
- **GDB** - tracing intermediate values to find the endianness bug
- **Python 3** - keygen + socket solve script

## Keygen

Full keygen is in [`keygen.py`](keygen.py). Usage:

```bash
python3 keygen.py <username> <hwid> <timestamp> <tier>
```

Example:

```
$ python3 keygen.py testuser ABCDEF0123456789 1769889821 1
Username:  testuser
HWID:      ABCDEF0123456789
Timestamp: 1769889821
Tier:      1
Key:       KGIII-85C83906-1118F893-E5166DD4-6E96FA6C-BRNZ
```
