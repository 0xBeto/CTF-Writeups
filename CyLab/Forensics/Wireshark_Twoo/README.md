# Wireshark twoo twooo two twoo... — Forensics (Medium) — 100 pts

<div align="center">

**Category:** Forensics · **Author:** Dylan · **Event:** picoCTF 2021

`pcapng`

</div>

---

> Can you find the flag?
>
> **Hint 1:** Did you really find *the* flag?
>
> **Hint 2:** Look for traffic that seems suspicious.

---

## Recon

```bash
file shark2.pcapng
```
```
shark2.pcapng: pcapng capture file - version 1.0
```

```bash
tshark -r shark2.pcapng -q -z io,phs
```

```
frame                                    frames:4831 bytes:3355920
  eth
    ip
      tcp
        tls
        http
      udp
        gquic
        dns                               frames:1512 bytes:223404
    arp
```

A large volume of DNS traffic relative to everything else stands out immediately — 1512 DNS frames is unusually high for a normal capture.

---

## The decoy

Hint 1 ("did you really find *the* flag") is a warning: an obvious flag-looking string typically shows up somewhere easy (HTTP body, plaintext string) and is a plant. The real target — per hint 2 — is the "suspicious traffic," which points at the DNS volume.

---

## Inspecting the DNS queries

```bash
tshark -r shark2.pcapng -Y dns -T fields -e dns.qry.name
```

The output is dominated by queries to a single domain, each one prefixed with a short random-looking label, repeated across three suffix variants:

```
<label>.reddshrimpandherring.com
<label>.reddshrimpandherring.com.us-west-1.ec2-utilities.amazonaws.com
<label>.reddshrimpandherring.com.windomain.local
```

The domain name itself (`redd shrimp and herring` → "red herring") is a strong hint that this is a decoy channel. But the *pattern* — many distinct-looking subdomain labels queried in sequence — is classic **DNS tunneling / DNS exfiltration**: data smuggled out one query at a time, encoded into subdomain labels.

---

## Isolating the exfil traffic

Filtering just the queries going to the external resolver IP (`18.217.1.57`) cuts the noise down to the handful of DNS queries that actually carry the encoded payload:

```bash
tshark -r shark2.pcapng -Y "dns && ip.dst==18.217.1.57"
```

```
DNS Standard query ... A cGljb0NU.reddshrimpandherring.com
DNS Standard query ... A RntkbnNf.reddshrimpandherring.com
DNS Standard query ... A M3hmMWxf.reddshrimpandherring.com
DNS Standard query ... A ZnR3X2Rl.reddshrimpandherring.com
DNS Standard query ... A YWRiZWVm.reddshrimpandherring.com
DNS Standard query ... A fQ==.reddshrimpandherring.com
```

Each unique label (the `.com`, `.amazonaws.com`, and `.windomain.local` variants are just repeats of the same label sent three times) looks like Base64.

---

## Reassembling and decoding

Concatenating the unique labels in the order they were queried:

```bash
echo "cGljb0NURntkbnNfM3hmMWxfZnR3X2RlYWRiZWVmfQ==" | base64 -d
```

```
picoCTF{dns_3xf1l_ftw_deadbeef}
```

### One-liner version

The whole extraction can be done in a single pipeline:

```bash
tshark -r shark2.pcapng \
  -Y "dns && ip.dst==18.217.1.57" \
  -T fields -e dns.qry.name |
  awk -F'.' '{print $1}' |
  uniq |
  tr -d '\n' |
  base64 -d
```

---

## Flag

```
picoCTF{dns_3xf1l_ftw_deadbeef}
```

---

## Tools

| Tool | Purpose |
|---|---|
| `file` | Confirm the capture format |
| `tshark -z io,phs` | Get a protocol hierarchy overview and spot the abnormal DNS volume |
| `tshark -Y dns` | List all DNS queries |
| `tshark -Y "dns && ip.dst==..."` | Isolate exfiltration traffic to the specific external resolver |
| `awk` / `uniq` / `tr` | Extract and reassemble the unique subdomain labels |
| `base64 -d` | Decode the reassembled payload |
