# Couple Gallery Writeup

**Category:** Android Reverse Engineering
**Difficulty:** Medium
**Flag:** `esch{wh0_s3aid_r3act_n4t1v3_1s_s3cure?_k3y_w4s_369963}`

---

## Challenge Description

> My girlfriend stored all our private images on a "Secure App". She forgot the passcode and now all our memories are lost. The passcode is a 6-DIGIT NUMBER. Can you help me recover our photos?

We're given an Android APK file: `com.mcsc.couplegallery.apk`

---

## Initial Analysis

### What Are We Working With?

First things first - let's figure out what kind of app this is. After extracting the APK (it's basically just a ZIP file), we find:

```
├── assets/
│   └── index.android.bundle    # <-- This is interesting!
├── lib/
│   └── libhermes.so           # <-- Hermes JavaScript engine
├── res/
│   └── raw/
│       └── assets_gallery_secure.db  # <-- SQLite database with encrypted images
└── classes.dex
```

The presence of `index.android.bundle` and `libhermes.so` tells us this is a **React Native** app. React Native apps are built with JavaScript, but they compile the JS into bytecode (called Hermes bytecode) for better performance.

### The Database

Inside the SQLite database, we find 5 encrypted images stored as base64 text:

```sql
CREATE TABLE images (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    encrypted_data TEXT NOT NULL
);
```

Each encrypted blob starts with `U2FsdGVkX1` which decodes to `Salted__` - this is the telltale sign of **OpenSSL/CryptoJS encryption format**.

---

## Digging Into the JavaScript Bundle

Since we can't easily decompile Hermes bytecode (it's not regular JavaScript), we resort to good old `strings` to extract readable text:

```bash
strings index.android.bundle | grep -i "decrypt\|blowfish\|password"
```

### Key Findings

1. **"BlowFish_Decrypting image"** - Confirms the encryption algorithm is Blowfish
2. **"d5db3d7813ed77d64da36b964d1a7e5c"** - A hardcoded secret! (32 hex chars = 16 bytes)
3. **"data:image/png;base64,"** - The expected decrypted format (a data URI)
4. **"g$c369963"** - A suspicious 6-digit number in the compiled code

The `g$c` prefix is how Hermes represents constants. So `369963` is likely our PIN!

---

## Finding the Hidden Flag Part 1

Here's where the challenge gets sneaky. The first part of the flag is hidden in plain sight using a clever steganography technique.

Searching the bundle for leet-speak patterns:

```bash
cat index.android.bundle | tr -cd '[:print:]\n' | grep -oE 'wh0_s3aid_r3act_n4t1v3[a-zA-Z0-9_]*' | head -10
```

Output:
```
wh0_s3aid_r3act_n4t1v3_offsetFromParentVirtualizedList
```

The challenge author **concatenated the flag with a legitimate React Native variable name**:

```
wh0_s3aid_r3act_n4t1v3_offsetFromParentVirtualizedList
└────────┬───────────┘└─────────────┬─────────────────┘
      FLAG PART 1            Legit RN code
```

This made it look like normal React Native internals, hiding in plain sight! You'd only find it by:
- Searching for leet-speak patterns
- Getting lucky with broad string searches
- Decrypting an image to find the hint

---

## Understanding the Encryption

The encrypted data follows the **OpenSSL "Salted__" format**:

```
[Salted__][8-byte salt][ciphertext]
   8 bytes    8 bytes     rest
```

CryptoJS uses this format with **EVP_BytesToKey** key derivation:
- Takes a password + salt
- Runs MD5 iterations to derive key and IV
- For Blowfish: 16-byte key, 8-byte IV

### The Decryption Flow

```
Password (PIN) + Salt → MD5 iterations → Key + IV → Blowfish CBC → Decrypted data
```

---

## The Attack Plan

We know:
- 6-digit PIN (000000 to 999999 = 1 million possibilities)
- Blowfish-CBC encryption
- Expected output starts with "data:image"

This is totally brute-forceable! Here's a Python script that does it:

```python
import base64
import hashlib
from Crypto.Cipher import Blowfish

def evp_derive(password, salt, key_len=16, iv_len=8):
    """OpenSSL-style key derivation"""
    d = b''
    key_iv = b''
    while len(key_iv) < key_len + iv_len:
        d = hashlib.md5(d + password + salt).digest()
        key_iv += d
    return key_iv[:key_len], key_iv[key_len:key_len+iv_len]

# Load encrypted data
with open('encrypted_sample.txt', 'r') as f:
    enc_b64 = f.read().strip()

raw = base64.b64decode(enc_b64)
salt = raw[8:16]      # Skip "Salted__"
ciphertext = raw[16:]

# Brute force all PINs
for pin in range(1000000):
    pinstr = f"{pin:06d}".encode()
    key, iv = evp_derive(pinstr, salt)

    cipher = Blowfish.new(key, Blowfish.MODE_CBC, iv)
    decrypted = cipher.decrypt(ciphertext[:16])

    if decrypted.startswith(b'data:image'):
        print(f"Found PIN: {pin:06d}")
        break
```

---

## Getting the Flag

Looking at the app screenshot (after running it with the correct PIN **369963**), we see cute hamster couple photos. The last image contains text:

```
FLAG PART 2:
_1s_s3cure?_k3y_w4s_<key here>
```

### Putting It All Together

| Part | Value | How Found | Meaning |
|------|-------|-----------|---------|
| Part 1 | `wh0_s3aid_r3act_n4t1v3` | Hidden in bundle (concatenated with RN code) | "who said react native" |
| Part 2 | `_1s_s3cure?_k3y_w4s_` | Visible in decrypted image | "is secure? key was" |
| PIN | `369963` | `g$c369963` constant in Hermes bytecode | The 6-digit passcode |

### Final Flag

```
esch{wh0_s3aid_r3act_n4t1v3_1s_s3cure?_k3y_w4s_369963}
```

**Translation:** *"Who said React Native is secure? Key was 369963"*

---

## Lessons Learned

1. **React Native apps aren't magic** - The JavaScript bundle contains strings and constants that can be extracted
2. **Hardcoded secrets are bad** - The encryption secret was sitting right there in the code
3. **6-digit PINs are weak** - 1 million combinations is nothing for a computer
4. **Steganography in code** - Flags can be hidden by concatenating them with legitimate variable names
5. **Defense in depth matters** - Even with Blowfish encryption, poor implementation = easy crack

---

## Tools Used

- **APKTool** - Decompile APK
- **strings/grep** - Extract readable text from binaries
- **SQLite3** - Read the encrypted database
- **Python + PyCryptodome** - Brute force decryption
- **Node.js + CryptoJS** - Verify encryption compatibility

---

## TL;DR

React Native "secure" gallery app encrypts images with Blowfish using a 6-digit PIN. The PIN (`369963`) was hardcoded in the JavaScript bundle as a Hermes constant. Flag part 1 (`wh0_s3aid_r3act_n4t1v3`) was cleverly hidden by concatenating it with a real React Native variable name. Flag part 2 was visible in the decrypted hamster photos.

*Who said React Native is secure? Nobody. Nobody said that.* 🐹💕
