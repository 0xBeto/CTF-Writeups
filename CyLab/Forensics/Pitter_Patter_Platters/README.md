# Pitter, Patter, Platters — Forensics (Medium) — 250 pts

**Category:** Forensics
**Difficulty:** Medium
**Author:** syreal
**Event:** picoCTF 2020 Mini-Competition

> 'Suspicious' is written all over this disk image.

**Hints:**
1. It may help to analyze this image in multiple ways: as a blob, and as an actual mounted disk.
2. Have you heard of slack space? There is a certain set of tools that now come with Ubuntu that I'd recommend for examining that disk space phenomenon…

---

## Overview

We're given a raw filesystem image (`suspicious.dd.sda1`). The hints point in two directions at once: treat the image both as a raw blob **and** as a real filesystem, and specifically look into **slack space** using Sleuth Kit-style tools. This suggests the flag isn't a plain visible file, but something recoverable only by digging into unallocated/slack areas of the disk.

---

## Steps

### 1. Identify and repair the filesystem

```bash
file suspicious.dd.sda1
```
```
suspicious.dd.sda1: Linux rev 1.0 ext3 filesystem data, UUID=fc168af0-183b-4e53-bdf3-9c1055413b40 (needs journal recovery)
```

The image is a Linux ext3 filesystem that needs journal recovery before it can be reliably parsed. Running a filesystem check to replay/recover the journal:

```bash
e2fsck -y suspicious.dd.sda1
```
```
e2fsck 1.47.4 (6-Mar-2025)
suspicious.dd.sda1: recovering journal
suspicious.dd.sda1: clean, 70/8032 files, 17303/32096 blocks
```

### 2. List the filesystem contents

Using Sleuth Kit's `fls` to walk the filesystem tree (recursively, with full paths) without mounting it:

```bash
fls -r -p suspicious.dd.sda1
```

The listing shows a fairly standard **Tiny Core Linux** install (GRUB boot files, `tce/optional` packages like nginx, openssh, openssl, etc.), plus two files that stand out:

- `suspicious-file.txt` (inode `12`) — the obviously-named "suspicious" file
- `tce/mydata.tgz` (inode `4018`) — a compressed archive, worth extracting

### 3. Read the suspicious file (a pointer, not the flag)

```bash
icat suspicious.dd.sda1 12
```
```
Nothing to see here! But you may want to look here -->
```

This is clearly a taunt/pointer rather than the flag itself — it hints that the *real* content is somewhere else, likely hidden right after this file on disk (foreshadowing the slack-space technique used later).

### 4. Extract and inspect the archive

```bash
icat suspicious.dd.sda1 4018 > mydata.tgz
mkdir extracted_data
tar -xvzf mydata.tgz -C extracted_data
```

This unpacks a full Tiny Core Linux persistence backup (`/opt`, `/home/tc`, SSH host keys, `/etc/passwd`, `/etc/shadow`, nginx config, etc.) — a realistic simulated home directory backup.

### 5. Check shell history for clues

```bash
cat extracted_data/home/tc/.ash_history
```

The history shows a typical Tiny Core Linux setup session: installing SSH, nginx, openssl, and various TCE extensions, editing config files, and persisting changes with `filetool.sh -b`. Nothing in here reveals the flag directly, but it confirms the disk is a legitimate simulated system rather than a stripped-down decoy — reinforcing that the flag is hidden via filesystem-level tricks (slack space) rather than in a normal file.

### 6. Search slack space across the whole image (first attempt)

Following hint #2, Sleuth Kit's `blkls -s` was used to dump **all slack space** across the entire filesystem in one go:

```bash
blkls -s suspicious.dd.sda1 > slack_data.txt
strings slack_data.txt | grep "pico"
```

This returned nothing — the flag wasn't found by searching *all* slack space blindly with a `pico` string match.

### 7. Target the slack space of the specific suspicious inode

Since `suspicious-file.txt` (inode `12`) explicitly hinted "look here," its metadata was inspected directly with `istat` to find its exact data block:

```bash
istat suspicious.dd.sda1 12
```
```
inode: 12
Allocated
...
size: 55
num of links: 1
...
Direct Blocks:
2049
```

The file's logical size is only 55 bytes, but it occupies a full disk block (block `2049`) — meaning there's unused slack space *after* the visible 55 bytes of text, within that same block, that `icat` doesn't show (since `icat` only returns the logical file size).

### 8. Dump the raw block content (including slack space)

Using `blkcat` to read the **entire physical block**, not just the logical file content:

```bash
blkcat suspicious.dd.sda1 2049
```
```
Nothing to see here! But you may want to look here -->
}3986312f_3<_|Lm_111t5_3b{FTCocip
```

Beyond the visible taunt message, the leftover slack space in that same block contains a garbled string — exactly the "look here" the file's message was hinting at.

### 9. Reverse the string to recover the flag

The trailing string is clearly the flag written **backwards**. Reversing it:

```bash
echo '}3986312f_3<_|Lm_111t5_3b{FTCocip' | rev
```
```
picoCTF{b3_5t111_mL|_<3_f2136893}
```

---

## Flag

```
picoCTF{b3_5t111_mL|_<3_f2136893}
```

---

## Tools Used

| Tool | Purpose |
|---|---|
| `file` | Identify the ext3 filesystem and detect it needs journal recovery |
| `e2fsck -y` | Recover/replay the ext3 journal so the filesystem can be parsed cleanly |
| `fls` (Sleuth Kit) | List filesystem entries recursively without mounting |
| `icat` (Sleuth Kit) | Extract the logical content of files by inode |
| `tar` | Unpack the extracted `.tgz` archive |
| `istat` (Sleuth Kit) | Inspect inode metadata to find the exact data block number |
| `blkls -s` (Sleuth Kit) | Dump slack space across the whole image (broad, unsuccessful attempt) |
| `blkcat` (Sleuth Kit) | Dump the full raw content of a specific block, including slack space beyond the logical file size |
| `rev` | Reverse the recovered garbled string to read the flag |

## Lesson Learned

A file's "size" as reported by the filesystem is only the *logical* size — the underlying disk block is usually larger (e.g., 4KB), and any bytes between the end of the logical file and the end of the block ("slack space") can retain leftover data that standard tools like `icat`/`cat` never show. When a broad slack-space sweep across an entire filesystem (`blkls -s`) doesn't turn up results, narrowing down to the slack space of a *specific, deliberately-suspicious file* (found via `istat` + `blkcat`) is a much more targeted and effective approach.
