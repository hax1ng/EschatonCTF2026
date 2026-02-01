# Old School Messaging - Writeup

**Category:** DFIR (Digital Forensics & Incident Response)
**Flag:** `esch{c0mmunicat1on_d3c0d3d_th3_0ld_way}`

---

## The Challenge

We're given a network capture file (`capture.pcapng`) and a hint about dial-up modems, 56k handshakes, and BBS (Bulletin Board Systems). Basically, we need to channel our inner 90s hacker and find a hidden flag.

> "Back in my day, we didn't have fancy encrypted messengers. We had dial-up modems and the sweet sound of a 56k handshake. Find the flag to be worthy of the BBS."

---

## Step 1: What's in the Capture?

First things first - let's see what kind of traffic we're dealing with. I opened the pcap file and checked what protocols were flying around:

```
Protocol Hierarchy:
- HTTP (web traffic)
- SMTP (email) <-- this looks promising for "messaging"!
- DNS (domain lookups)
- ICMP (ping)
```

SMTP stands for Simple Mail Transfer Protocol - it's how emails get sent around. Given the challenge is about "old school messaging," email seems like the obvious place to look. Back in the day, email WAS the messaging app!

---

## Step 2: Digging Into the Emails

I found 4 different SMTP conversations in the capture. Most were newsletters and spam (classic 90s internet vibes - Netscape 4.0 announcements, Yahoo reaching 1 million pages, etc.).

But one email caught my eye - it was from **"SysOp Dave"** at `retro-bbs.net` about account verification:

```
From: SysOp Dave <sysadmin@retro-bbs.net>
To: New User <user42@dial-up.com>
Subject: RE: BBS Account Verification - Action Required

Hello fellow Netizen!

Thank you for registering at Retro-BBS! We're excited to have you join our
community of dial-up enthusiasts and message board aficionados.

To complete your account verification, please scan the attached QR code...
```

A QR code attachment? That's definitely suspicious. The email even mentions:

> "P.S. If the image doesn't open, your mail client might have corrupted it during transmission. Those darn 8-bit clean issues!"

This is a huge hint that the image might be intentionally broken!

---

## Step 3: Extracting the Attachment

The email had a base64-encoded PNG file attached called `verify_account.png`. I extracted the base64 data and decoded it:

```python
import base64
decoded = base64.b64decode(base64_data)
```

When I tried to open it... nothing. The file was corrupted.

---

## Step 4: Fixing the Broken PNG

Looking at the raw bytes of the file, I noticed the problem immediately:

```
Actual:   00 50 4E 47 0D 0A 1A 0A ...
Expected: 89 50 4E 47 0D 0A 1A 0A ...
```

The first byte was `0x00` instead of `0x89`. Every PNG file starts with the magic bytes `89 50 4E 47` (which spells out `.PNG` with a special header byte). Someone had zeroed out that first byte!

This was the "8-bit clean issue" the email warned about - a clever nod to real problems that used to happen when sending binary files through old email systems.

The fix was simple - just change that first byte:

```python
data = bytearray(open('broken.png', 'rb').read())
data[0] = 0x89  # Fix the magic byte
open('fixed.png', 'wb').write(data)
```

---

## Step 5: Reading the QR Code

With the PNG fixed, I could finally see it was a valid QR code. Using a QR decoder:

```python
from pyzbar.pyzbar import decode
from PIL import Image

img = Image.open('fixed.png')
result = decode(img)
print(result[0].data.decode())
```

**Output:**
```
esch{c0mmunicat1on_d3c0d3d_th3_0ld_way}
```

---

## TL;DR

1. Open the pcap, find SMTP (email) traffic
2. Extract a suspicious email with a QR code attachment
3. Notice the PNG is broken (first byte corrupted)
4. Fix the PNG header (change `0x00` to `0x89`)
5. Scan the QR code to get the flag

---

## Tools Used

- **tshark/Wireshark** - For analyzing the network capture
- **Python** - For extracting and fixing the image
- **pyzbar** - For decoding the QR code

---

## Lessons Learned

This challenge was a fun throwback to the early internet days. The "8-bit clean" hint in the email was actually referencing real issues from back then - some email systems couldn't handle binary data properly and would corrupt attachments. The challenge author cleverly simulated this by breaking the PNG header.

Always check your magic bytes, kids!

---

*Flag: `esch{c0mmunicat1on_d3c0d3d_th3_0ld_way}`*
