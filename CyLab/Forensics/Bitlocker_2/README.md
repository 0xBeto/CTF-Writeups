# Bitlocker-2 — Forensics (Medium) — 300 pts

**Category:** Forensics
**Difficulty:** Medium
**Author:** Venax

> Jacky has learnt about the importance of strong passwords and made sure to encrypt the BitLocker drive with a very long and complex password. We managed to capture the RAM while this drive was opened however. See if you can break through the encryption!

**Hint:** Try using a volatility plugin

---

## Overview

This time we're given two files:
- `bitlocker-2.dd` — a BitLocker-encrypted disk image
- `memdump.mem.gz` — a compressed RAM dump captured while the drive was mounted/unlocked

Since the password is described as "very long and complex," brute-forcing or dictionary-attacking the BitLocker hash is not realistic. Instead, since we have a memory dump taken **while the drive was open**, the decryption key (FVEK) is very likely still resident in RAM. This points directly toward memory forensics rather than password cracking.

---

## Steps

### 1. Identify the disk image

```bash
file bitlocker-2.dd
```

Same as before, the `-FVE-FS-` OEM-ID confirms this is a BitLocker-protected volume. `fdisk -l` and `mmls` again produce garbage partition entries, since there is no real partition table — the file is a raw FVE volume.

```bash
xxd bitlocker-2.dd | head -n 20
```

Confirms the BitLocker boot sector signature at offset `0x00`.

### 2. Prepare the memory dump

The RAM capture was provided compressed:

```bash
gunzip memdump.mem.gz
```

This produces `memdump.mem`, a raw memory image.

### 3. Attempt standard triage (dead ends)

A few standard forensic tools were tried first and didn't pan out:

```bash
fls -o memdump.mem      # fails — not a disk image, it's a memory dump
mmls memdump.mem        # no output — same reason
```

These confirmed the `.mem` file needs to be analyzed with a **memory forensics framework** (Volatility) rather than disk/filesystem tools.

### 4. Use Volatility to recover BitLocker keys

Volatility 3 doesn't have a plugin literally named `windows.bitlocker` (confirmed by the plugin list error), so the correct approach is to search the memory dump directly for the flag/key material using `strings`, since BitLocker key material and any decrypted plaintext that touched the volume can remain in memory in a recoverable form.

```bash
strings memdump.mem | grep "picoCTF{"
```

This immediately surfaces the flag in plaintext — because at some point while the volume was mounted, the flag file's contents were read into memory and remained there uncleared:

```
picoCTF{B1tl0ck3r_dr1v3_d3crypt3d_9029ae5b}
picoCTF{B1tl0ck3r_dr1v3_d3crypt3d_9029ae5b}
picoCTF{B1tl0ck3r_dr1v3_d3crypt3d_9029ae5b}
```

*(Note: a more thorough/intended approach would be to use a Volatility plugin such as `windows.memmap` / `windows.vadinfo` combined with tools like `bitlocker-fvek` extraction utilities to pull the FVEK/VMK straight out of RAM and then mount the volume with `dislocker`/`bdemount` using the recovered key — but since the flag was stored as plaintext and had already been cached in memory, a direct `strings` search was sufficient to solve the challenge.)*

---

## Flag

```
picoCTF{B1tl0ck3r_dr1v3_d3crypt3d_9029ae5b}
```

---

## Tools Used

| Tool | Purpose |
|---|---|
| `file` / `xxd` | Confirm BitLocker FVE volume signature |
| `gunzip` | Decompress the provided RAM dump |
| `fls` / `mmls` | Ruled out — not applicable to raw memory images |
| `vol` (Volatility 3) | Framework for memory forensics (plugin listing reviewed) |
| `strings` + `grep` | Directly recover the flag from memory contents |

## Lesson Learned

Encrypting a drive with a strong password protects against offline brute-force attacks, but it does not protect against **live memory capture**. Once a BitLocker volume is unlocked, the decryption key (and potentially plaintext data that was accessed) resides in RAM and can be extracted by anyone with access to a memory dump — regardless of how strong the on-disk password is. This is a classic illustration of the "cold boot attack" / memory-forensics risk to full-disk encryption.
