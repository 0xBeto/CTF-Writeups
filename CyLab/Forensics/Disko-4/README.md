# DISKO 4 - picoCTF 2026

**Category:** Forensics
**Difficulty:** Medium
**Points:** 200
**Author:** Darkraicg492

## Challenge Description

> Can you find the flag in this disk image? This time I deleted the file! Let see you get it now!

We're given a disk image and told the flag file was deleted from it - so this is a classic **file carving / deleted file recovery** challenge.

## Solution

### 1. Identify the file

First step, as always, is to check what kind of file we're actually dealing with:

```bash
kali@sNaKe:~/Desktop$ file disko-4.dd.gz
disko-4.dd.gz: gzip compressed data, was "disko-4.dd", last modified: Wed Feb  4 21:55:31 2026, from Unix, original size modulo 2^32 104857600
```

It's a gzip-compressed disk image. Let's decompress it:

```bash
kali@sNaKe:~/Desktop$ gunzip disko-4.dd.gz
kali@sNaKe:~/Desktop$ file disko-4.dd
disko-4.dd: DOS/MBR boot sector, code offset 0x58+2, OEM-ID "mkfs.fat", Media descriptor 0xf8, sectors/track 32, heads 8, sectors 204800 (volumes > 32 MB), FAT (32 bit), sectors/FAT 1576, serial number 0x49838d0b, unlabeled
```

We now have a raw **FAT32** disk image (`disko-4.dd`).

### 2. Explore the filesystem

Tried `mmls` to look for a partition table, but it returned nothing - meaning this is a raw filesystem image without a partition table, not a full disk with multiple partitions:

```bash
kali@sNaKe:~/Desktop$ mmls disko-4.dd
```
*(no output)*

So we go straight to `fls` from **The Sleuth Kit** to list the filesystem contents directly:

```bash
kali@sNaKe:~/Desktop$ fls disko-4.dd
d/d 4:	log
v/v 3225859:	$MBR
v/v 3225860:	$FAT1
v/v 3225861:	$FAT2
V/V 3225862:	$OrphanFiles
```

There's a `log` directory, plus the usual FAT virtual entries. Since the challenge says the file was **deleted**, a normal `fls` listing won't show it - we need to include deleted/orphan entries.

### 3. Recover the deleted file

Using `fls -r -d` to recursively list **deleted** files:

```bash
kali@sNaKe:~/Desktop$ fls -r -d disko-4.dd
r/r * 522629:	log/messages
r/r * 532021:	log/dont-delete.gz
```

The `*` marks these as deleted entries. `log/dont-delete.gz` is exactly the kind of ironically-named file we're looking for.

### 4. Extract the file content

Using `icat` to dump the deleted file's content by its inode number:

```bash
kali@sNaKe:~/Desktop$ icat disko-4.dd 532021 > dont-delete.gz
```

The file was still gzip-compressed, so:

```bash
kali@sNaKe:~/Desktop$ gunzip dont-delete.gz
kali@sNaKe:~/Desktop$ cat dont-delete
Here is your flag
picoCTF{d3l_d0n7_h1d3_w3ll_3da42114}
```

## Flag

```
picoCTF{d3l_d0n7_h1d3_w3ll_3da42114}
```

## Key Takeaways

- Deleted files on a FAT filesystem aren't actually erased - their directory entries are just marked as unused, and the data often remains on disk until overwritten.
- `fls -r -d` is the go-to command to recursively list deleted entries in an image.
- `icat` can extract file content directly by inode number, even for deleted files, as long as the data blocks haven't been overwritten.
- Always double check extracted files with `file` - a "deleted and hidden" flag was still sitting inside a gzip archive.
