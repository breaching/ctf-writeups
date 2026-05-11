# Forenzeek - Compromission initiale - Walkthrough

## Challenge

**Category:** Forensics  
**Points:** 119  
**Description:** A compromise was observed on 192.168.1.42 via a malicious email. Can you find the uid of the connection associated with the email download?  
**File:** `forenzeek.csv.gz`

## Analysis

The file is a **Zeek `conn` log** (tab-separated) containing network connection records with the following fields:
```
ts  uid  id.orig_h  id.orig_p  id.resp_h  id.resp_p  bytes
```

### Step 1 - Identify email-related connections

Email download protocols are IMAP (143), IMAPS (993), POP3 (110), or POP3S (995). Filtering connections from the compromised host 192.168.1.42:

```bash
zcat forenzeek.csv.gz | grep "192.168.1.42" | awk '{print $6}' | sort -u
# → ports: 53, 80, 123, 135, 138, 389, 443, 445, 587, 5353, 5986, 67, 7680, 88, 9200, 993
```

Port **993 (IMAPS)** is present - the host is downloading mail over TLS-encrypted IMAP from `192.168.1.4`.

### Step 2 - Identify the malicious email connection

There are 7 IMAPS connections. Comparing byte counts:

| uid | timestamp | bytes |
|-----|-----------|-------|
| 3bdf97d666eb3e2315 | 1756306535 | 31,677 |
| ad5f40ce496a84830c | 1756307469 | 24,655 |
| f6950e08a64ef8e28b | 1756308295 | 36,599 |
| **c2ad3fb71679d16ec9** | **1756310527** | **102,025** |
| ef5eb1d990fc801d05 | 1756311805 | 29,372 |
| 7bcfa0d6d873c204e7 | 1756311837 | 51,839 |
| 51420c424aba3152c8 | 1756313622 | 39,891 |

The connection `c2ad3fb71679d16ec9` is **~3x larger** than the typical IMAPS session - consistent with a malicious email carrying a payload attachment.

### Step 3 - Confirm by post-compromise activity

After timestamp 1756310527, the host initiates **SMTP submissions (port 587)** to 192.168.1.4:

```
1756311352 → 192.168.1.4:587 (first SMTP send, ~13 min after large IMAPS)
1756313608 → 192.168.1.4:587
1756315566 → 192.168.1.4:587
```

This is a classic indicator of compromise: after receiving the malicious email, the machine started forwarding/spreading via SMTP - confirming the large IMAPS connection was the infection vector.

## Flag

`FCSC{c2ad3fb71679d16ec9}`
