# Timeline 0

**Category:** Forensics
**Difficulty:** Medium
**Points:** 100
**Author:** LT 'syreal' Jones

## Challenge Description

> Can you find the flag in this disk image? Wrap what you find in the picoCTF flag format.

**Hints:**
1. Create a Sleuthkit MAC timeline!
2. Sloppy timestomping can yield strange (very old) timestamps

**Files provided:** `partition4.img.gz`

## Tools Used

- `file`, `binwalk`
- `fls`, `mactime`, `icat` (The Sleuth Kit)
- `base64`

## Overview

The challenge hints point straight at **timestomping** (an anti-forensics
technique where a file's MAC timestamps — Modified/Accessed/Changed — are
manually altered to make it blend in or hide from timeline analysis). The
hint "sloppy timestomping" means the attacker didn't do it carefully — a
file will stand out because its timestamps are set to something absurd
(here, 1985), instead of being close to the surrounding files.

## Step 1 — Identify the image

```bash
gunzip partition4.img.gz
file partition4.img
```

```
partition4.img: Linux rev 1.0 ext4 filesystem data, ...
```

Unlike the full-disk images from the previous Git challenges, this one is a
**single ext4 partition** with no MBR/partition table — `mmls` correctly
returns nothing, since there's no partition table to parse. That means every
`fls`/`icat` call can skip the `-o <offset>` argument entirely.

## Step 2 — Build a MAC timeline

```bash
fls -f ext4 -r -m / partition4.img > bodyfile.txt
mactime -b bodyfile.txt > timeline.txt
```

- `fls -m /` produces a **bodyfile** (the raw MAC-time data for every file,
  in mactime's input format).
- `mactime -b` turns that bodyfile into a human-readable, chronologically
  sorted timeline.

## Step 3 — Spot the timestomped file

Instead of scrolling through the whole timeline, filtering `fls -l` output
directly for suspicious-looking short/garbled names works just as well:

```bash
fls -f ext4 -r -l partition4.img | grep "bcab"
```

```
r/r 4945:  bcab   1985-01-01 19:00:00 (EET)  1985-01-01 19:00:00 (EET)  1985-01-01 19:00:00 (EET)  1985-01-01 19:00:00 (EET)   41   0   0
```

A file named `bcab`, only 41 bytes, with **all four timestamps identical and
set to January 1st, 1985** — years before the rest of the filesystem's
activity (late 2025) — is exactly the "sloppy timestomping" the hint
describes. A real file rarely has every single MAC timestamp match exactly;
that uniformity is the signature of a script blindly setting one fixed date
on all fields.

## Step 4 — Extract and decode the file

```bash
icat -f ext4 partition4.img 4945
```

```
NzFtMzExbjNfMHU3MTEzcl9oM3JfNDNhMmU3YWYK
```

That's clearly base64. Decoding it:

```bash
echo "NzFtMzExbjNfMHU3MTEzcl9oM3JfNDNhMmU3YWYK" | base64 -d
```

```
71m311n3_0u7113r_h3r_43a2e7af
```

## Flag

```
picoCTF{71m311n3_0u7113r_h3r_43a2e7af}
```

## Key Takeaways

- `mmls` returning nothing isn't always an error — some images are a single
  raw filesystem with no partition table, in which case `fls`/`icat` are
  used without an `-o` offset at all.
- `fls -m /` + `mactime -b` is the standard Sleuthkit combo for building a
  full MAC timeline from a raw image — the fastest way to spot files that
  don't belong chronologically.
- Timestomping tools that blindly overwrite all four MAC timestamps with the
  same fixed value are easy to spot: real filesystem activity almost never
  produces four identical timestamps, especially ones from decades before
  everything else on the disk.
- Suspicious files hidden this way are often small, oddly-named, and encoded
  (here, plain base64) rather than containing the flag in cleartext.
