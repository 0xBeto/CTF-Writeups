# tunn3l vision — Forensics (Medium) — picoCTF 2020

<div align="center">

**Category:** Forensics · **Event:** picoCTF 2020

`bmp` `header-repair`

</div>

---

> We found this file. Recover the flag.

---

## Recon

```bash
file tunn3l_v1s10n
```
```
tunn3l_v1s10n: data
```

`file` can't identify it. Checking with `exiftool` gives a strong clue anyway:

```bash
exiftool tunn3l_v1s10n
```
```
File Type      : BMP
BMP Version    : Unknown (53434)
Image Width    : 1134
Image Height   : 306
Bit Depth      : 24
Red Mask       : 0x27171a23
...
```

It's a BMP, but the header fields are garbage — meaningless color masks, an unknown version, all signs of a **corrupted BMP header** rather than a corrupted image body.

---

## Repairing the header

A standard BMP header layout:

| Offset | Size | Field | Expected |
|:---:|:---:|---|---|
| 0–1 | 2 | Magic | `BM` |
| 2–5 | 4 | File size | matches actual file size |
| 10–13 | 4 | Pixel data offset (`bfOffBits`) | `54` for a 24-bit BMP with no palette |
| 14–17 | 4 | DIB header size (`biSize`) | `40` (`0x28`) for `BITMAPINFOHEADER` |
| 18–21 | 4 | Width | — |
| 22–25 | 4 | Height | — |

```bash
xxd -l 32 tunn3l_v1s10n
```
```
00000000: 424d 8e26 2c00 0000 0000 bad0 0000 bad0
00000010: 0000 6e04 0000 3201 0000 0100 1800 0000
```

Magic `BM` is intact and the file size looks plausible, but bytes 10–17 (the offset and DIB-header-size fields) are filled with junk (`ba d0`), which explains the bogus masks and version reported by `exiftool`.

### Step 1 — Fix the DIB header size

```bash
printf '\x00\x00' | dd of=tunn3l_v1s10n bs=1 seek=10 conv=notrunc
printf '\x28\x00' | dd of=tunn3l_v1s10n bs=1 seek=14 conv=notrunc
```

```bash
file tunn3l_v1s10n
```
```
tunn3l_v1s10n: PC bitmap, Windows 3.x format, 1134 x 306 x 24, ... bits offset 0
```

`file` now recognizes it as a valid BMP — but `bits offset 0` is wrong; pixel data can't start inside the header itself.

### Step 2 — Fix the pixel data offset

For a 24-bit BMP with a 40-byte `BITMAPINFOHEADER` and no color table, the pixel data always starts at byte `54` (`0x36`):

```bash
printf '\x36\x00\x00\x00' | dd of=tunn3l_v1s10n bs=1 seek=10 conv=notrunc
```

```bash
file tunn3l_v1s10n
```
```
tunn3l_v1s10n: PC bitmap, Windows 3.x format, 1134 x 306 x 24, ... bits offset 54
```

The header is now fully valid.

---

## First look — and a decoy

Opening the repaired file:

```bash
xdg-open tunn3l_v1s10n
```

The image renders, but only shows the **bottom half** — and it contains a decoy string sitting in plain sight:

```
notaflag{sorry}
```

Since the declared height (`306`) was clearly a corrupted/truncated value relative to the amount of actual pixel data in the file, the fix is to widen the declared height so the renderer reads further into the pixel buffer and reveals the rest of the image.

```bash
xxd -l 32 tunn3l_v1s10n
```
```
00000010: 0000 6e04 0000 3201 0000 0100 1800 0000
```

Bytes 22–23 hold the low bytes of the height field (currently `32 01` → `0x0132` = `306`).

```bash
printf '\x52\x03' | dd of=tunn3l_v1s10n bs=1 seek=22 conv=notrunc
```

This raises the declared height to `0x0352` = `850`, exposing the full image — including the real flag rendered across the top.

---

## Flag

```
picoCTF{qu1t3_a_v13w_2020}
```

*(`notaflag{sorry}` is a decoy embedded lower in the image — not the real flag.)*

---

## Tools

| Tool | Purpose |
|---|---|
| `file` / `exiftool` | Detect the file type and spot corrupted BMP header fields |
| `xxd` | Inspect the raw header bytes |
| `dd` + `printf` | Patch specific header fields (offset, DIB size, height) in place |
| `xdg-open` / `feh` | Render the repaired image to inspect visually |

## Note

`binwalk` and `zsteg` also flagged large amounts of "extra data" after the declared image end and several bit-planes that looked like OpenPGP key material — these turned out to be red herrings produced by the corrupted header itself (the file being misread past its true boundaries), not actual hidden payloads. The real fix was purely at the BMP header level.
