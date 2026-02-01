# Total Recall - Writeup

**CTF:** EschatonCTF 2026
**Category:** DFIR (Digital Forensics & Incident Response)
**Author:** @andrewcanil

## The Challenge

> A rogue process is hiding in this RAM dump. It is impersonating a critical Windows Service but running from the wrong path. Find the process and dump its memory strings to find the flag.

We get a file called `total-recall.mem.tar.gz`. Classic memory forensics stuff.

## Step 1 - Unpack and Figure Out What We're Working With

First things first, extract the archive:

```bash
tar -xzf total-recall.mem.tar.gz
```

This gives us `challenge1.mem` — a ~2GB RAM dump. The challenge mentions a "Windows Service" so your first instinct might be to throw Volatility at it with Windows plugins. I did exactly that. It didn't work.

```bash
vol -f challenge1.mem windows.pslist
# Unsatisfied requirement plugins.PsList.kernel.layer_name...
```

Hmm. Let's check what OS this actually is:

```bash
vol -f challenge1.mem banners.Banners
```

```
Linux version 6.16.8+kali-amd64 (devel@kali.org) ...
```

Plot twist — it's a **Linux** memory dump, not Windows. The "Windows Service" thing in the challenge description is a hint about what the rogue process is *pretending* to be.

## Step 2 - Hunt for the Rogue Process

Since Volatility couldn't find the right kernel symbols for this specific Kali version, I went old school — just run `strings` on the dump and look for anything that smells like a Windows service name:

```bash
strings challenge1.mem | grep -iE '(svchost|lsass|csrss|services\.exe)' | sort | uniq -c | sort -rn
```

Bingo:

```
18 svchost_fake.py
 6 "svchost_fake.py"  |  264 bytes  |  Python script
 5 python3 svchost_fake.py &
 5 /home/kali/svchost_fake.py
```

So someone wrote a Python script called `svchost_fake.py` (not exactly subtle naming lol) and ran it in the background. `svchost.exe` is one of the most critical Windows services — malware loves to impersonate it.

## Step 3 - How Is It Hiding?

Digging a bit more into the strings, I found:

```
[200~pip install setproctitle
"/usr/sbin/apache2 -k start"
```

`setproctitle` is a Python library that lets you change what your process looks like in tools like `ps` and `top`. So this script was renaming itself to look like an Apache web server process. Sneaky.

## Step 4 - Extract the Script and Get the Flag

The challenge says to "dump its memory strings." We know the script is 264 bytes, and we can find exactly where it lives in the dump:

```bash
grep -aboP 'import setproctitle' challenge1.mem
```

```
743108684:import setproctitle
```

Then just carve out the bytes around that offset:

```bash
dd if=challenge1.mem bs=1 skip=743108600 count=400 | strings
```

And there it is — the full script sitting in memory:

```python
import time
import setproctitle

SECRET_FLAG = "esch{linux_proc_maps_reveal_all}"

setproctitle.setproctitle("/usr/sbin/apache2 -k start")
while True:
    time.sleep(1)
```

## Flag

```
esch{linux_proc_maps_reveal_all}
```

## TL;DR

1. The memory dump is Linux, not Windows — don't get baited by the challenge description
2. A Python script named `svchost_fake.py` (impersonating the Windows `svchost.exe` service) was running from `/home/kali/`
3. It used `setproctitle` to disguise itself as `/usr/sbin/apache2 -k start` in process listings
4. The flag was stored as a plaintext string variable right in the script's memory
5. `strings` + `grep` + `dd` is all you need when Volatility can't find symbols

## Takeaways

- Don't always trust process names. Tools like `setproctitle` or even just symlinks can make a process appear as something completely different.
- When Volatility fails (missing symbols, weird kernel versions), fall back to raw string searches. It's ugly but it works.
- The flag name itself is a hint — `/proc/<pid>/maps` on Linux reveals the true memory mappings of a process regardless of what it calls itself. That's how you'd catch this kind of trick on a live system.
