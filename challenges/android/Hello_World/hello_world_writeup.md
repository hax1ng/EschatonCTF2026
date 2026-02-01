# Hello World - Android Challenge Writeup

## What We're Working With

We get a file called `helloworld.dat` - about 11MB of mystery data. Time to figure out what it is and find that flag!

## Step 1: What Even Is This File?

First things first - let's see what we're dealing with. Looking at the file header, it starts with "ANDROID BACKUP". That tells us this is a backup file from an Android phone, the kind you'd get if you ran `adb backup` on a device.

Here's the catch though - it's encrypted with AES-256. So we can't just unzip it and look inside. We need the password first.

The backup header shows:
- Encryption: AES-256 (the strong stuff)
- 10,000 PBKDF2 iterations (makes brute-forcing slower)

## Step 2: Cracking the Password

Android backups can be password-protected, but people often use weak passwords. I extracted the hash in the right format and threw hashcat at it with the famous rockyou wordlist (14 million common passwords):

```bash
hashcat -m 18900 hash.txt /usr/share/wordlists/rockyou.txt
```

After a few minutes of crunching... **Password found: `hello world`**

Yep, with a space in the middle! The challenge name was literally the hint the whole time. Classic.

## Step 3: Extracting the Backup

Now that we have the password, we can use Android Backup Extractor (abe.jar) to unpack it:

```bash
java -jar abe.jar unpack helloworld.dat backup.tar "hello world"
```

This spits out a tar archive. Inside we find a bunch of Android app data, but one stands out: `com.mcsc.helloworld` - that's our target app!

The interesting bits:
- `base.apk` - The actual Android app
- `.notinhere.db` - A suspiciously named database file (hidden files start with a dot)

## Step 4: The Database is Locked Too

I tried opening `.notinhere.db` with regular SQLite tools but got "file is not a database". That's because it's encrypted with SQLCipher - basically SQLite but with encryption baked in.

So where's the key? It's gotta be in the app somewhere...

## Step 5: Reverse Engineering the APK

I decompiled the APK using jadx (a tool that turns Android apps back into readable Java code). Poking around the source code, I found `NoteDatabase.java`:

```java
private static final String NOT_HERE = ".notinhere.db";
private static final String NOT_THIS = "1s_th1s_th3_fl4g?";
```

Well well well... `NOT_THIS` is literally the encryption key, and they even named the variable like "don't look here!" The key is: **`1s_th1s_th3_fl4g?`**

(Spoiler: it's not the flag, but it unlocks what is!)

## Step 6: Decrypting the Database

Armed with the key, I used Python with the sqlcipher3 library:

```python
import sqlcipher3

conn = sqlcipher3.connect('.notinhere.db')
cursor = conn.cursor()
cursor.execute("PRAGMA key = '1s_th1s_th3_fl4g?'")
cursor.execute("SELECT * FROM notes")
```

The database is a simple notes app. Most entries are just "Hello" and "World" test data, but one entry caught my eye:

```
(5, 'HelloWorld', 'esch{he1!0o0o0o0o0_w0r!d}', ...)
```

There it is!

## Flag

```
esch{he1!0o0o0o0o0_w0r!d}
```

## The Attack Chain

To summarize what we did:
1. Identified the file as an encrypted Android backup
2. Cracked the weak backup password ("hello world")
3. Extracted the backup and found an app with an encrypted database
4. Decompiled the APK to find the hardcoded database encryption key
5. Decrypted the database and found the flag hiding in a note

## Lessons Learned

- **Don't use weak backup passwords** - "hello world" got cracked in minutes
- **Don't hardcode encryption keys in your app** - Anyone can decompile an APK and find them
- **Naming things "NOT_THIS" doesn't actually hide them** - Security through obscurity isn't security at all

The challenge name was a double hint: the backup password AND the flag both play on "Hello World" - the first program every developer writes. Pretty clever!
