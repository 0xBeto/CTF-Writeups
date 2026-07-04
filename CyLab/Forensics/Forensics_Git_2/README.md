# Forensics Git 2

**Category:** Forensics
**Difficulty:** Medium
**Points:** 400
**Author:** LT 'syreal' Jones

## Challenge Description

> The agents interrupted the perpetrator's disk deletion routine. Can you recover this git repo?

**Hint:** We think the deletion was interrupted before any git objects were touched.

**Files provided:** `disk.img.gz`

## Tools Used

- `mmls`, `fls`, `icat` (The Sleuth Kit)
- `python3` (`zlib` module)
- `bash`

## Overview

The disk image contains an ext-based Linux filesystem. Somewhere on it is a git
repository (`killer-chat-app`) whose deletion was interrupted before the raw
git objects were unlinked — only some of the working tree / metadata was
touched. The goal is to manually reconstruct the git history directly from
the raw filesystem, without ever running `git` itself (the `.git/objects`
directory has to be walked and decompressed by hand).

## Step 1 — Locate the partition

```bash
gunzip disk.img.gz
mmls disk.img
```

`mmls` shows three partitions. The relevant Linux filesystem starts at sector
`1140736`. This offset is passed to every subsequent `fls`/`icat` call with
`-o 1140736`.

## Step 2 — Find the git repository

```bash
fls -pr -o 1140736 disk.img | grep "\.git"
```

The `-p` flag is important here — without it, `fls` only prints filenames,
not full paths, which makes it impossible to tell which `.git/objects/xx/yyyy`
hash a given inode belongs to.

This reveals the repo at:

```
home/ctf-player/Code/killer-chat-app/.git
```

and lists every object under `.git/objects/<xx>/<yyyy...>` together with its
inode number.

## Step 3 — Read the reflog to recover commit history

```bash
fls -r -o 1140736 disk.img | grep master        # -> inode 65710
icat -o 1140736 disk.img 65710
```

`.git/logs/refs/heads/master` is a plain-text reflog and survived intact. It
lists all six commits in order:

| # | Commit hash | Message |
|---|---|---|
| 1 | `2c0a9b2b15dce92f800393d5030c7454efc278ae` | Add netcat scripts (initial) |
| 2 | `26b809e0c41d8421f1126ed3a4eb06ad66e6d90a` | Add video game chat log |
| 3 | `5827632e046a80a1e0d7b4fc5c7800dd539baeaf` | Add TV show chat log |
| 4 | `e80b38b3322a5ba32ac07076ef5eeb4a59449875` | **Add secret hideout chat log** |
| 5 | `2151ef0ccc15aed1ab88e1afdc7484aaeff211c4` | Remove secret hideout log |
| 6 | `01533f718556a0e59f1467dae4fa462eed82c2a1` | Add random chat log |

Commit **#4** adds the file that gets deleted in commit **#5** — that deleted
file is almost certainly what we're after.

## Step 4 — Decompress git objects manually

