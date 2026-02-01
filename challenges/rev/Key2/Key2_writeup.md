# Key 2 - Writeup

**Category:** Reverse Engineering
**Author:** @solvz

## The Challenge

We're given a "GameForge activation system" - basically a license key validator. The goal is to reverse engineer how valid keys are generated so we can build a keygen that produces valid keys for any username and hardware ID combo.

## First Look

We get a single file called `validator` - a stripped, statically linked Linux binary. Running it shows us the expected format:

```
Usage: ./validator <username> <hwid> <key>
  username: 4-20 alphanumeric characters
  hwid: 8-character hex string
  key: A1B2-XXXX-XXXX-XXXX-CCCC format
```

So we need to figure out how those `XXXX` parts and the `CCCC` checksum are calculated from the username and hardware ID.

## Finding the Validation Logic

I loaded the binary in Binary Ninja and searched for interesting strings. Found "Valid!" and "Invalid!" which led me straight to the main validation function at `sub_404db0`.

The function does a bunch of checks:
1. Verifies the key has dashes at positions 4, 9, 14, and 19
2. Checks that it starts with "A1B2"
3. Parses the HWID as a hex number
4. Does some math magic to validate the key segments

## The Secret Sauce

Here's where it gets interesting. The validator uses a few key pieces:

### The Hash Function

There's a custom hash function that takes a string and produces a 32-bit value. It starts with `0x4e7f2a19` and for each byte, it does this wild dance of bit rotations and XORs:

```python
def custom_hash(data):
    result = 0x4e7f2a19
    for i, byte in enumerate(data):
        shift = (i & 3) * 8
        xored = ((byte << shift) ^ result)
        rotated = rol(xored, 5) + 0x3c91e6b7
        result = ror(rotated, 11) ^ rotated
    return result
```

### The Transformation Chain

The binary creates 4 objects with virtual function tables (fancy C++ stuff). Each one has a transform function that mangles the input in a specific way:

1. **Transform 1:** `rol(x, 7) ^ 0x8d2f5a1c` - rotate left 7 bits, XOR with magic constant
2. **Transform 2:** Swaps the high and low 16-bit halves, XORing each with different constants (`0x6b3e` and `0x1fa9`)
3. **Transform 3:** `ror(x, 13) + 0x47c83d2e` - rotate right 13 bits, add constant

### Putting It Together

The key generation works like this:

1. Hash the username
2. XOR that hash with the HWID (parsed as hex) - this is your "seed"
3. Run the seed through Transform 1 → take lower 16 bits → first key segment
4. Run that result through Transform 2 → take lower 16 bits → second key segment
5. Run that result through Transform 3 → take lower 16 bits → third key segment
6. Pack all three segments into 6 bytes, hash them, XOR with `0x52b1` → checksum

## The Keygen

Once I understood the algorithm, writing the keygen was straightforward:

```python
def generate_key(username, hwid):
    hwid_val = int(hwid, 16)
    seed = hwid_val ^ custom_hash(username.encode())

    result = transform1(seed)
    key_part0 = result & 0xFFFF

    result = transform2(result)
    key_part1 = result & 0xFFFF

    result = transform3(result)
    key_part2 = result & 0xFFFF

    # Pack parts for checksum
    checksum_buf = pack('<HHH', key_part0, key_part1, key_part2)
    checksum = custom_hash(checksum_buf) ^ 0x52b1

    return f"A1B2-{key_part0:04X}-{key_part1:04X}-{key_part2:04X}-{checksum:04X}"
```

## Getting the Flag

The remote service asks for 5 valid keys with random usernames and HWIDs. I wrote a quick solver using pwntools that:

1. Connects to the server
2. Parses each challenge's username and HWID
3. Generates a valid key
4. Sends it back

```
$ python3 solve.py
[+] Opening connection to node-3.mcsc.space on port 35045: Done
[*] Challenge 1: gamma_streamer / 12345678 -> A1B2-7913-6330-CE99-7C55
[*] Challenge 2: psi_founder / C3C3C3C3 -> A1B2-B75F-AD53-203B-C8EA
[*] Challenge 3: epsilon_pro / 87654321 -> A1B2-BAD9-58A7-CC68-8322
[*] Challenge 4: tau_community / 0F0F0F0F -> A1B2-6515-F294-AE8D-431F
[*] Challenge 5: omicron_artist / FEDCBA98 -> A1B2-F7B8-5921-2160-C9FC

╔══════════════════════════════════════╗
║        Congratulations!              ║
╠══════════════════════════════════════╣
║  esch{loops-hide-secrets-lunar-viper-1048}
╚══════════════════════════════════════╝
```

## Flag

```
esch{loops-hide-secrets-lunar-viper-1048}
```

## Key Takeaways

- Virtual function tables in C++ can be identified by looking for vtable pointers loaded into objects
- Custom hash/crypto algorithms often use bit rotations and magic constants - just trace through them carefully
- When you see a chain of transformations, work through each one step by step
- Binary Ninja's decompiler made this much easier than staring at raw assembly

## Files

- `keygen.py` - Standalone keygen for manual use
- `solve.py` - Automated solver for the CTF challenge
