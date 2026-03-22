# Cowrie Honeypot - Setup Writeup

## Objective
Set up Cowrie SSH honeypot on an Ubuntu Server VM to capture and log attacker behavior as part of a cybersecurity homelab project.

## Environment
- **Host:** Windows 11, AMD Ryzen 7 8840HS, 16GB RAM
- **Hypervisor:** VirtualBox
- **VM:** Ubuntu Server 24 LTS
- **Network:** NAT (switched to Host-Only post setup)

---

## Step 1: Move OpenSSH to a Different Port

Before setting up Cowrie, OpenSSH must be moved off port 22 so Cowrie can take over.

Edited `/etc/ssh/sshd_config` and changed the port to `22222`.
```bash
sudo nano /etc/ssh/sshd_config
```

![sshd_config port change](assets/sshd_config.png)

### Troubleshooting: ssh.socket Issue

After restarting SSH, the port was still showing as 22. The issue was that `ssh.socket` (systemd socket activation) was running and overriding the config file.
```bash
sudo systemctl stop sshd
# Output: "triggering units still active: ssh.socket"
```

Fixed by disabling the socket and enabling the service directly:
```bash
sudo systemctl disable --now ssh.socket
sudo systemctl enable --now ssh.service
sudo systemctl restart ssh.service
```

Verified the port change:
```bash
sudo ss -tlnp | grep ssh
```

---

## Step 2: Install System Dependencies
```bash
sudo apt-get install git python3-pip python3-venv libssl-dev libffi-dev build-essential libpython3-dev python3-minimal authbind iproute2
```

![Dependencies installing](assets/dependencies.png)

---

## Step 3: Create Dedicated Cowrie User
```bash
sudo adduser --disabled-password cowrie
sudo su - cowrie
```

![Cowrie user created](assets/cowrie-user.png)

---

## Step 4: Clone the Repository
```bash
git clone https://github.com/cowrie/cowrie
cd cowrie
```

![Repo cloned](assets/clone.png)

---

## Step 5: Set Up Virtual Environment
```bash
python3 -m venv cowrie-env
source cowrie-env/bin/activate
python -m pip install --upgrade pip
python -m pip install -e .
```

![Virtual environment setup](assets/venv.png)

![Dependencies installed](assets/pip-install.png)

---

## Step 6: Start Cowrie
```bash
cowrie start
```

![Cowrie starting](assets/cowrie-start.png)

Verified Cowrie is listening on port 2222:
```bash
ss -tlnp | grep 2222
```

![Cowrie listening on 2222](assets/cowrie-listening.png)

---

## Step 7: Configure iptables Redirect

Redirected port 22 traffic to Cowrie on port 2222:
```bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2222
```

Verified the rule:
```bash
sudo iptables -t nat -L -n -v
```

---

## Step 8: Verify Cowrie is Working

Connected to Cowrie directly on port 2222:
```bash
ssh -p 2222 root@localhost
```

Checked the logs to confirm the connection was captured:
```bash
cat ~/cowrie/var/log/cowrie/cowrie.log
```
---

## Wazuh Agent Integration

### Objective
Install and enroll the Wazuh agent on the Ubuntu Server VM so Cowrie logs can be forwarded to the Wazuh SIEM dashboard.

---

### Step 1: Add Host-Only Network Adapter

Both VMs were on NAT by default, meaning they couldn't communicate with each other. To enable inter-VM communication, a second Host-Only adapter was added to both the Ubuntu Desktop (Wazuh manager) and Ubuntu Server (Cowrie).

In VirtualBox for each VM:
- Settings → Network → Adapter 2
- Attached to: Host-Only Adapter
- Promiscuous Mode: Allow All

![Host-Only adapter settings](assets/wazuh-adapter-settings.png)

---

### Step 2: Assign Static IP to Ubuntu Server

After booting, `enp0s8` (Host-Only interface) had no IP address assigned. `dhclient` was not available on Ubuntu Server 24, so the IP was assigned manually:
```bash
sudo ip link set enp0s8 up
sudo ip addr add 192.168.56.10/24 dev enp0s8
```

![enp0s8 IP assigned](assets/wazuh-ip.png)

To make the IP persistent across reboots, the netplan config was updated:
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: no
      addresses:
        - 192.168.56.10/24
```
```bash
sudo netplan apply
```

![Netplan config](assets/wazuh-netplan.png)

---

### Step 3: Install Wazuh Agent

From the Wazuh dashboard on the Desktop VM:
- Navigated to **Agents → Deploy new agent**
- Selected Linux (DEB), entered the manager's Host-Only IP, named the agent `cowrie-server`
- Copied the generated install command and ran it on the Ubuntu Server

![Wazuh deploy agent page](assets/wazuh-deploy-agent.png)

---

### Step 4: Start and Enroll the Agent
```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Verified status:
```bash
sudo systemctl status wazuh-agent
```

![Wazuh agent running](assets/wazuh-agent-running.png)

---

### Step 5: Confirm on Dashboard

Agent appeared as **Active** on the Wazuh dashboard under Agents.

![Wazuh agent active on dashboard](assets/wazuh-agent-dashboard.png)

---

### Outcome

Wazuh agent successfully enrolled and communicating with the manager. Next step is configuring Cowrie JSON log output and pointing the Wazuh agent at the log file.