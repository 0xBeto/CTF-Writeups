# Milkslap — Forensics (Medium) — 200 pts

<div align="center">

**Category:** Forensics · **Author:** James Lynch · **Event:** picoCTF 2021

</div>

---

> 🥛

**Hint:** Look at the problem category.

---

## Recon

The challenge points at a live web instance. Mirroring the whole site locally for offline analysis:

```bash
wget -r -l1 http://wily-courier.picoctf.net:62527/
```

This pulls down three files:

```
index.html
script.js
style.css
```

### index.html

```bash
cat index.html
```

The page is a **"MilkSlap"** interactive widget — explicitly credited as inspired by [eelslap.com](http://eelslap.com), a novelty site where moving the mouse over an image scrubs through an animation.

### style.css

```bash
cat style.css
```

```css
#image {
  height: 720px;
  background-image: url(concat_v.png);
  background-position: 0 0; }
```

The interactive image is a single background image, `concat_v.png`, positioned via CSS `background-position`.

### script.js

```bash
cat script.js
```

```js
var frame_ht = 720;
...
background_y = -1 * frame_ht * (frame_num + 1);
image.style.backgroundPositionY = background_y.toString() + "px";
```

This confirms the mechanism: `concat_v.png` is a **vertical sprite sheet** — a single tall image made of many stacked 720px-high frames — and mouse movement scrubs through frames by shifting the background's Y position. This matches the "problem category" hint: the interesting content is baked into the image itself as a set of frames, not just a static picture.

---

## Fetching the sprite sheet

```bash
wget http://wily-courier.picoctf.net:62527/concat_v.png
file concat_v.png
```
```
concat_v.png: PNG image data, 1280 x 47520, 8-bit/color RGB, non-interlaced
```

A 47,520px-tall image — consistent with dozens of stacked 720px animation frames, way too large to view normally (image viewers like `eog` refuse to render it).

```bash
exiftool concat_v.png
```

No useful embedded metadata — just standard PNG dimensions and encoding info.

---

## First attempts (dead ends)

Given the "Forensics" category and a suspiciously oversized image, steganography was the obvious next angle.

```bash
binwalk -e concat_v.png
```
Only found the PNG's own zlib-compressed IDAT stream — nothing embedded as a separate file.

```bash
zsteg -a concat_v.png
```
Crashed with a Ruby stack overflow (`SystemStackError`) — the image is too large/tall for `zsteg`'s naive scanline recursion to handle in one pass. Narrowing to specific bit planes also crashed the same way:

```bash
zsteg -E "b1,rgb,lsb,xy" concat_v.png
zsteg -E "b1,bgr,lsb,xy" concat_v.png
```

```bash
strings concat_v.png | grep -E "picoCTF"
```
No match — as expected for image data, since raw LSB-encoded steganography doesn't produce contiguous printable strings.

A custom Python script scanning the **combined RGB LSBs across every pixel** (treating R, G, B bits as one continuous bitstream, tried both normal and reversed bit order) also came up empty:

```bash
python3 -c "
from PIL import Image
...
for y in range(h):
    for x in range(w):
        r, g, b = pixels[x, y]
        bits.extend([r&1, g&1, b&1])
...
"
```
```
[-] Flag not found in standard LSB layers.
```

---

## Success — isolating individual color channels

Since interleaving all three channels together didn't work, the next step was checking each color channel's LSB plane **separately** rather than combined:

```python
if check_bits(r_bits, 'Red Channel'): sys.exit()
if check_bits(g_bits, 'Green Channel'): sys.exit()
if check_bits(b_bits, 'Blue Channel'): sys.exit()
```

```
[*] Checking individual channels...

[+] Found in Blue Channel (Normal): picoCTF{imag3_m4n1pul4t10n_sl4p5}
```

The flag was encoded purely in the **LSBs of the blue channel**, read in normal bit order — mixing in the red/green LSBs (as the earlier combined-channel attempt did) corrupted the bitstream and hid the message.

---

## Flag

```
picoCTF{imag3_m4n1pul4t10n_sl4p5}
```

---

## Tools

| Tool | Purpose |
|---|---|
| `wget -r -l1` | Mirror the challenge's static site locally |
| `file` / `exiftool` | Identify the oversized PNG sprite sheet and check metadata |
| `binwalk` | Rule out embedded/appended file signatures |
| `zsteg` | Attempted automated LSB steganography detection (crashed on this large image) |
| `strings` / `grep` | Quick check for a plaintext flag |
| Python + Pillow (`PIL`) | Custom LSB extraction, tried combined and per-channel bit planes |

## Note

Generic LSB scanners can fail on very large or unusually shaped images (here, a 47,520px-tall sprite sheet), so falling back to a custom script gives full control over how bits are read. The key fix here was isolating **one color channel at a time** instead of interleaving R, G, and B LSBs together — combined-channel extraction is a common default assumption that doesn't always match how the data was actually embedded.
