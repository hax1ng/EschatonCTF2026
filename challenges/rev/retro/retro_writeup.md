# Retro - CTF Writeup

**Category:** Reverse Engineering
**CTF:** EschatonCTF 2026

---

## TL;DR

The flag is: `esch{g4m3b0y_r3vved_634512}`

---

## The Challenge

We're given a Game Boy ROM file called `retro.gb`. The challenge description hints that the game is "broken" and we need to "fix it" to find the flag.

Sounds simple enough, right? Spoiler: it's a trap.

---

## First Look

Opening the ROM in a disassembler (Binary Ninja with a Game Boy loader), we can see this is a legit Game Boy ROM. The header says "RETRO" and there's actual code here, not just garbage.

Poking around the strings, I found some interesting things:
- Text that looks like "THE LEGEND OF ZELDA" (lol, nice theming)
- Some encrypted-looking data around offset 0x508

---

## Finding the "Broken" Part

After tracing through the code, I found a suspicious `CALL` instruction at address `0x1A6`:

```
0x1A6: CD FD 01    ; CALL $01FD
```

Looking at what's at `0x1FD`... it's just `C9` (RET). So this CALL does literally nothing - it immediately returns.

But wait! Right next door at `0x1FE` is the actual function:

```asm
0x1FE: FA 06 C0    ; LD A, [$C006]   - Load the XOR key
       EE FF       ; XOR A, $FF      - Flip all bits (toggle key)
       EA 06 C0    ; LD [$C006], A   - Store it back
       C9          ; RET
```

This is a key toggle function! It's supposed to alternate the XOR key between `0xA5` and `0x5A` for encrypting/decrypting data.

So the "broken" part is that the CALL goes to `0x1FD` (just returns) instead of `0x1FE` (toggles the key).

---

## The Encryption Scheme

The ROM has two encrypted data blocks:
- **Block 1** (13 bytes at 0x508): `ba88b887e5849c829db99096e7`
- **Block 2** (14 bytes at 0x515): `899d9595babbe79e9d9c9f9392e4`

These get XORed with a key stored at memory address `0xC006`, which starts at `0xA5`.

With the "broken" code:
- Block 1 decrypts with key `0xA5`
- Key toggle is SKIPPED (CALL goes to RET)
- Block 2 ALSO decrypts with key `0xA5`

If we "fixed" the code to call `0x1FE`:
- Block 1 decrypts with key `0xA5`
- Key toggles to `0x5A`
- Block 2 decrypts with key `0x5A` (produces garbage!)

---

## The Tile Font Mystery

Game Boy graphics work with tiles - 8x8 pixel images that get placed on a grid. Each tile has an index number, and the game uses these indices to display text.

After analyzing the font data starting at `0x523`, I mapped out the character set:

| Tile Range | Characters |
|------------|------------|
| 0x00 | Space |
| 0x01-0x1A | A-Z |
| 0x1B-0x34 | a-z |
| 0x35-0x3E | 0-9 |
| 0x40 | { |
| 0x41 | } |
| 0x42 | _ |

---

## Two Flags, One ROM

Here's where it gets spicy. The ROM actually contains TWO different flags:

### The Fake Flag (Hardcoded)

At addresses `0x37B` and `0x3A8`, there are hardcoded tile writes that spell out:

```
flag{f4ke_but_g3tting_clos3r}
```

Notice the "f4ke" (fake)? Yeah, they're telling us straight up this isn't the real flag. But it's a good sign we're on the right track!

### The Real Flag (Encrypted)

When we decrypt the two data blocks with key `0xA5`:

**Block 1:** `1f 2d 1d 22 40 21 39 27 38 1c 35 33 42`
Decodes to: `esch{g4m3b0y_`

**Block 2:** `2c 38 30 30 1f 1e 42 3b 38 39 3a 36 37 41`
Decodes to: `r3vved_634512}`

Combined: **`esch{g4m3b0y_r3vved_634512}`**

---

## The Plot Twist

Here's the beautiful irony of this challenge:

**The "broken" code is actually CORRECT for revealing the flag!**

If you "fix" the CALL instruction to properly toggle the key:
- Block 2 would decrypt with key `0x5A` instead of `0xA5`
- You'd get garbage instead of the flag
- The challenge would become unsolvable!

The challenge author intentionally made the game "broken" so that both blocks use the same XOR key, which is the only way to see the real flag.

---

## Running the ROM

To verify, you can run `retro.gb` in any Game Boy emulator:
- **mGBA** (recommended)
- **BGB** (Windows, great debugger)
- **Gambatte**

You should see both flags displayed on screen - the fake one and the real one.

---

## Lessons Learned

1. **Don't assume "broken" means what you think** - Sometimes the broken state IS the intended behavior
2. **Check for decoy flags** - "f4ke" was a dead giveaway
3. **Understand the encryption before patching** - My initial instinct to "fix" the CALL would have made the challenge unsolvable
4. **Game Boy reversing is fun** - The tile-based graphics system is actually pretty elegant once you understand it

---

## Flag

```
esch{g4m3b0y_r3vved_634512}
```

Translation: "GameBoy revved" (reverse engineered) - because that's exactly what we did!

---

*Writeup by Claude Code*
