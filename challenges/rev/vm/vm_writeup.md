# VM Challenge Writeup

**Category:** Reverse Engineering
**Flag:** `esch{br0k3_th3_vm_4ndd_th3_c1pher!!}`

## The Challenge

We're given two files:
- `vm` - An executable that runs a custom virtual machine
- `binary.bin` - Bytecode that the VM executes

The program asks us to enter 16 bytes as 32 hex characters, does *something* to our input, and tells us if we got it right or wrong.

```
$ ./vm binary.bin
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Wrong!
```

Our job: figure out what input gives us the flag.

## Initial Recon

First things first - let's see what we're dealing with:

```
$ file vm
vm: ELF 64-bit LSB executable, x86-64, not stripped
```

Not stripped means we get function names! Loading it into Binary Ninja, I immediately spotted some interesting functions:

- `op_nop`, `op_push`, `op_pop` - basic stack operations
- `op_sbox`, `op_permute`, `op_decrypt` - crypto stuff!
- `op_cmpmem` - comparison (probably checks our answer)

This is a stack-based virtual machine with crypto operations baked in. The bytecode file contains the program it runs.

## Understanding the VM

The VM has a pretty simple architecture:
- A stack for operations
- Memory for storing data
- A dispatch table mapping opcodes to handler functions

One quirk I discovered: all memory operations add `0x100` to the address. So when the bytecode says "load from address 0x10", it actually reads from `memory[0x110]`. This tripped me up for a while!

### The Opcodes

After reversing the dispatch table, I mapped out the key opcodes:

| Opcode | Name | Description |
|--------|------|-------------|
| 0x01 | push | Push a byte onto stack |
| 0x20 | load | Load from memory |
| 0x21 | store | Store to memory |
| 0x40 | sbox | S-box substitution |
| 0x41 | permute | Shuffle bytes around |
| 0x42 | decrypt | XOR-decrypt bytecode data |
| 0x52 | input | Read user input |
| 0x61 | cmpmem | Compare memory regions |

## Disassembling the Bytecode

The bytecode starts with a jump to address `0x200`, skipping over embedded data tables. The program flow is:

1. **Decrypt embedded tables** - The bytecode contains encrypted S-box, round keys, and permutation tables
2. **Read user input** - 32 hex chars = 16 bytes
3. **Encrypt the input** - Apply a custom cipher
4. **Compare result** - Check against expected ciphertext
5. **Print result** - Flag if correct, "Wrong!" if not

### Extracting the Crypto Tables

The decrypt operations revealed where all the goodies are hidden:

```python
# S-box: 256 bytes at bytecode[0x10], XOR'd with 0x5a
SBOX = [b ^ 0x5a for b in bytecode[0x10:0x110]]

# Permutation: [2, 5, 0, 7, 4, 1, 6, 3]
PERM = [b ^ 0x33 for b in bytecode[0x110:0x118]]

# Round keys: 32 bytes (4 rounds x 8 bytes each)
ROUND_KEYS = bytes([b ^ 0x7f for b in bytecode[0x118:0x138]])

# Expected output we need to match
EXPECTED = bytes([b ^ 0x42 for b in bytecode[0x158:0x168]])
# = 10e08e4e669108f8478c5b3a31c15ada
```

## The Cipher

After tracing through the disassembly, I figured out the encryption algorithm. It processes the 16-byte input as two 8-byte halves, each going through 4 rounds of:

1. **S-box substitution** - Each byte is replaced using a lookup table
2. **XOR with round key** - Mix in the round-specific key
3. **Rotate left** - Each byte is rotated by a different amount
4. **Permute** - Shuffle the byte positions
5. **Feistel mixing** - XOR bytes together with rotations

The Feistel step was the trickiest part. Each output byte depends on three input bytes with position-dependent rotations:

```
result[i] = block[i] XOR rol(block[(i+1) % 8], i+1) XOR block[(i+3) % 8]
```

## Building the Inverse

To find our input, we need to reverse the cipher. Most steps are straightforward to invert:
- Inverse S-box: just reverse the lookup table
- XOR: same operation both ways
- Rotate left → Rotate right
- Inverse permute: reverse the shuffle

The Feistel mixing was trickier since each output byte depends on multiple inputs. I used Z3 (an SMT solver) to solve the system of equations:

```python
def inv_feistel_mix(result):
    solver = Solver()
    b = [BitVec(f'b{i}', 8) for i in range(8)]

    for i in range(8):
        solver.add(b[i] ^ RotateLeft(b[(i+1)%8], (i+1)%8) ^ b[(i+3)%8] == result[i])

    if solver.check() == sat:
        model = solver.model()
        return [model[b[i]].as_long() for i in range(8)]
```

## Getting the Flag

With the inverse function working, I decrypted the expected output:

```python
first_half = decrypt_half(EXPECTED[:8])
second_half = decrypt_half(EXPECTED[8:16])
solution = bytes(first_half + second_half)
print(solution.hex().upper())
# DEADBEEFCAFEBABE1337C0DEF00DFACE
```

Classic CTF-style hex values! Let's verify:

```
$ echo "DEADBEEFCAFEBABE1337C0DEF00DFACE" | ./vm binary.bin
esch{br0k3_th3_vm_4ndd_th3_c1pher!!}
```

## Key Takeaways

1. **VM challenges are about pattern recognition** - Once you identify it's a stack-based VM with crypto ops, you know what to look for.

2. **Watch for address offsets** - The +0x100 offset on all memory operations was a subtle but crucial detail.

3. **Don't try to z3 the whole thing** - I initially tried symbolic execution through the entire cipher, which timed out. Breaking it into smaller inversions (especially for the Feistel step) was much faster.

4. **Test incrementally** - Building an emulator and verifying round-trips saved me from chasing phantom bugs.

## Files

- `emulator.py` - Full VM emulator
- `inverse.py` - Cipher inverse and solver
- `disasm.py` - Bytecode disassembler

## Final Answer

**Input:** `DEADBEEFCAFEBABE1337C0DEF00DFACE`
**Flag:** `esch{br0k3_th3_vm_4ndd_th3_c1pher!!}`
