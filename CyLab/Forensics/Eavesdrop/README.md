# Eavesdrop — Forensics (Medium) — 300 pts

<div align="center">

**Category:** Forensics · **Difficulty:** Medium · **Author:** LT 'syreal' Jones · **Event:** picoCTF 2022

`pcap`

</div>

---

> Download this packet capture and find the flag.
>
> **Hint:** All we know is that this packet capture includes a chat conversation and a file transfer.

---

## Recon

```bash
tshark -r capture.flag.pcap
```

Two TCP conversations stand out on non-standard ports:

| Port | Purpose |
|:---:|---|
| `9001` | Plaintext chat conversation |
| `9002` | Binary file transfer |

---

## Step 1 — Read the chat

```bash
tshark -r capture.flag.pcap -Y "tcp.port==9001" -T fields -e tcp.payload
```

Each payload comes back as hex. Decoding a few key lines:

```
4865792c20686f7720646f20796f75206465637279707420746869732066696c65...
→ "Hey, how do you decrypt this file again?"

2a736967682a206f70656e73736c2064657333202d64202d73616c74202d696e2066696c652e64657333202d6f75742066696c652e747874202d6b20737570657273656372657470617373776f7264313233
→ "*sigh* openssl des3 -d -salt -in file.des3 -out file.txt -k supersecretpassword123"
```

The conversation hands over the exact decryption command and the key: `supersecretpassword123`.

---

## Step 2 — Pull the encrypted file

```bash
tshark -r capture.flag.pcap -Y "tcp.port==9002" -T fields -e tcp.payload
```

```
53616c7465645f5f3c4b26e8b8f91e2c4af8031cfaf5f8f16fd40c25d40314e6497b39375808aba186f48da42eefa895
```

Convert the hex dump back into a binary file:

```bash
echo "53616c7465645f5f3c4b26e8b8f91e2c4af8031cfaf5f8f16fd40c25d40314e6497b39375808aba186f48da42eefa895" \
  | xxd -r -p > file.des3
```

*(`Salted__` prefix confirms this is a standard OpenSSL-salted DES3 blob.)*

---

## Step 3 — Decrypt

Using the exact command from the chat log:

```bash
openssl des3 -d -salt \
  -in file.des3 \
  -out file.txt \
  -k supersecretpassword123
```

```bash
cat file.txt
```

---

## Flag

```
picoCTF{nc_73115_411_0ee7267a}
```

---

## Tools

| Tool | Purpose |
|---|---|
| `tshark` | Extract raw TCP payloads by port |
| `xxd -r -p` | Convert hex text back to binary |
| `openssl des3` | Decrypt the salted DES3 file |
