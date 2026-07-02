# CyLab Security Academy - Timeline 1 Writeup

*(formerly picoCTF)*

**Category:** Forensics | **Difficulty:** Medium | **Points:** 300
**Author:** LT 'syreal' Jones

## Challenge
Find a hidden flag inside a disk image (`partition4.img`) and wrap it in `picoCTF{...}` format.

## Tools Used
- `file` — identify filesystem type
- `fls` (Sleuthkit) — list files, generate MAC timeline
- `mactime` (Sleuthkit) — parse timeline into readable format
- `icat` (Sleuthkit) — extract file content by inode number
- `base64` — decode the final payload

## Solution Steps

1. **Identify the image:**

file partition4.img
   → Confirmed it's a Linux ext4 filesystem.

2. **Generate a MAC timeline:**
fls -r -m / partition4.img > timeline.body
mactime -b timeline.body > real_timeline.txt

3. ** Trap to avoid:** One of the hints says to filter by `macb`. Grepping literally for the string `"macb"` just returns a harmless kernel driver file (`macb.ko.gz`) — a decoy. The real clue is the `macb` timestamp flag pattern (Modified/Accessed/Changed/Birth all at once), not the literal word.

4. **Spot the anti-forensic action:**
   Searching the timeline around the shutdown sequence (`shred`, `poweroff`) revealed a suspicious file `/etc/chat` (inode 32716) modified seconds before the wipe attempt:
grep "00:50:" real_timeline.txt

5. **Extract the file content directly by inode:**
icat partition4.img 32716
   → Output: `NTczNDE3aDEzcl83aDRuXzdoM18xNDU3XzU4NTI3YmIyMjIK`

6. **Decode Base64:**
echo "NTczNDE3aDEzcl83aDRuXzdoM18xNDU3XzU4NTI3YmIyMjIK" | base64 -d
   → `573417h13r_7h4n_7h3_1457_58527bb222`

## Flag
picoCTF{573417h13r_7h4n_7h3_1457_58527bb222}

## Key Takeaway
Even after anti-forensic wiping (`shred`), file content can survive at the inode level if the underlying blocks haven't been overwritten. Timeline analysis (`mactime`) is essential for spotting suspicious activity around deletion events.
S
