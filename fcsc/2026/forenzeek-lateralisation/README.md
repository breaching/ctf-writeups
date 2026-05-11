# Forenzeek - Latéralisation - Walkthrough

## Challenge

**Category:** Forensics  
**Points:** 217  
**Description:** Can you identify the uid of the connection that allowed the attacker to compromise the administrator's machine?  
**File:** `forenzeek2.csv.gz`

## Analysis

Same Zeek `conn` log format as the first Forenzeek challenge.

### Step 1 - Identify the administrator's machine

`192.168.1.38` is clearly the admin/management machine: it consistently initiates WinRM (port 5986) connections to every host in the network throughout the entire log period - this is typical of a Windows management system (SCCM, Ansible, or custom admin scripts using WinRM).

```bash
zcat forenzeek2.csv.gz | awk '$6 == "5986"' | awk '{print $3}' | sort | uniq -c
# → 192.168.1.38 appears as the source for nearly all WinRM connections
```

### Step 2 - Find the anomalous connection

Normal traffic pattern: `.38` → other hosts on `:5986`

Anomalous: `.42` → `.38:5986`

```
ts=1756308531  uid=9a4fe41babf12d1bdf  192.168.1.42:53559 → 192.168.1.38:5986  34224 bytes
```

This **reversed direction** (a compromised workstation connecting to the admin machine instead of the other way around) is the attacker using stolen credentials from `.42` to authenticate into the admin machine via WinRM HTTPS.

### Step 3 - Confirm the attack chain

Prior to this connection, `.42` made suspicious connections:
- `192.168.1.42 → 192.168.1.2:445` (SMB to domain controller - credential harvesting)
- `192.168.1.42 → 192.168.1.2:135` (WMI/RPC - likely DCE/RPC for credential dump)

These established the foothold and credential access, which were then used to authenticate to the admin machine via WinRM.

After this connection, `192.168.1.38` is under attacker control - confirmed by the subsequent connection `192.168.1.38 → 192.168.1.42:5986` (`.38` connecting back to `.42`), which is consistent with the attacker moving commands back through the compromised chain.

## Flag

`FCSC{9a4fe41babf12d1bdf}`
