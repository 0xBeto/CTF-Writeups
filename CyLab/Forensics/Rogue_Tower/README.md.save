Markdown
# CyLab Security Academy - Rogue Tower Writeup
*(Challenge Platform: CyLab Security Academy)*

**Category:** Forensics / Network Analysis
**Difficulty:** Medium (300 pts)

## Challenge Description
A suspicious cell tower has been detected in the network. Analyze the captured network traffic to identify the rogue tower, find the compromised device, and recover the exfiltrated flag.

---

## Solution Steps

### 1. Inspecting the PCAP (Finding Anomalies)
We started by verifying the general details of `rogue_tower.pcap`. Running basic packet analysis showed that one specific device (`10.100.204.98`) was communicating with a suspicious external IP address (`198.51.100.205`), while all other normal devices on the network interacted with the legitimate gateway `198.51.100.7`. This was our first indicator of a rogue connection.

### 2. Extracting the IMSI & Rogue Cell ID
Following the clues, we filtered the HTTP traffic to inspect the `User-Agent` configurations:
```bash
tshark -r rogue_tower.pcap -Y "http.request" -T fields -e http.user_agent
Result:

Plaintext
MobileDevice/1.0 (IMSI:310410728284734; CELL:92771)
The normal, legitimate towers used CELL:16274. A separate broadcast packet on UDP port 55000 explicitly broadcasted an unauthorized network signature:

Plaintext
UNAUTHORIZED-TEST-NETWORK PLMN=00101 CELLID=92771
This confirmed 92771 as the Rogue Cell Tower (IMSI-Catcher).

3. Gathering the Exfiltrated Data
We isolated all HTTP POST requests sent by the victim device to the malicious server (198.51.100.205) on the endpoint /upload:

Bash
tshark -r rogue_tower.pcap -Y "http.request.method==POST" -T fields -e http.file_data
Extracted Hex Chunks:

Plaintext
516c46525633646a64
5539414346564e4232
685142313555625577
455141424762566b46
437755485556454252
513d3d
4. Reconstructing the Payload
Combining the hex strings and converting them back into raw data revealed a Base64 encoded payload:

Bash
echo -n "516c46525633646a645539414346564e4232685142313555625577455141424762566b46437755485556454252513d3d" | xxd -r -p
Resulting Ciphertext:

Plaintext
QlFRV3djdU9ACFVNB2hQB15UbUwEQABGbVkFCwUHUVEBRQ==
Direct Base64 decoding results in raw unreadable binary bytes, indicating that a stream encryption layer (XOR) was applied over it.

5. Key Derivation & Decryption
Based on the network metadata parameters, encryption keys in cellular-themed challenges are often derived from device identifiers. Using the victim's IMSI number (310410728284734), we extracted the last 8 digits: 28284734.

We applied a rolling XOR decryption routine using Python:

Python
import base64

cipher = base64.b64decode("QlFRV3djdU9ACFVNB2hQB15UbUwEQABGbVkFCwUHUVEBRQ==")
key = b"28284734"

plain = bytes(c ^ key[i % len(key)] for i, c in enumerate(cipher))
print(plain.decode())
Flag
Plaintext
picoCTF{r0gu3_c3ll_t0w3r_a7310be3}

#### 💾 طريقة الحفظ والخروج من Nano:
1. اضغطي على زر الاختصار **`Ctrl + O`** ثم اضغطي **`Enter`** لتأكيد الحفظ.
2. اضغطي على زر الاختصار **`Ctrl + X`** للعودة إلى شاشة التيرمينال الرئيسية.

---

أخبريني بمجرد خروجكِ من ملف النانو لننتقل معاً إلى الجزء الأخير وهو الـ Git والرفع 
