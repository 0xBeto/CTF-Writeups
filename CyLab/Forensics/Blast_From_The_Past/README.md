# Blast from the past — Forensics (Medium) — 300 pts

**Category:** Forensics
**Difficulty:** Medium
**Author:** syreal
**Tags:** `browser_webshell_solvable`, `metadata`

> The judge for these pictures is a real fan of antiques. Can you age this photo to the specifications?
>
> Set the timestamps on this picture to **1970:01:01 00:00:00.001+00:00** with as much precision as possible for each timestamp. Any timezone is acceptable as long as the time is equivalent. For timestamps without a timezone adjustment, put them in GMT time (+00:00). The checker program provides the timestamp needed for each.

**Hint:** Exiftool is really good at reading metadata, but you might want to use something else to modify it.

---

## Overview

We're given a JPEG picture and two netcat services:

- `nc mimas.picoctf.net 49274 < picture.jpg` — submit the modified picture
- `nc mimas.picoctf.net 58090` — check the current picture against 7 required EXIF timestamp tags

The goal is to edit **7 different metadata timestamp fields** in the JPEG so they all read `1970:01:01 00:00:00` (with sub-second precision `.001` where required) until the remote checker confirms all 7 tags match.

---

## Steps

### 1. Baseline check

First, submit the original picture to the checker to see what tags are being validated:

```bash
nc mimas.picoctf.net 58090
```

The checker walks through the tags one by one, printing the tag name, the expected value, and the found value. Tag 1 was:

```
Checking tag 1/7
Looking at IFD0: ModifyDate
Looking for '1970:01:01 00:00:00'
Found: 2023:11:20 15:46:23
Oops! That tag isn't right. Please try again.
```

So the file's original `ModifyDate` still had the real capture date.

### 2. Rewrite the standard EXIF timestamps

Using `exiftool`, the standard timestamp fields (`ModifyDate`, `DateTimeOriginal`, `CreateDate`, and their sub-second composites) can be set directly:

```bash
exiftool -AllDates="1970:01:01 00:00:00.001" final_fixed.jpg
```

*(Illustrative — in practice this was done tag-by-tag / iteratively while checking against the remote validator, adjusting sub-second precision as needed for the `SubSecCreateDate`, `SubSecDateTimeOriginal`, and `SubSecModifyDate` composite tags.)*

Resubmitting to the check service confirmed tags 1–6 (the standard `IFD0`/`ExifIFD`/`Composite` date fields) now passed:

```
Checking tag 1/7 ... Great job, you got that one!
Checking tag 2/7 ... Great job, you got that one!
Checking tag 3/7 ... Great job, you got that one!
Checking tag 4/7 ... Great job, you got that one!
Checking tag 5/7 ... Great job, you got that one!
Checking tag 6/7 ... Great job, you got that one!

Checking tag 7/7
Looking at Samsung: TimeStamp
Looking for '1970:01:01 00:00:00.001+00:00'
Found: 2023:11:20 20:46:21.420+00:00
Oops! That tag isn't right. Please try again.
```

### 3. The tricky tag — Samsung TimeStamp

Tag 7/7, `Samsung:TimeStamp`, is **not a standard EXIF tag** — it lives in a proprietary Samsung trailer appended to the JPEG (this is the hint: *"you might want to use something else to modify it"*, since `exiftool` can't write unknown/vendor-specific tags out of the box):

```bash
exiftool -Samsung:0x01a4="1970:01:01 00:00:00.001+00:00" final_fixed.jpg -o brand_new.jpg
```

This attempt produced a warning:
```
Warning: Tag 'Samsung:0x01a4' is not defined
```
meaning exiftool copied the file but did **not** actually write the tag — confirmed by resubmitting and seeing tag 7 was still unchanged.

### 4. Locate the raw bytes of the Samsung timestamp

Since `exiftool` can't write this proprietary tag, the fix is to **patch the raw bytes directly** in the file. First, the tag's exact location and encoding were found using verbose exiftool output:

```bash
exiftool -v3 final_fixed.jpg | grep -B5 -A5 'TimeStamp'
```

This revealed the Samsung trailer structure:
```
SamsungTrailer_0x0a01Name = Image_UTC_Data
TimeStamp = 1700513181420
- Tag '0x0a01' (13 bytes):
  2b8300: 31 37 30 30 35 31 33 31 38 31 34 32 30  [1700513181420]
```

The key insight: the Samsung `TimeStamp` tag is stored as a **plain ASCII string of a Unix epoch timestamp in milliseconds** (`1700513181420`), not a formatted EXIF date string. It's 13 ASCII digits, at a fixed byte offset in the file.

### 5. Patch the raw bytes to set epoch time to zero

Since `1970:01:01 00:00:00.001` corresponds to Unix epoch `1` millisecond (i.e., `0000000000001` padded to keep the same 13-character length), the string was overwritten directly at that offset using `dd`:

```bash
printf "0000000000001" | dd of=final_fixed.jpg bs=1 seek=2851584 conv=notrunc
```

- `bs=1 seek=2851584` — seek to the exact byte offset of the timestamp field found via the verbose exiftool dump
- `conv=notrunc` — critical: prevents `dd` from truncating the rest of the file, since we're overwriting in place without changing file size (13 digits in, 13 digits out)

Verifying the patch:

```bash
exiftool -Samsung:TimeStamp final_fixed.jpg
```

```
Time Stamp                      : 1970:01:01 02:00:00.001+02:00
```

This is `1970:01:01 00:00:00.001+00:00` expressed in a different (but equivalent) timezone — exactly what the challenge description said is acceptable.

### 6. Final submission

```bash
nc -w 2 mimas.picoctf.net 49274 < final_fixed.jpg
nc mimas.picoctf.net 58090
```

All 7 tags now passed:

```
Checking tag 7/7
Timezones do not have to match, as long as it's the equivalent time.
Looking at Samsung: TimeStamp
Looking for '1970:01:01 00:00:00.001+00:00'
Found: 1970:01:01 00:00:00.001+00:00
Great job, you got that one!

You did it!
picoCTF{71m3_7r4v311ng_p1c7ur3_12e0c36b}
```

---

## Flag

```
picoCTF{71m3_7r4v311ng_p1c7ur3_12e0c36b}
```

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nc` (netcat) | Submit the picture and query the remote checker |
| `exiftool` | Read/write standard EXIF metadata timestamps |
| `exiftool -v3` | Dump verbose/raw tag info to locate the proprietary Samsung tag's byte offset |
| `dd` | Patch the raw ASCII epoch-timestamp bytes directly, since exiftool doesn't support writing this vendor-specific tag |

## Lesson Learned

Not all metadata fields are exposed or writable through standard tools like `exiftool` — proprietary vendor trailers (like Samsung's) may store data in non-standard formats (e.g., a raw millisecond epoch string instead of a formatted date). When a high-level tool refuses to write a tag, dropping down to raw byte-level inspection and patching (`exiftool -v3` for reconnaissance + `dd` for surgical edits) is a reliable fallback, as long as the replacement data preserves the original byte length to avoid corrupting the file structure.
