# Now You See Me - Writeup

**Category:** Web
**Points:** 500
**Author:** @psychoSherlock

## TL;DR

The flag was hidden in plain sight using invisible Unicode characters in a JavaScript file. Decoding the invisible text revealed the flag.

---

## First Look

Opening the challenge URL, we're greeted with a creepy eyeball animation - dozens of eyes following your cursor around the screen. Spooky stuff, but where's the flag?

The page source has an interesting HTML comment:

```html
<!--Now you try more to see me, but still you dont-->
```

Right-clicking? Blocked. Ctrl+U to view source? Blocked with an alert. The challenge is literally telling us "you can't see me." Classic CTF trolling.

## Poking Around

Let's check the usual suspects:

```bash
curl http://node-2.mcsc.space:10627/robots.txt
```

Output:
```
esch{not_that_easy_bro}
```

Lol, nice try. That's obviously a decoy flag. Moving on...

## The JavaScript Rabbit Hole

Looking at `index.js`, most of it is just physics code for the eyeball animation (using Matter.js). But scroll down to the bottom and you'll see something... weird:

```javascript
new Proxy(
  {},
  {
    get: (_, n) =>
      eval(
        [...n].map((n) => +("ﾠ" > n)).join``.replace(/.{8}/g, (n) =>
          String.fromCharCode(+("0b" + n)),
        ),
      ),
  },
)
  .ﾠﾠㅤﾠﾠﾠﾠﾠﾠﾠㅤﾠﾠﾠﾠﾠﾠﾠㅤﾠﾠﾠﾠﾠ...
```

Wait, what's after that dot? It looks empty but... it's not. If you copy-paste that "empty" space, you'll find it's actually thousands of invisible characters!

## The Invisible Ink Trick

The code uses two special Unicode characters that look identical (both invisible):

| Character | Unicode | Code Point |
|-----------|---------|------------|
| ﾠ | HALFWIDTH HANGUL FILLER | U+FFA0 (65440) |
| ㅤ | HANGUL FILLER | U+3164 (12644) |

These are legitimate Unicode characters used in Korean text, but here they're being abused to hide data.

## How the Decoder Works

Let's break down that Proxy magic:

```javascript
get: (_, n) => eval(
    [...n]                              // Split property name into chars
    .map((n) => +("ﾠ" > n))             // Compare each to ﾠ: true=1, false=0
    .join``                             // Join into binary string
    .replace(/.{8}/g, (n) =>            // Take 8 bits at a time
        String.fromCharCode(+("0b" + n)) // Convert binary to ASCII
    )
)
```

So basically:
1. Access a property with an invisible name (the long string of ﾠ and ㅤ)
2. Each character gets converted: `ㅤ` → 1 (smaller codepoint), `ﾠ` → 0
3. Every 8 bits becomes an ASCII character
4. The result gets `eval()`'d

It's like binary, but instead of 0s and 1s, you have invisible characters. Sneaky!

## Decoding It

I wrote a quick Python script to decode:

```python
with open('index.js', 'r', encoding='utf-8') as f:
    content = f.read()

halfwidth = '\uffa0'  # ﾠ (represents 0)
hangul = '\u3164'     # ㅤ (represents 1)

# Extract invisible characters
invisible = ''
for c in content:
    if c == halfwidth or c == hangul:
        invisible += c

# Decode to binary
binary = ''
for c in invisible:
    binary += '1' if c == hangul else '0'

# Convert to ASCII
result = ''
for i in range(0, len(binary), 8):
    byte = binary[i:i+8]
    if len(byte) == 8:
        result += chr(int(byte, 2))

print(result)
```

## The Hidden Code

Running the decoder reveals hidden JavaScript that creates a styled message div, and more importantly:

```javascript
messageDiv.textContent = "Do you see me?";
// esch{y0u_s33_,_but_u_d0_n0t_0bs3rv3}
// Said Sherlock Holmes.
```

## Flag

```
esch{y0u_s33_,_but_u_d0_n0t_0bs3rv3}
```

## The Reference

The flag is a play on a famous Sherlock Holmes quote:

> "You see, but you do not observe."

Which is perfect given the challenge author is @psychoSherlock. The whole challenge is about looking at something without truly *seeing* what's there - just like the invisible Unicode characters hiding in plain sight.

## Lessons Learned

1. **Always check for invisible/zero-width characters** - They're a popular way to hide data in web challenges
2. **Decoy flags exist** - Don't submit the first flag-looking thing you find
3. **View the raw source** - Browser dev tools might not show you everything
4. **Unicode is weird** - There are thousands of invisible or look-alike characters that can be abused

## Tools Used

- curl (for fetching raw files)
- Python (for decoding)
- A good text editor that shows invisible characters

---

*Now you see it. Now you don't... but now you do again!*
