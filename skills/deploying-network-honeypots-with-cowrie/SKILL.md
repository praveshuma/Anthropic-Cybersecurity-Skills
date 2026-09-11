---
name: deploying-network-honeypots-with-cowrie
description: >-
  Deploy, configure, and monitor Cowrie SSH/Telnet network honeypots using Docker to capture brute-force attempts,
  log attacker commands, and collect dropped malware payloads for threat intelligence.
domain: cybersecurity
subdomain: deception-technology
tags: [deception, honeypot, cowrie, ssh, telnet, threat-intelligence, malware-capture, active-defense]
atlas_techniques: []
d3fend_techniques: [D3-DEC]
nist_ai_rmf: []
nist_csf: [DE.CM-01, DE.AE-02, ID.RA-03]
version: "1.0"
author: Pravesh
license: Apache-2.0
---

## When to Use
Trigger this skill when:
- Deploying decoy SSH or Telnet services on internal or perimeter networks to detect unauthorized lateral movement or external scanning.
- Capturing attacker IP addresses, brute-force credentials, and interactive TTY session logs for threat intelligence.
- Intercepting and isolating automated malware payloads (e.g., Mirai, botnet scripts) dropped via `curl`, `wget`, or SFTP/SCP.

**Do not use** this skill for platform selection or multi-honeypot deployments — see `implementing-network-deception-with-honeypots` for choosing between honeypot types and orchestrating multiple decoys. Use this skill specifically for Cowrie deployment, operation, and log analysis.

## Prerequisites
- Linux host with Docker and Docker Compose installed.
- Administrative (`sudo` or `root`) access to bind host network ports or configure IPTables redirects.
- Basic understanding of JSON log parsing tools (`jq`) and log shipping pipelines (SIEM/ELK).

## Workflow

### Step 1: Deploy Cowrie via Docker
Pull the image and start a container with persistent volumes for logs, downloads, and TTY session recordings. See `implementing-network-deception-with-honeypots`'s `references/api-reference.md` (lines 102-110) for the minimal `docker run` invocation; the mounts below extend it with the persistent volume layout this skill relies on in later steps.

```bash
# Pull official Cowrie image
docker pull cowrie/cowrie:latest

# Create host persistent volumes for logs and captured downloads
mkdir -p /var/log/cowrie /var/lib/cowrie/downloads /var/lib/cowrie/tty

# Start Cowrie container with volume mounts
docker run -d \
  --name cowrie-honeypot \
  --restart unless-stopped \
  -p 2222:2222 \
  -p 2223:2223 \
  -v /var/log/cowrie:/cowrie/var/log/cowrie \
  -v /var/lib/cowrie/downloads:/cowrie/var/lib/cowrie/downloads \
  -v /var/lib/cowrie/tty:/cowrie/var/lib/cowrie/tty \
  cowrie/cowrie:latest
```

### Step 2: Route Production Port 22 Traffic to Decoy (Optional)
Redirect real SSH/Telnet ports to the Cowrie container so external scanners hit the decoy instead of the production service:

```bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2222
sudo iptables -t nat -A PREROUTING -p tcp --dport 23 -j REDIRECT --to-port 2223
```

### Step 3: Monitor and Parse Structured JSON Logs
Cowrie records all login attempts, executed commands, and network downloads in `/var/log/cowrie/cowrie.json`.

```bash
# View real-time authentication attempts (successful and failed)
tail -f /var/log/cowrie/cowrie.json | jq -c 'select(.eventid=="cowrie.login.success" or .eventid=="cowrie.login.failed") | {timestamp, src_ip, username, password, eventid}'

# Extract all executed terminal commands from interactive session logs
jq -s '.[] | select(.eventid=="cowrie.command.input") | {timestamp, session, src_ip, input}' /var/log/cowrie/cowrie.json

# List downloaded binary payloads and their calculated SHA-256 hashes
jq -s '.[] | select(.eventid=="cowrie.session.file_download") | {timestamp, src_ip, url, shasum, outfile}' /var/log/cowrie/cowrie.json
```

### Step 4: Replay Attacker Interactive Sessions
Session log files are recorded in UML format under `tty/`. Replay a captured session with Cowrie's built-in `playlog` utility (see the honeypots reference's `bin/playlog` line for the base command):

```bash
docker exec -it cowrie-honeypot playlog /cowrie/var/lib/cowrie/tty/<session_file_id>
```

### Verification
1. Test connectivity from an external or separate test host:
```bash
   ssh -p 2222 root@<honeypot_ip>
```
2. Enter mock credentials (e.g., `root` / `123456`).
3. Run test decoy commands in the fake shell (`uname -a`, `cat /etc/passwd`, `wget http://example.com/test.sh`).
4. Check `/var/log/cowrie/cowrie.json` to verify that `cowrie.login.success`, `cowrie.command.input`, and file-download events were generated.