Every git object (commit, tree, or blob) is just zlib-compressed data. Since
the object's filename **is** its SHA-1 hash (`<first 2 chars>/<remaining 38
chars>`), and `fls -pr` already gave us the inode for every hash, each object
can be pulled straight off the disk:

```bash
icat -o 1140736 disk.img <inode> > obj.bin
python3 -c "import zlib; print(zlib.decompress(open('obj.bin','rb').read()))"
```

Decompressing each of the six commit objects (by matching their hash to the
`fls -pr` listing) gives their `tree <hash>` line — the root tree for that
snapshot:

| Commit | Tree hash |
|---|---|
| Add netcat scripts | `5eb896e3ccd51175f66480cdb247fc45f3e8ac2d` |
| Add video game chat log | `201c707b43219a63c1d3499b29c7d539af079861` |
| Add TV show chat log | `6bf83de540f7d12cc3b683a83d69432e03d84509` |
| **Add secret hideout chat log** | **`ead27e2bd5a0fc22868ffb629a768f82dfcda11c`** |
| Remove secret hideout log | `6bf83de540f7d12cc3b683a83d69432e03d84509` (same tree as commit #3) |
| Add random chat log | `c931ae0868411e5f23656a2436e78a4c4699e18c` |

Note that commit #5's tree hash is *identical* to commit #3's — makes sense,
since removing the secret log just brings the `logs/` directory back to the
exact state it was in before it was added.

## Step 5 — Walk the tree for the secret hideout commit

A git tree object is a binary structure: repeated entries of
`<mode> <filename>\0<20 raw hash bytes>`. Because the hash bytes are raw
binary (not hex text), naive `str.decode()` calls throw `UnicodeDecodeError`
— you have to parse it at the byte level:

```python
import zlib
data = open('secret_tree.bin','rb').read()
decoded = zlib.decompress(data)
pos = decoded.find(b'\x00') + 1          # skip "tree <size>\0" header
while pos < len(decoded):
    end = decoded.find(b'\x00', pos)
    mode, filename = decoded[pos:end].decode().split(' ')
    pos = end + 1
    h = decoded[pos:pos+20].hex()
    pos += 20
    print(mode, h, filename)
```

Running this against the secret hideout commit's tree (inode for
`ead27e2bd5a0fc22868ffb629a768f82dfcda11c`) gives three entries:

```
100755  d7b4a371ebd23e682ffebc7ec355690fdc94fbd1   client
40000   22f7d0c9bd045563ae33bfacfbe46fe406a5b318   logs
100755  71fd2fafcd5ebd62fbf857769c92a91225ab3954   server
```

Decompressing the `logs` subtree (`22f7d0c9bd045563ae33bfacfbe46fe406a5b318`)
gives:

```
100644  aa1cc01687b4ec94faf9916c3fc6efd83f23b816   1.txt
100644  f150f0b963ab3ee95ba5656212abd76d7f2fed2e   2.txt
100644  7178644433e7cb6da3adf028f1c80d382a18e7b6   3.txt
```

Cross-referencing the `logs` trees of every other commit confirms `3.txt` is
the odd one out — it only ever appears in this one snapshot (later commits'
`logs` trees jump straight from `2.txt` to `4.txt`), which lines up exactly
with the "Add secret hideout" → "Remove secret hideout" commit pair.

## Step 6 — Extract the flag

```bash
icat -o 1140736 disk.img 65730 > secret_3.txt.bin   # hash 7178644433e7...
python3 -c "import zlib; print(zlib.decompress(open('secret_3.txt.bin','rb').read()).decode())"
```

Output:

```
blob 188Rex: Meet at the old arcade basement for the secret hideout.
Jay: Ask Rusty at the door and use password picoCTF{g17_r35cu3_16ac6bf3}.
Rex: Bring the decoder map so we can plan the route.
```

## Flag

```
picoCTF{g17_r35cu3_16ac6bf3}
```

## Key Takeaways

- Git objects are content-addressed: the SHA-1 hash *is* the filename
  (`objects/<first 2 hex chars>/<remaining 38 hex chars>`), so once you have
  a filesystem listing (`fls -pr`) mapping every path to an inode, you can
  pull any object directly with `icat` — no need for `git` to be installed
  or for the repo to be intact.
- A commit object only stores a `tree` hash and a `parent` hash — the actual
  file content always lives one or two levels down, inside tree → subtree →
  blob. Don't expect the flag inside the commit object itself.
- Tree objects are binary, not text: `<mode> <name>\0<20 raw hash bytes>`
  repeated. Parse them byte-by-byte instead of relying on `str.decode()` or
  naive splitting, or the raw hash bytes will corrupt the parse.
- When a file is deleted in git, its blob and every tree that ever referenced
  it are still sitting in `.git/objects` untouched, unless a `git gc` runs.
  Comparing the `logs` tree across every commit made the missing/skipped
  filename (`3.txt`) immediately obvious.
