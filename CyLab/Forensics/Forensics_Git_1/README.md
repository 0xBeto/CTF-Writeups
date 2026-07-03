Markdown
# Write-up: Forensics Git 1 - CyLab

## Challenge Overview
- **Category:** Forensics
- **Platform:** CyLab Security Academy
- **Difficulty:** Medium
- **Points:** 300

## Objective
Recover a flag hidden within a disk image by investigating the Git version control system history.

---

## Investigation Methodology

### 1. Partition Analysis
First, analyze the disk image to identify the partition structure. The `mmls` utility is used to locate the starting sector of the file system.

```bash
mmls disk.img
Observation: The Linux partition (type 0x83) begins at sector 1140736.

2. File System Traversal
Using the identified offset, use fls to traverse the directory structure and locate the .git directory.

Bash
fls -o 1140736 disk.img 64770/64771/65663/65664/65665
Path: /home/ctf-player/Code/secrets/.git

3. Git History Exploration
Inspect the repository's commit logs located at inode 65704 to identify the changes made to the repository.

Bash
icat -o 1140736 disk.img 65704
Findings: The logs reveal two distinct commits: Add flag and Remove flag. To recover the data, target the initial commit.

4. Data Recovery
To recover the deleted flag.txt, extract the objects from the Git history using icat and decompress them using zlib via Python.

A. Extracting the Initial Commit Object:

Bash
icat -o 1140736 disk.img 65700 > commit1
python3 -c "import zlib;print(zlib.decompress(open('commit1','rb').read()).decode())"
B. Extracting and Decoding the Blob:
After identifying the tree hash and the corresponding blob for flag.txt (inode 65695), retrieve the final content:

Bash
icat -o 1140736 disk.img 65695 > blob
python3 -c "import zlib;print(zlib.decompress(open('blob','rb').read()).decode())"

Flag
picoCTF{g17_r3m3mb3r5_d4ddf904}
