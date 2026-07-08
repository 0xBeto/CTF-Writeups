# endianness-v2 — Forensics (Medium) — 300 pts

**Category:** Forensics
**Difficulty:** Medium
**Author:** Junias Bonou
**Event:** picoCTF 2024
**Tags:** `browser_webshell_solvable`

> Here's a file that was recovered from a 32-bits system that organized the bytes a weird way. We're not even sure what type of file it is.
>
> Download it here and see what you can get out of it.

---

## Overview

We're given a single unnamed file (`challengefile`) with no extension. The challenge description hints strongly at **byte-order (endianness) corruption**: a "32-bit system that organized the bytes a weird way" — meaning the file is very likely a normal file format whose bytes have been shuffled/swapped in groups, and needs to be reordered correctly before any tool can parse it.

---

## Steps

### 1. Identify the file

```bash
file challengefile
```
```
challengefile: data
```

No recognizable magic bytes — `file` can't identify it. `exiftool` and `binwalk` were tried too:

```bash
exiftool challengefile
binwalk challengefile
binwalk -e challengefile
```

`exiftool` gave a telling warning:
```
Warning: Processing JPEG-like data after unknown 1-byte header
```

This confirms the file is **JPEG-like**, but shifted/misaligned — consistent with a byte-swapping issue rather than random corruption.

### 2. Inspect the raw bytes

```bash
xxd challengefile | head -n 20
```
```
00000000: e0ff d8ff 464a 1000 0100 4649 0100 0001  ....FJ....FI....
00000010: 0000 0100 4300 dbff 0606 0800 0805 0607  ....C...........
```

A normal JPEG file starts with the magic bytes `FF D8 FF E0` (SOI marker + APP0 marker), followed by ASCII `JFIF`. Here we instead see:
```
E0 FF D8 FF 46 4A ...
```
Every pair of bytes looks reversed compared to a standard JPEG header (`FFD8 FFE0` → `E0FF D8FF`), and `JFIF` appears as `FJFI`. This is consistent with the bytes being swapped in **2-byte (16-bit) chunks** — a classic **endianness** issue, since a 32-bit system storing data in the "wrong" byte order for this context would produce exactly this kind of swap.

### 3. First attempt — swap bytes in pairs

Using `dd`'s `conv=swab` option, which swaps every pair of adjacent bytes:

```bash
dd if=challengefile of=fixed.image.jpg conv=swab
```

Checking the result:
```bash
xxd fixed.image.jpg | head -n 20
```
```
00000000: ffe0 ffd8 4a46 0010 0001 4946 0001 0100
```

Closer, but still not a valid JPEG header — the SOI/APP0 markers (`FFD8 FFE0`) are still in the wrong order relative to each other, even though each individual byte pair looks more "normal." A 2-byte swab wasn't enough — the corruption is at a **larger granularity**.

`file` and `exiftool` on this intermediate file confirmed it still wasn't valid:
```bash
file fixed.image.jpg      # data
exiftool fixed.image.jpg  # Error: File format error
```

### 4. Second attempt — full 4-byte word reversal

Since the challenge explicitly says the file came from a **32-bit system**, the natural granularity to test is **4-byte (32-bit) words**, not 2-byte pairs. A small Python one-liner reverses each 4-byte chunk of the original file completely:

```bash
python3 -c "import sys; data = open('challengefile', 'rb').read(); fixed = b''.join([data[i:i+4][::-1] for i in range(0, len(data), 4)]); open('real_fixed.jpg', 'wb').write(fixed)"
```

This reads the file, splits it into 4-byte groups, reverses each group internally, and writes the result to `real_fixed.jpg`.

### 5. Verify the fixed file

```bash
xxd real_fixed.jpg | head -n 20
```
```
00000000: ffd8 ffe0 0010 4a46 0001 4946 0001 0100
```

Now the header is correct: `FF D8 FF E0` (JPEG SOI + APP0 marker), followed by the proper `JFIF` string layout. Confirming with `file` and `exiftool`:

```bash
file real_fixed.jpg
```
```
real_fixed.jpg: JPEG image data
```

```bash
exiftool real_fixed.jpg
```
```
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
Warning                         : [minor] Skipped unknown 2 bytes after JPEG APP0 segment
```

The file is now a valid, openable JPEG (the "skipped 2 bytes" warning is just leftover artifact padding and doesn't prevent the image from rendering).

### 6. Open the image and read the flag

```bash
xdg-open real_fixed.jpg
```

The recovered image contains the flag rendered as text directly in the picture.

---

## Flag

```
picoCTF{cert!f1Ed_iNd!4n_s0rrY_3nDian_6d3ad08e}
```

---

## Tools Used

| Tool | Purpose |
|---|---|
| `file` | Identify (or fail to identify) the corrupted file format |
| `exiftool` | Warn about malformed JPEG-like structure, confirm final valid JPEG |
| `xxd` | Inspect raw hex bytes to diagnose the exact type of corruption |
| `binwalk` | Scan for embedded signatures (came up empty, since headers were scrambled) |
| `dd conv=swab` | First (insufficient) attempt — 2-byte pair swap |
| `python3` | Correctly reverse the file in 4-byte (32-bit word) chunks |
| `xdg-open` | View the recovered image to read the flag |

## Lesson Learned

Endianness issues don't always mean a simple 2-byte swap — the correct "chunk size" to reverse depends on the word size of the system that produced the corruption (here, 4 bytes for a 32-bit system). When a partial fix (`conv=swab`) gets you *closer* to a valid header but not all the way there, it's a strong signal to try a larger swap granularity rather than a completely different theory. Comparing the corrupted header byte-for-byte against the known-good magic bytes of the suspected format (`FF D8 FF E0` for JPEG) is the fastest way to reverse-engineer the exact transformation that was applied.
