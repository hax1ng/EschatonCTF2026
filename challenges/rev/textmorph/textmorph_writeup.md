# TextMorph Writeup

**Challenge:** textmorph
**Category:** Reverse Engineering
**Author:** @solvz
**Flag:** `esch{f0und_th3_h1dden_r1gby}`

---

## The Challenge

We're given a mysterious binary called `textmorph` described as a "text transformation utility from a suspicious source." Our job? Figure out what it's hiding.

---

## Step 1: What Are We Dealing With?

First things first - let's see what kind of binary this is:

```bash
file textmorph
```

Turns out it's a **PyInstaller-packed Python executable**. PyInstaller is a tool that bundles Python scripts into standalone executables. This means somewhere inside this 11MB binary is the actual Python code we need to look at.

Running the binary shows it's a legit-looking text utility:

```bash
./textmorph --help
```

It has commands like `encode`, `decode`, `hash`, `reverse` - seems innocent enough. But we know there's more to it.

---

## Step 2: Unpacking the Python Code

To get at the Python code inside, we use a tool called `pyinstxtractor`:

```bash
python3 pyinstxtractor.py textmorph
```

This extracts all the bundled files. The juicy one is `textmorph_embedded.pyc` - a compiled Python file weighing in at 4MB (suspiciously large for a simple text utility).

---

## Step 3: Digging Into the Bytecode

Python `.pyc` files are compiled bytecode, but we can still extract useful info using Python's `marshal` module:

```python
import marshal

with open('textmorph_extracted/textmorph_embedded.pyc', 'rb') as f:
    f.read(16)  # Skip the header
    code = marshal.load(f)

# Look at all the string constants
for const in code.co_consts:
    if isinstance(const, str):
        print(const)
```

This reveals some interesting stuff:

1. **Fake flags** designed to troll us:
   - `CTF{tr0lled_by_v3rsi0n_str1ng}`
   - `CTF{c0nfig_1s_n0t_th3_w4y}`
   - `CTF{3nv_v4r_tr4p_l0l}`
   - `CTF{l3g4cy_c0d3_tr4p_h4h4}`

2. **A hidden command-line flag:** `--morph-hierarchical-sync`

3. **A massive base64-encoded blob** starting with `eNrc...`

---

## Step 4: The Hidden Command

Running the secret flag:

```bash
./textmorph --morph-hierarchical-sync
```

Just prints "Cache synchronized." - not very exciting. But we know there's that huge encoded blob in the code...

---

## Step 5: Decoding the Hidden Data

That base64 blob is actually **zlib-compressed data**. Let's decode it:

```python
import base64
import zlib

blob = "eNrc22VTW23Dtm..."  # The full blob from the pyc
decoded = base64.b64decode(blob)
decompressed = zlib.decompress(decoded)

with open('hidden.gif', 'wb') as f:
    f.write(decompressed)
```

And boom - we get a **3MB animated GIF**!

---

## Step 6: The Flag Reveal

Opening `hidden.gif` reveals a 74-frame animation of a dancing cat. But overlaid on the animation is text that spells out the flag:

```
esch{f0und_th3_h1dden_r1gby}
```

The text uses leetspeak:
- `0` instead of `o` in "found"
- `3` instead of `e` in "the"
- `1` instead of `i` in "hidden"
- `1` instead of `i` in "rigby"

---

## Summary

The solve path:
1. Recognize PyInstaller-packed binary
2. Extract with pyinstxtractor
3. Analyze the embedded `.pyc` bytecode
4. Find the hidden base64+zlib blob
5. Decompress to reveal a GIF
6. Read the flag from the animation

The challenge author hid multiple fake flags to throw off anyone just grepping for `CTF{`. The real flag was buried in compressed image data embedded in the Python bytecode. Sneaky!

---

## Tools Used

- `file` - identify binary type
- `pyinstxtractor` - unpack PyInstaller executables
- Python `marshal` module - analyze bytecode
- Python `base64` and `zlib` - decode hidden data
- Any image viewer - view the GIF

---

*GG @solvz - that was a fun one!*
