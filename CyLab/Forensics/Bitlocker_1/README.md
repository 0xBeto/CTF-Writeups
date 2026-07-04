# Bitlocker-1 — Forensics (Medium) — 200 pts

**Category:** Forensics
**Difficulty:** Medium
**Author:** Venax

> Jacky is not very knowledgeable about the best security passwords and used a simple password to encrypt their BitLocker drive. See if you can break through the encryption!

**Hint:** Hash cracking

---

## Overview

We're given a disk image (`bitlocker-1.dd`) containing a BitLocker-encrypted volume. The goal is to crack the password protecting the volume and recover the flag stored inside.

---

## Steps

### 1. Identify the file

```bash
file bitlocker-1.dd
```

The output shows the `-FVE-FS-` signature, which is the marker for a **BitLocker Full Volume Encryption (FVE)** volume.

Attempts to read the partition table with `mmls` and `fdisk -l` returned nonsensical results (partitions of several terabytes inside a 100 MB file). This happens because the image is a raw BitLocker volume with no real MBR/GPT partition table — those tools misinterpret the FVE metadata as partition entries.

### 2. Extract the crackable hash

Use `bitlocker2john` to pull out the VMK (Volume Master Key) blobs protected by the Recovery Password and the User Password:

```bash
bitlocker2john -i bitlocker-1.dd > bitlocker.hash
```

This produces several hash formats (`$bitlocker$0$`, `$bitlocker$1$`, `$bitlocker$2$`, `$bitlocker$3$`), depending on protection type (User/Recovery) and verification mode (fast vs. MAC-verified).

Since a *simple password* is implied, we target the **User Password** hash:

```bash
echo '$bitlocker$0$16$cb4809fe9628471a411f8380e0f668db$1048576$12$d04d9c58eed6da010a000000$60$68156e51e53f0a01c076a32ba2b2999afffce8530fbe5d84b4c19ac71f6c79375b87d40c2d871ed2b7b5559d71ba31b6779c6f41412fd6869442d66d' > hash.txt
```

### 3. Crack the password with John the Ripper

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

After a few minutes, John recovers the password:

```
jacqueline
```

### 4. Mount the decrypted volume

`dislocker` failed to grab the VMK/FVEK directly, so `bdemount` (from `libbde-utils`) was used instead:

```bash
sudo apt install libbde-utils
mkdir ~/Desktop/mountpoint
sudo bdemount -p jacqueline bitlocker-1.dd ~/Desktop/mountpoint
```

This exposes a decrypted raw block device at `~/Desktop/mountpoint/bde1`.

### 5. Mount the inner filesystem

```bash
sudo mkdir -p /mnt/ctf_drive
sudo mount -o loop ~/Desktop/mountpoint/bde1 /mnt/ctf_drive
sudo ls -la /mnt/ctf_drive
```

The filesystem contains `flag.txt`.

### 6. Read the flag

```bash
sudo cat /mnt/ctf_drive/flag.txt
```

```
picoCTF{us3_b3tt3r_p4ssw0rd5_pl5!_3242adb1}
```

---

## Flag

```
picoCTF{us3_b3tt3r_p4ssw0rd5_pl5!_3242adb1}
```

---

## Tools Used

| Tool | Purpose |
|---|---|
| `file` | Identify the BitLocker FVE signature |
| `bitlocker2john` | Extract crackable hash from the VMK |
| `john` (John the Ripper) | Dictionary attack against the hash |
| `bdemount` | Decrypt & expose the BitLocker volume as a block device |
| `mount` | Mount the inner NTFS filesystem to read files |

## Lesson Learned

Even with a cryptographically strong scheme like BitLocker (AES-CCM), security ultimately depends on password strength. A password found in a common wordlist (`rockyou.txt`) can be cracked in minutes regardless of how strong the underlying encryption algorithm is.
