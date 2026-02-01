# Auth Writeup

**Category:** Crypto
**Flag:** `esch{modes-change-security-shadow-viper-2317}`

---

## The Challenge

We're told that a security team intercepted network traffic from a company's "secure" file server. We get a `.pcap` file (basically a recording of network traffic) and access to a live server to exploit.

## Step 1: Digging Through the Traffic

First thing I did was crack open the `capture.pcap` file with `tshark` to see what was going on. Right away, something juicy popped out - a DNS query for `auth.internal.corp` with a TXT record response:

```
TXT: v=auth1 key=k3yM4st3r_S3cr3t!
```

Uh oh. Someone put their secret authentication key in a DNS record. That's... not great security practice, to put it mildly.

## Step 2: Understanding the Auth Protocol

Looking at the HTTP traffic in the capture, I could see how the authentication worked:

**Request 1 - Get a Challenge:**
```
GET /challenge
```
Response:
```json
{"type": "challenge", "challenge": "502589772c996cd8b11f0a1898eb565a", "nonce": 1}
```

**Request 2 - Prove You Know the Secret:**
```
POST /verify
{"response": "26996bf1f34408ece51ea8d4bb547212471f6d6cc42cee670483e829363afdef",
 "nonce": 2,
 "challenge": "502589772c996cd8b11f0a1898eb565a"}
```

So basically:
1. Server gives you a random challenge and a nonce (a number that increments)
2. You compute some hash using the secret key and send it back
3. Server checks if you got it right

## Step 3: Cracking the Hash Formula

The response is 64 hex characters, which screams SHA256. But what exactly gets hashed?

I tried a bunch of combinations:
- `key + challenge + nonce`
- `challenge + key + nonce`
- `nonce + challenge + key`
- Various HMAC combinations

After some trial and error, the winner was:

```
SHA256(key + challenge + nonce)
```

Where:
- `key` = `k3yM4st3r_S3cr3t!`
- `challenge` = whatever the server gives you
- `nonce` = the server's nonce + 1

## Step 4: The Exploit

With the secret key from DNS and the hash formula figured out, exploiting the live server was straightforward:

```python
import requests
import hashlib

HOST = "node-3.mcsc.space"
PORT = 35048
KEY = "k3yM4st3r_S3cr3t!"

# Get challenge from server
r = requests.get(f"http://{HOST}:{PORT}/challenge")
data = r.json()

challenge = data["challenge"]
nonce = data["nonce"]

# Calculate the response hash
response_nonce = nonce + 1
to_hash = f"{KEY}{challenge}{response_nonce}"
response = hashlib.sha256(to_hash.encode()).hexdigest()

# Send it back
payload = {
    "response": response,
    "nonce": response_nonce,
    "challenge": challenge
}
r = requests.post(f"http://{HOST}:{PORT}/verify", json=payload)
print(r.json())
```

Output:
```json
{"type":"success","message":"Authentication successful!","flag":"esch{modes-change-security-shadow-viper-2317}"}
```

## The Vulnerability

The whole security of this system depended on keeping that key secret. But someone decided to broadcast it via DNS - which travels over the network in plaintext for anyone to see.

Lessons learned:
1. Don't put secrets in DNS records
2. Don't put secrets anywhere they can be sniffed on the network
3. Use proper key exchange protocols (like TLS) to protect sensitive data in transit

---

*Solved during EschatonCTF 2026*
