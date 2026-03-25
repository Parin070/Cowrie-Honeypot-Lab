# Cowrie SSH Honeypot Lab

A documentation of setting up Cowrie, a medium-interaction SSH honeypot, on an Ubuntu Server VM. Built as part of a cybersecurity homelab to capture and analyze attacker behavior.

## What is Cowrie?

Cowrie is an SSH honeypot that simulates a vulnerable SSH server. When an attacker connects, they are dropped into a fake shell environment where all their commands are logged. It captures login attempts, credentials used, and commands executed.

## Environment

- **Host:** Windows 11, AMD Ryzen 7 8840HS, 16GB RAM
- **Hypervisor:** VirtualBox
- **Honeypot VM:** Ubuntu Server 24.04 LTS (NAT adapter)
- **Attacker VM:** Kali Linux (planned)
- **SIEM:** Wazuh (planned integration)

## Architecture

### Network Setup

All virtual machines use two adapters each:

| Adapter | Purpose |
|---|---|
| NAT | Internet access (updates, package installs) |
| Host-Only | Internal lab communication between VMs |

The Host-Only network is how Kali reaches Cowrie to simulate attacks, and how the Wazuh Agent forwards logs to the Wazuh Manager.

---

### Attack Flow

```
Attacker (Kali Linux)
        │
        │  SSH to port 22
        ▼
Ubuntu Server (iptables)
        │
        │  Redirects to port 2222
        ▼
Cowrie Honeypot
        │
        │  Writes cowrie.json
        ▼
Wazuh Agent
        │
        │  Forwards over TLS (port 1514)
        ▼
Wazuh Manager → OpenSearch → Wazuh Dashboard
```

---

### Components

**Cowrie (Honeypot)**
Emulates a vulnerable SSH server on port 2222, accepting attacker logins and recording every credential and command entered. Nothing is actually executed — it's a fully faked shell environment.

**iptables (Port Redirect)**
Silently redirects inbound traffic from port 22 to port 2222, making Cowrie appear as a legitimate SSH service to the attacker.

**Wazuh Agent**
Runs on the Ubuntu Server and monitors Cowrie's JSON log file. Ships new log entries to the Wazuh Manager in real time over an encrypted connection.

**Wazuh Manager**
Parses incoming Cowrie logs using custom decoders and runs them through detection rules to flag brute-force attempts, suspicious commands, and malware download patterns.

**Wazuh Dashboard**
Web UI that displays all alerts, attacker IPs, and session activity pulled from OpenSearch in real time.

---

### Port Reference

| Port | Service | Notes |
|---|---|---|
| 22 | SSH (public-facing) | Redirected to Cowrie via iptables |
| 2222 | Cowrie | Actual honeypot listener |
| 2223 | Real SSH | Admin access (moved off port 22) |
| 1514 | Wazuh Agent → Manager | Log forwarding over TLS |
| 5601 | Wazuh Dashboard | Web UI |

## Setup Overview

1. Install system dependencies
2. Create dedicated `cowrie` user
3. Clone Cowrie repository
4. Set up Python virtual environment
5. Install Python dependencies
6. Configure iptables redirect (port 22 → 2222)
7. Start Cowrie and verify logs

## Prerequisites

- Ubuntu Server VM with internet access
- OpenSSH moved off port 22 before setup
- Python 3.12+

## Troubleshooting

See `WRITEUP.md` for detailed setup walkthrough and troubleshooting notes.

## What I Learned

- **How honeypots work** — Deception-based security in practice; letting attackers in on purpose to observe behavior without exposing real systems.
- **How attacks happen** — Brute-force attempts, credential stuffing, and post-login commands like `wget` and `chmod +x` went from textbook concepts to things I could see in logs in real time.
- **Linux networking & iptables** — How traffic is isolated across VMs and how packet-level redirection works (port 22 → 2222).
- **Wazuh/SIEM setup** — How a SIEM is architected, how agents communicate with a manager, and what it takes to get a full monitoring stack running.
- **Log analysis & detection rules** — How SIEMs turn raw JSON logs into actionable alerts using decoders, pattern matching, and rule chaining.

## Status

Cowrie is fully operational with the Wazuh integration complete. SSH honeypot activity is being forwarded to the Wazuh SIEM dashboard and triggering custom detection rules in real time. Attack simulation from Kali Linux and Suricata IDS integration are pending.
