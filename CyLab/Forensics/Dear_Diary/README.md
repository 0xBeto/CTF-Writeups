# Dear Diary — Forensics (Medium) — 400 pts

**Category:** Forensics
**Difficulty:** Medium
**Author:** syreal
**Event:** picoCTF 2024
**Tags:** `disk`, `browser_webshell_solvable`

> If you can find the flag on this disk image, we can close the case for good!

**Hint:** If you're observing binary data raw in the terminal you may be misled about the contents of a block.

---

## Overview

We're given a raw disk image (`disk.flag.img`). The goal is to find a hidden flag on the filesystem. The hint warns that raw binary viewed directly can be misleading — a strong signal that the flag isn't sitting in a plain file, but scattered across **directory entry slack space** (leftover bytes from previously deleted/renamed directory entries that the live filesystem no longer references).

---

## Steps

### 1. List the contents of the suspicious directory

After mapping partitions with `mmls` and locating the data partition offset (`1140736`), a directory of interest was found at inode `1842`. Listing it with `fls` (including allocated **and** deleted entries with `-a`, recursively with `-r`):

```bash
fls -o 1140736 -a -r disk.flag.img 1842
```

```
d/d 1842:	.
d/d 204:	..
r/r 1843:	force-wait.sh
r/r 1844:	innocuous-file.txt
r/r 1845:	its-all-in-the-name
```

Three regular files stand out:
- `force-wait.sh` — sounds like a delay/anti-automation script
- `innocuous-file.txt` — deliberately named to look boring
- `its-all-in-the-name` — a strong hint pointing at *filenames* being significant, not file contents

### 2. Check the actual file contents (dead end)

```bash
icat -o 1140736 disk.flag.img 1843
```
```
#!/bin/ash

sleep 10
```

`force-wait.sh` is just a script that sleeps — likely a red herring / anti-automation trick with no real value for the flag.

```bash
icat -o 1140736 disk.flag.img 1844
```

`innocuous-file.txt` (inode 1844) returned **empty content** — the file itself is a zero-byte file. This confirms the hint: the flag isn't stored as file *content* at all. Combined with the file named `its-all-in-the-name`, this points toward the flag being hidden in **filenames** used across multiple deleted/renamed versions of this file, preserved as slack space in the directory's data block.

### 3. Inspect the directory inode's metadata

```bash
istat -o 1140736 disk.flag.img 1842
```
```
inode: 1842
Allocated
Group: 0
Generation Id: 3140876723
uid / gid: 0 / 0
mode: drwxr-xr-x
Flags: Extents, 
size: 1024
num of links: 2

Inode Times:
Accessed:	2024-02-17 22:12:06.779061969 (+03)
File Modified:	2024-02-17 22:12:05.249056065 (+03)
Inode Modified:	2024-02-17 22:12:05.249056065 (+03)
File Created:	2024-02-17 22:05:21.830009163 (+03)

Direct Blocks:
15602 
```

The directory's actual on-disk data lives in a single 4KB block: block number `15602`.

### 4. Dump the raw directory block

```bash
dd if=disk.flag.img bs=512 skip=$((1140736 + 15602 * 8)) count=8 | xxd | cat -v
```

This seeks directly to the raw directory block on disk (converting the block number to a 512-byte sector offset: `partition_offset + block_number * 8`, since ext4 blocks here are 4096 bytes = 8 sectors) and dumps its full 4KB content as hex.

The dump turned out to be much larger and noisier than expected — it actually revealed **unrelated leftover data** (strings from an OpenSSL CMP client library, error messages, etc.) that had nothing to do with the flag. This was a dead end caused by mis-locating the true directory block — the real fragments were still further down the filesystem, in different reused/slack blocks belonging to the same directory's history.

### 5. Search the full inode 8 content for the real fragments

Instead of chasing a single block, the raw content of the **directory inode itself** (inode `8`, the parent structure that had previously held many renamed copies of the same file at different points in time) was dumped in full and filtered for recognizable filename fragments:

```bash
icat -o 1140736 disk.flag.img 8 | xxd | grep ".txt" -A3
```

Every match centers on the tail end of `...s-file.txt` (i.e., `innocuous-file.txt`), each time followed 3 lines later by a short fragment of a **different filename** that the file was renamed to and from, over and over, before ending up as `innocuous-file.txt`. Each of those historical filenames carried a small piece of the flag:

```
001fdc70: 7069 6300 ...                          → pic
001ff450: a803 0301 6f43 5400 ...                 → oCT
00201860: 467b 3100 ...                            → F{1
00203c50: a803 0301 5f35 3300 ...                  → _53
00206060: 335f 6e00 ...                             → 3_n
00207850: a803 0301 346d 3300 ...                   → 4m3
00209c60: 355f 3800 ...                              → 5_8
0020b450: a803 0301 3064 3200 ...                    → 0d2
0020d860: 3462 3300 ...                               → 4b3
0020fc50: a803 0201 307d 0000 ...                     → 0}
```

Each of these was a **short-lived directory entry name** the ext4 directory-entry hash table (dx_root/htree) had reused across successive renames of the same inode — the old entry names weren't fully wiped when the file was renamed, leaving fragments behind as slack in the directory's hashed-tree blocks.

### 6. Reassemble the flag

Concatenating the fragments in the order they were found:

```
pic + oCT + F{1 + _53 + 3_n + 4m3 + 5_8 + 0d2 + 4b3 + 0}
= picoCTF{1_533_n4m35_80d24b30}
```

---

## Flag

```
picoCTF{1_533_n4m35_80d24b30}
```

---

## Tools Used

| Tool | Purpose |
|---|---|
| `mmls` (Sleuth Kit) | Map partition offsets without mounting the image |
| `fls` (Sleuth Kit) | List directory entries (including deleted, `-a`) by inode, without mounting |
| `icat` (Sleuth Kit) | Extract raw inode/file content directly from the image |
| `istat` (Sleuth Kit) | Inspect inode metadata (timestamps, block pointers) |
| `dd` | Seek to and dump a specific raw block by sector offset |
| `xxd` | Render binary as hex + ASCII |
| `grep -A3` | Filter matches with trailing context lines to catch nearby fragments |

## Lesson Learned

Renaming a file on ext4 doesn't necessarily scrub the old directory entry name from disk immediately — especially in hashed-tree (`htree`) directories, old entry name fragments can persist as slack space alongside the currently active entry. A file with empty/meaningless content (`innocuous-file.txt`) combined with a suggestively-named sibling (`its-all-in-the-name`) was the clue that the real evidence was in the **history of directory entries**, not file contents — recoverable only by dumping the raw inode/block data and hunting for leftover fragments, rather than relying on `icat`'s "logical" view of a file's content.
