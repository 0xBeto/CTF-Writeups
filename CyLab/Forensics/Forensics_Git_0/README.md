# Forensics Git 0

## Challenge Description
Can you find the flag in this disk image?

## Methodology
1. **Disk Analysis:** - Decompressed the disk image using `gunzip disk.img.gz`.
   - Used `mmls disk.img` to inspect the partition table and identify offsets.
2. **File System Navigation:** - Explored the file system using `fls` with the offset `-o 1140736`.
   - Navigated to `/home/ctf-player/Code/secrets/`.
3. **Data Retrieval:** - Identified a hidden Git repository.
   - Used `icat -o 1140736 disk.img 65693` to extract the `COMMIT_EDITMSG` file.
4. **Flag Extraction:** - Found the flag in the commit message.

## Flag
picoCTF{g17_1n_7h3_d15k_041217d8}
