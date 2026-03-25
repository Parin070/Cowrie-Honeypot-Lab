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
```
Kali Linux (Attacker) → Cowrie Honeypot (Ubuntu Server) → Wazuh SIEM (Dashboard)
```

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

## Status

## Status

Cowrie is fully operational with the Wazuh integration complete. SSH honeypot activity is being forwarded to the Wazuh SIEM dashboard and triggering custom detection rules in real time. Attack simulation from Kali Linux and Suricata IDS integration are pending.