# Going Down - OSINT Challenge Writeup

**Category:** OSINT
**Points:** 100
**Author:** @andrewcanil

## Challenge Description

> This photo shows you something, but not everything. Find out what lies within.

We're given a single image file: `Going_Down.png`

## Initial Analysis

Opening the image, we see a beautifully carved stone sculpture of what appears to be a Hindu deity. The intricate stonework suggests this is from an ancient Indian temple or monument.

But the challenge says the photo "shows you something, but not everything" and asks us to "find out what lies within." That's a classic hint that there's hidden data in the image!

## Finding the Hidden Message

When you suspect an image has hidden data, one of the first tools to try is `zsteg` - a tool that detects steganography (hidden data) in PNG files.

```bash
zsteg Going_Down.png
```

And boom! Hidden in the least significant bits of the RGB channels, we find a secret message:

> "You prolly carry an image of this particular statue in your pocket everyday (If you're an Indian). But you might never have seen it. Find out where it's from.
>
> Flag Format: esch{name_of_place}
>
> P.S: use _ as separator, and refer to wikipedia for exact case"

## Connecting the Dots

So now we have some solid clues:

1. **"Going Down"** - the challenge title
2. **"carry an image in your pocket everyday"** - sounds like currency!
3. **100 points** - hmm, suspicious number
4. **Indian context** - the sculpture style and the hint about Indians

If you're Indian and you carry something in your pocket with a monument on it... that's money! And 100 points for the challenge? That's pointing us to the **100 Rupee note**.

## The Answer

A quick search reveals that the back of the Indian ₹100 note (introduced in 2018) features **Rani ki Vav** - a stunning stepwell located in Patan, Gujarat.

And suddenly the challenge title makes perfect sense: **"Going Down"** - because a stepwell is literally a structure you walk DOWN into to reach water!

### What is Rani ki Vav?

- Built around 1050 AD by Queen Udayamati in memory of her husband King Bhima I
- It's an inverted temple that highlights the sanctity of water
- Features over 500 principal sculptures and 1000+ minor ones
- Measures about 64 meters long, 20 meters wide, and 27 meters deep
- UNESCO World Heritage Site since 2014
- Featured on the ₹100 note since July 2018

## Getting the Exact Flag

The hint said to check Wikipedia for the exact capitalization. The Wikipedia article title is "Rani ki Vav" (with lowercase "ki").

Using underscores as separators as instructed:

## Flag

```
esch{Rani_ki_Vav}
```

## Tools Used

- `zsteg` - for detecting LSB steganography in PNG files
- Web search - for identifying the monument on Indian currency

## Lessons Learned

1. Always check images for hidden data - `strings`, `exiftool`, `binwalk`, and `zsteg` are your friends
2. Pay attention to challenge titles and point values - they're often hints!
3. OSINT challenges love to connect everyday objects (like currency) to historical/cultural knowledge
4. When in doubt, Google it - reverse image search and keyword searches can solve a lot

Pretty neat challenge that combines steganography with cultural knowledge. The 100-point value matching the ₹100 note was a clever touch!
