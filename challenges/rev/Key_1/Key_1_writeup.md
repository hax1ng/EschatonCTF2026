# Key 1 - Writeup

**Category:** Reverse Engineering
**Points:** 450
**Flag:** `esch{destructors-clean-up-lunar-nova-2168}`

## The Challenge

We're given a "legacy license validator" from KeyCorp. The goal? Reverse engineer how it validates license keys and build a keygen that can generate valid keys for any username. The server will test us with 5 random usernames - nail all 5 and we get the flag.

## First Look

We get a binary called `validator`. Running it shows basic usage:

```
./validator <username> <license_key>
```

It either prints "Valid!" or "Invalid!" - classic crackme style.

## Diving Into the Binary

I loaded it up in Binary Ninja and started poking around. The `main` function is dead simple:

```c
if (argc == 3) {
    if (proc_a(argv[1], argv[2]) != 0)
        puts("Valid!");
    else
        puts("Invalid!");
}
```

So all the magic happens in `proc_a`. Let's see what it does.

## The Validation Logic

Breaking down `proc_a`, there are a few checks:

### 1. Username Rules
- Must be 4-16 characters long
- Only alphanumeric characters and underscores allowed

### 2. Key Format
- Must be exactly 19 characters
- Format: `XXXX-XXXX-XXXX-XXXX` (dashes at positions 4, 9, and 14)
- Each group must be valid hex digits (0-9, A-F)

### 3. The Actual Validation

This is where it gets interesting. The binary:

1. **Hashes the username** using a polynomial rolling hash
2. **Derives 4 values** from that hash
3. **Compares them** against the 4 hex groups in your key

If they match, you're in!

## Cracking the Algorithm

### The Hash Function

Looking at the disassembly, the hash is pretty standard:

```python
h = 0x7a2f  # Starting seed
for c in username:
    h = (h * 33 + ord(c)) & 0xFFFFFFFF
```

This is basically the djb2 hash variant - multiply by 33 (same as `h + (h << 5)`) and add the character.

### Generating the Key Parts

Here's where I had to be careful. The assembly does some funky stuff with register sizes:

```asm
lea     edx, [rax*8]        ; edx = hash << 3
mov     esi, eax            ; esi = hash
shr     ax, 0x5             ; ONLY shifts lower 16 bits!
xor     eax, edx            ; XOR full 32-bit values
xor     si, 0x9c3e          ; XOR lower 16 bits
xor     ax, 0xb7a1          ; Another XOR
```

The tricky part is that `shr ax, 5` only affects the lower 16 bits of the register while preserving the upper bits. This is a classic gotcha when reversing x86.

After working through all the math, here's how the 4 key components are calculated:

```python
# si - first group
si = (hash ^ 0x9c3e) & 0xFFFF

# ax - second group (the tricky one!)
eax_shifted = (hash & 0xFFFF0000) | ((hash & 0xFFFF) >> 5)
ax = ((eax_shifted ^ (hash << 3)) ^ 0xb7a1) & 0xFFFF

# dx - third group
sum_val = (si + ax) & 0xFFFF
dx = (sum_val ^ 0xe4d2) & 0xFFFF

# cx - fourth group
cx = ((ax ^ sum_val) ^ 0x78ec) & 0xFFFF
```

## Building the Keygen

With the algorithm figured out, the keygen is straightforward:

```python
def keygen(username):
    # Hash the username
    h = 0x7a2f
    for c in username:
        h = (h + (h << 5) + ord(c)) & 0xFFFFFFFF

    # Calculate key components
    si = (h ^ 0x9c3e) & 0xFFFF

    eax_after_shr = (h & 0xFFFF0000) | ((h & 0xFFFF) >> 5)
    edx = (h << 3) & 0xFFFFFFFF
    ax = ((eax_after_shr ^ edx) ^ 0xb7a1) & 0xFFFF

    sum_val = (si + ax) & 0xFFFF
    dx = (sum_val ^ 0xe4d2) & 0xFFFF
    cx = ((ax ^ sum_val) ^ 0x78ec) & 0xFFFF

    return f"{si:04X}-{ax:04X}-{dx:04X}-{cx:04X}"
```

## Testing Locally

Quick sanity check against the binary:

```
$ ./validator test "$(python3 -c 'from keygen import keygen; print(keygen("test"))')"
Valid!

$ ./validator admin "$(python3 -c 'from keygen import keygen; print(keygen("admin"))')"
Valid!
```

## Getting the Flag

Connected to the server and let the keygen do its thing:

```
╔══════════════════════════════════════╗
║         key-1 License Server         ║
╚══════════════════════════════════════╝

Challenge 1/5:
Username: theta_operator
Enter key: 636E-4ADB-4A9B-9C7E
[+] Valid!

Challenge 2/5:
Username: pi_developer
Enter key: 1233-C3B9-313E-6EB9
[+] Valid!

...

Challenge 5/5:
Username: test_account
Enter key: 1885-965C-4A33-4051
[+] Valid!

╔══════════════════════════════════════╗
║        Congratulations!              ║
╠══════════════════════════════════════╣
║  esch{destructors-clean-up-lunar-nova-2168}
╚══════════════════════════════════════╝
```

## Lessons Learned

1. **Watch your register sizes** - x86 has 8/16/32/64-bit variants of the same register (al/ax/eax/rax). Operations on smaller sizes can preserve or zero the upper bits differently.

2. **Verify with the binary** - Always test your keygen against the actual binary before hitting the server. I caught my bug this way.

3. **Polynomial hashes are everywhere** - djb2 and its variants show up constantly in CTFs and real software.

## Files

- `keygen.py` - The keygen script
- `solve.py` - Automated solver for the server
- `validator` - The original binary
