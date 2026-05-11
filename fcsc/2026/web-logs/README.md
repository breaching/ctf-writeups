# Web Logs - Walkthrough

## Challenge

**Category:** Forensics  
**Points:** 193  
**Description:** Find the CWE ID, the date, and the requested routes for which the attack succeeded on this exposed web server.  
**File:** `webserver.log.gz`

## Analysis

The file is a gzip-compressed Apache/nginx access log in Combined Log Format:

```
<syslog prefix> <ip> - - [date] "METHOD /path HTTP/x.x" <status> <bytes> "<referer>" "<UA>"
```

~25,470 lines covering 2025-05-06 to 2025-05-16. Numerous automated attack categories present: path traversal, SSRF, SSTI, command injection, admin path brute-force, `.env`/`.git` scans.

### Step 1 - Identify the attack technique

The challenge asks for attacks that **actually worked**, meaning we look for:
- `2xx` status codes on paths that shouldn't return real content
- Response body sizes that differ from the normal page size (2272 bytes)

Filtering for successful non-standard responses:

```bash
zcat webserver.log.gz | awk '{
  match($0, /"[A-Z]+ ([^ ]+) HTTP[^"]*" ([0-9]+) ([0-9]+)/, arr)
  if (arr[2] ~ /^2/ && arr[3] != "2272") print arr[2], arr[3], arr[1], $0
}'
```

### Step 2 - Spot the path traversal

Among the results, two requests stand out at `[07/May/2025:00:40:21 +0000]`:

```
"GET /?asset=../../../../home/webserver/.ssh/id_rsa HTTP/1.0"      200  2500
"GET /?asset=../../../../home/webserver/.ssh/known_hosts HTTP/1.0"  200  2750
```

- The homepage returns **2272 bytes** for all normal/failed attack requests
- These two requests return **2500** and **2750 bytes** respectively - actual file content was exfiltrated
- The attacker used the `?asset` parameter to traverse out of the web root and read SSH private keys

All other 200 responses (SSRF probes, template injection, etc.) returned exactly 2272 bytes, confirming they did NOT succeed - the server just echoed the homepage regardless.

### Step 3 - Classify the vulnerability

**CWE-22: Improper Limitation of a Pathname to a Restricted Directory (Path Traversal)**

The `asset` parameter was passed directly to a file-read function without sanitization, allowing `../` sequences to escape the web root.

## Solution

- **CWE ID:** CWE-22
- **Date:** 05/07
- **Routes that worked:**
  - `/?asset=../../../../home/webserver/.ssh/id_rsa`
  - `/?asset=../../../../home/webserver/.ssh/known_hosts`

## Flag Format

`FCSC{CWE-XXXXX-MM/DD-requests}` with `-` as separator between all fields, routes joined by `-`.

## Flag

`FCSC{CWE-22-05/07-/?asset=../../../../home/webserver/.ssh/id_rsa-/?asset=../../../../home/webserver/.ssh/known_hosts}`
