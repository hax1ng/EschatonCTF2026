# Paranoia - CTF Writeup

**Category:** AI/ML
**Points:** 100
**Difficulty:** Medium

## The Challenge

We're given a mysterious MNIST classifier that an ML engineer left behind before quitting under "mysterious circumstances." The description tells us she was paranoid about "them watching" and might have left us a message. The model works perfectly - standard digit recognition with great accuracy. But why would she leave it behind?

## Initial Recon

We get two files:
- `model_architecture.py` - The PyTorch model definition
- `trained_model.pth` - The trained model weights

Looking at the architecture, something immediately stands out:

```python
# Suspicious "padding" layers that form a residual connection
self.padding_fc1 = nn.Linear(512, 2048)
self.padding_fc2 = nn.Linear(2048, 2048)  # This is HUGE - 4 million parameters!
self.padding_fc3 = nn.Linear(2048, 512)
```

These "padding" layers are way overkill for MNIST (which is just 28x28 grayscale images of digits). The `padding_fc2` layer alone has 2048 x 2048 = **4,194,304 parameters**. That's 16MB of data just sitting there in a "residual connection" that essentially does nothing if the weights are near-zero.

Perfect place to hide a message.

## Going Down the Rabbit Hole

I tried a LOT of things that didn't work:

1. **Visualizing weights as images** - Just looked like noise
2. **Extracting LSBs from float mantissas** - Random garbage
3. **Sign bit encoding** - Nope
4. **Scaling weights to ASCII values** - Found some partial matches like "lf{" but nothing complete
5. **XORing layers together** - More noise
6. **Running the model with special inputs** - It just always predicts "1"

I was overthinking it.

## The Breakthrough

PyTorch `.pth` files are actually ZIP archives containing the serialized tensors. I extracted it:

```bash
unzip trained_model.pth -d extracted_model/
```

This gave me a `data/` folder with numbered files - each one is a raw tensor dump. File `29` was the big one at 16MB (our suspicious `padding_fc2` weights).

Here's the key insight: **16,777,216 bytes = 4096 x 4096 pixels**

What if the raw bytes of the weights aren't meant to be interpreted as floats at all, but as image data?

I extracted just the first byte of each float32 (the lowest byte of the mantissa) and saved it as an image:

```python
with open('data/29', 'rb') as f:
    raw = f.read()

arr = np.frombuffer(raw, dtype=np.uint8)
r_channel = arr[0::4].reshape(2048, 2048)  # Every 4th byte starting at 0
Image.fromarray(r_channel, mode='L').save('file29_r.png')
```

Then I ran `zsteg` on it (a tool for detecting steganography in images):

```bash
zsteg file29_r.png
```

And boom:

```
b1,r,lsb,xy  .. text: "esch{w31ght5_h1d3_s3cr3t5_m1_st3g}"
```

## The Flag

```
esch{w31ght5_h1d3_s3cr3t5_m1_st3g}
```

Which in leet speak translates to: **"weights hide secrets ml steg"**

## How It Actually Works

The engineer embedded her message using LSB (Least Significant Bit) steganography:

1. Take the raw bytes of the neural network weights
2. Treat them as if they were pixels in an image
3. Hide data in the least significant bit of each byte
4. The LSBs of float32 mantissas are so tiny they don't affect the model's accuracy at all

This is actually a really clever technique! The model still works perfectly for digit classification because changing the last bit of a weight like `0.0234567` to `0.0234566` makes essentially zero difference to the neural network's behavior. But those tiny changes can encode an entire hidden message.

## Lessons Learned

1. **Check file formats** - `.pth` files are just ZIPs, always extract and examine the contents
2. **Think about data capacity** - Suspiciously large layers in a model = room to hide stuff
3. **Use the right tools** - `zsteg` is amazing for image steganography
4. **Don't overthink it** - I spent way too long on complex ML-specific techniques when the answer was classic image stego

## Tools Used

- Python with PyTorch, NumPy, PIL
- `unzip` to extract the model archive
- `zsteg` for steganography detection
- `binwalk` for file analysis

---

*"I don't trust this model. It's too... perfect."* - Yeah, because it had a secret message hiding in plain sight the whole time.
