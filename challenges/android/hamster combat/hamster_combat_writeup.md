# Hamster Combat - Writeup

**CTF:** EschatonCTF 2026
**Category:** Android
**Author:** @psychoSherlock

## TL;DR

Decompile the APK, find a native library doing all the game logic, extract a base64-encoded flag from a function that checks if your click streak hits 9999. Don't fall for the fake flag.

## Challenge Description

The author made a clicker game for his girlfriend who has ADHD -- tap a hamster as fast as you can without pausing for more than 250ms. If you reach a certain streak, a "surprise" pops up. Our job is to find that surprise (the flag) without actually tapping 9999 times in a row.

## Step 1 - Crack Open the APK

First things first, decompile the APK with `jadx`:

```bash
jadx -d /tmp/hamster/jadx HamsterCombat.apk
```

The interesting app code lives in `com/mcsc/hamstercombat/`. There are only two real files:

- **MainActivity.java** - the UI, tap handling, animations
- **GameState.java** - a simple data class holding click count, high score, scale, face ID, game over status, and... a `flag` field

## Step 2 - Spot the Native Library

Looking at `MainActivity.java`, the juicy stuff is all `native`:

```java
private final native GameState checkGameTimeout();
private final native GameState getGameState();
private final native void initGame(int initialHighScore);
private final native GameState onTap();
private final native GameState resetGame();

static {
    System.loadLibrary("hamstercombat");
}
```

So the actual game logic (and flag) lives inside `libhamstercombat.so`, not in Java. The Java side just checks if `state.getFlag()` returns something non-empty and shows it in a dialog titled "You found a secret!".

Extract it from the APK:

```bash
unzip HamsterCombat.apk "lib/x86_64/libhamstercombat.so"
```

I grabbed the x86_64 version since it's the easiest to reverse on a normal PC.

## Step 3 - The Fake Flag Trap

Running `strings` on the .so immediately reveals some interesting base64 blobs:

```
X2ZsNGdfMXNfZjRrM30K
WW91IGZpbmdlciBnb29k
```

Decoding them:

```bash
$ echo -n "X2ZsNGdfMXNfZjRrM30K" | base64 -d
_fl4g_1s_f4k3}

$ echo -n "WW91IGZpbmdlciBnb29k" | base64 -d
You finger good
```

Digging into the `get_flag()` function with radare2 reveals a third base64 string: `ISBlc2Noe2J1dF90aDFz` which decodes to `! esch{but_th1s`.

All three get concatenated together into:

> You finger good! esch{but_th1s_fl4g_1s_f4k3}

The name literally says "this flag is fake." Don't submit this one.

## Step 4 - The Real Flag in native_condition

The function `native_condition(int)` is where the real flag hides. Disassembling it in r2:

```bash
r2 -c 'aaa; s sym.native_condition_int_; pdf' libhamstercombat.so
```

Right at the top, there's a comparison:

```asm
cmp esi, 0x270f    ; 0x270f = 9999 decimal
jne 0x28dde        ; if streak != 9999, skip everything
```

So the streak threshold is **9999 taps without a 250ms pause**. Good luck doing that legitimately.

When the condition IS met, the function builds a base64 string one character at a time using `push_back` calls. Each character is loaded into `esi` before calling the append function:

```asm
mov esi, 0x5a  ; 'Z'
call push_back
mov esi, 0x58  ; 'X'
call push_back
mov esi, 0x4e  ; 'N'
call push_back
... (36 characters total)
```

Extracting every `mov esi` value in order gives us:

```
Z X N j a H t p X 2 w w d m V f b X l f Q U R I R F 9 n M X I h f Q = =
```

Which forms the base64 string:

```
ZXNjaHtpX2wwdmVfbXlfQURIRF9nMXIhfQ==
```

## Step 5 - Decode

```bash
$ echo -n "ZXNjaHtpX2wwdmVfbXlfQURIRF9nMXIhfQ==" | base64 -d
esch{i_l0ve_my_ADHD_g1r!}
```

## Flag

```
esch{i_l0ve_my_ADHD_g1r!}
```

## Key Takeaways

- When an Android app uses `System.loadLibrary()`, the real logic is in the native .so, not the Java/Kotlin code. Always extract and reverse the native lib.
- Strings visible in `strings` output can be decoys. The challenge literally told us `_fl4g_1s_f4k3}`.
- Building strings character-by-character is a common anti-strings technique in CTFs. Grep for repeated `push_back`-style calls and reconstruct manually.
- r2/Ghidra/Binary Ninja are your friends for pulling apart native Android libraries.

## Tools Used

- `jadx` - APK decompilation
- `radare2` - native library disassembly
- `base64` - decoding
- `strings` - initial recon
