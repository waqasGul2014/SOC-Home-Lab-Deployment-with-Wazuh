# Home SIEM Lab: Threat Detection and Automated Response with Wazuh

## 📌 Project Overview
This repository documents a small Security Operations Center (SOC) home lab built with VMware Workstation. A Wazuh 4.14.9 server acts as the SIEM, pfSense provides the firewall and DHCP, an Ubuntu VM is the monitored endpoint, and a Kali Linux VM is the attacker.

Three scenarios were tested: an SSH brute-force attack with automatic blocking (Active Response), file integrity monitoring, and new user creation. The resulting alerts were mapped to MITRE ATT&CK. The lab is designed for cybersecurity learning, threat detection and incident response practice.

> The lab is isolated on a VMware LAN Segment. All attacks were run only against my own virtual machines.

## 🗺 Lab Architecture

| VM | Role | Resources | IP Address |
|---|---|---|---|
| pfSense 2.7.2 | Firewall and DHCP server | 1.4 GB RAM, 1 CPU | LAN 192.168.1.1, WAN 192.168.235.133 (NAT) |
| Wazuh 4.14.9 (OVA) | SIEM server (indexer, manager, dashboard) | 8 GB RAM, 4 CPU | 192.168.1.10 (static) |
| Ubuntu | Monitored endpoint with Wazuh agent and SSH server | 4 GB RAM, 2 CPU | 192.168.1.100 |
| Kali Linux | Attacker | 2 GB RAM, 4 CPU | 192.168.1.101 |

The Wazuh, Ubuntu and Kali VMs share one LAN Segment behind pfSense. DHCP pool: 192.168.1.100 to 192.168.1.199.

![Lab VMs](screenshots/01-lab-vms.png)
![pfSense interfaces](screenshots/02-pfsense-dashboard.png)

## 🛠 Tools & Their Purpose
- **VMware Workstation** – Virtualization platform used to create and manage the VMs.
- **Wazuh (SIEM/XDR)** – Collects and analyzes logs, raises alerts, runs file integrity monitoring and Active Response.
- **pfSense** – Firewall, gateway and DHCP server for the lab network.
- **Ubuntu + Wazuh agent** – The monitored endpoint that sends logs to the Wazuh server.
- **Kali Linux (sshpass, OpenSSH client)** – Generates the brute-force traffic.
- **MITRE ATT&CK** – Framework used to label the observed techniques.

## 🔧 Step-by-Step Setup

### 1. Create the Virtual Machines
1. Import the Wazuh OVA into VMware Workstation and set its memory to **8 GB** (see Challenges).
2. Create VMs for pfSense, Ubuntu and Kali.
3. pfSense gets two network adapters: **NAT** (WAN) and **LAN Segment** (LAN). Ubuntu, Kali and Wazuh use the same **LAN Segment**.
4. Boot order: **pfSense first**, then Wazuh, then Ubuntu and Kali (they get their IPs from pfSense DHCP).

### 2. Verify the Network (Ubuntu VM)
```bash
ip a
ping -c 3 192.168.1.1
ping -c 3 8.8.8.8
ping -c 3 google.com
```

### 3. Give the Wazuh Server a Static IP
In pfSense: **Services > DHCP Server > LAN > DHCP Static Mappings > Add**
- MAC address: the Wazuh VM's MAC
- IP address: `192.168.1.10` (outside the dynamic pool)
- Hostname: `wazuh-server`

Click **Save**, then **Apply Changes**. Delete the old lease under **Status > DHCP Leases**, then power off and power on the Wazuh VM. Confirm on the Wazuh VM:
```bash
ip a
```

![DHCP static mapping](screenshots/04-static-mapping.png)
![Wazuh using 192.168.1.10](screenshots/05-wazuh-ip.png)

### 4. Open the Dashboard
The lab is isolated, so open the dashboard from a browser **inside the Ubuntu VM**:
```
https://192.168.1.10
```

### 5. Install the Wazuh Agent on Ubuntu
The agent version must match the server version (4.14.9). Replace `<WAZUH_SERVER_IP>` with your server address.
```bash
sudo apt install gnupg apt-transport-https curl -y
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
apt-cache madison wazuh-agent
sudo WAZUH_MANAGER="<WAZUH_SERVER_IP>" apt install wazuh-agent=4.14.9-1 -y
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent
```

If the server IP changes later, update the agent:
```bash
sudo sed -i 's|<address>OLD_IP</address>|<address>NEW_IP</address>|' /var/ossec/etc/ossec.conf
sudo grep -A2 "<server>" /var/ossec/etc/ossec.conf
sudo systemctl restart wazuh-agent
```

In the dashboard, **Agents Summary** should show **Active (1)**.

![Agent active](screenshots/03-agent-active.png)

## ⚔️ Scenario 1: SSH Brute-Force Detection

On the Ubuntu VM, make sure the SSH server is running:
```bash
sudo apt install openssh-server -y
sudo systemctl enable --now ssh
```

On the Kali VM, run 15 failed logins for a user that does not exist:
```bash
sudo apt install sshpass -y
for i in {1..15}; do sshpass -p 'wrongpass' ssh -o StrictHostKeyChecking=no fakeuser@192.168.1.100; done
```

Wazuh raised these alerts (Threat Hunting > Events):

| Rule ID | Level | Description |
|---|---|---|
| 5710 | 5 | sshd: attempt to login using a non-existent user |
| 5503 | 5 | PAM: user login failed |
| 5712 | 10 | sshd: brute force trying to get access to the system |

Useful queries:
```
rule.id: 5712
rule.id: 5712 and data.srcip: *
rule.mitre.id: T1110
```

![Brute-force attack from Kali](screenshots/06-bruteforce-attack.png)
![Rule 5712](screenshots/07-rule-5712.png)

## 🛡 Scenario 2: Active Response (Automatic Block)

On the **Wazuh VM**, back up the config, append the block, validate it and restart the manager:
```bash
sudo cp /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.bak
sudo tee -a /var/ossec/etc/ossec.conf > /dev/null <<'EOF'

<ossec_config>
  <active-response>
    <command>firewall-drop</command>
    <location>local</location>
    <rules_id>5712</rules_id>
    <timeout>180</timeout>
  </active-response>
</ossec_config>
EOF
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
```

When rule 5712 fires, the agent runs `firewall-drop` and adds an iptables DROP rule for the attacker for **180 seconds**. Wait 5 to 6 minutes after restarting the manager so the services come up fully.

**Evidence that the attacker was blocked:**
1. Kali: SSH changes from `Permission denied` to `Connection timed out`, and `ping 192.168.1.100` shows 100% packet loss.
2. Ubuntu: `sudo iptables -L -n | grep DROP` shows a DROP rule for `192.168.1.101`.
3. Dashboard: `rule.id: 651` (Host Blocked by firewall-drop Active Response).

![Active Response config](screenshots/08-active-response-config.png)
![Rule 651](screenshots/09-rule-651.png)
![Attacker blocked, Kali](screenshots/10-attacker-blocked-kali.png)
![Attacker blocked, Ubuntu](screenshots/10b-attacker-blocked-ubuntu.png)

## 📁 Scenario 3: File Integrity Monitoring (FIM)

By default FIM scans every 12 hours (43200 seconds). Real-time monitoring was enabled for one test directory on the **Ubuntu VM**:
```bash
mkdir -p ~/fim_test
sudo cp /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.bak
sudo tee -a /var/ossec/etc/ossec.conf > /dev/null <<'EOF'

<ossec_config>
  <syscheck>
    <directories realtime="yes" check_all="yes" report_changes="yes">/home/waqas/fim_test</directories>
  </syscheck>
</ossec_config>
EOF
sudo systemctl restart wazuh-agent
```
Change `/home/waqas/fim_test` to your own path. Wait 1 to 2 minutes for the baseline, then create, modify and delete a file:
```bash
echo "secret data" > ~/fim_test/passwords.txt
echo "changed" >> ~/fim_test/passwords.txt
rm ~/fim_test/passwords.txt
```

| Rule ID | Level | Description |
|---|---|---|
| 554 | 5 | File added to the system |
| 550 | 7 | Integrity checksum changed |
| 553 | 7 | File deleted |

Query:
```
syscheck.path: "/home/waqas/fim_test/passwords.txt"
```

![FIM alerts](screenshots/11-fim-alerts.png)

## 👤 Scenario 4: New User Creation

On the Ubuntu VM, create a test account, give it sudo rights, then remove it:
```bash
sudo adduser --disabled-password --gecos "" testhacker
sudo usermod -aG sudo testhacker
sudo deluser --remove-home testhacker
id testhacker
```

| Rule ID | Level | Description |
|---|---|---|
| 5901 | 8 | New group added to the system |
| 5902 | 8 | New user added to the system |

Routine sudo events were level 3, so level 8 stands out. Query:
```
rule.id: 5902 or rule.id: 5901
```

![New user alerts](screenshots/12-new-user-alerts.png)

## 🧭 MITRE ATT&CK Mapping
The Wazuh MITRE ATT&CK dashboard (last 30 days) grouped the alerts into techniques such as Brute Force (T1110), Password Guessing, Valid Accounts, SSH and Sudo, across tactics such as Credential Access, Privilege Escalation, Persistence and Defense Evasion. The chart also contains older alerts from the Kali agent generated during setup.

![MITRE ATT&CK dashboard](screenshots/13-mitre-attack.png)

## 🧩 Challenges & Solutions

| Challenge | Observed cause | Solution |
|---|---|---|
| Dashboard error 3099 (`wazuh-remoted`, `wazuh-execd` failed) | Wazuh VM had 2.1 GB RAM against the recommended 8 GB (most likely cause, not proven) | Increased memory to 8 GB, restarted the manager, API showed Online |
| VMs had no IP address | pfSense (DHCP) was not running yet | Boot order: pfSense, Wazuh, Ubuntu, Kali |
| Dashboard unreachable from the host | Isolated LAN Segment | Opened the dashboard from the Ubuntu VM |
| Agent install returned HTTP 403 | Placeholder `4.x.x` was used as the version | Installed `wazuh-agent=4.14.9-1` from the apt repository |
| Search returned "No results" | Only a field name was entered | Used `field: value` syntax |
| Could not paste into the Wazuh console | VM console has no clipboard paste | Connected over SSH from the Ubuntu VM |
| Static IP not applied | Old DHCP lease was still active | Applied changes, deleted the lease, power cycled the VM |
| Agent disconnected after the IP change | Agent pointed to the old address | Updated `<address>` in `ossec.conf`, restarted the agent |
| Timeouts after restarting services | Several restarts issued at once | Restarted one service at a time, waited 5 to 6 minutes |

## ⚠️ Limitations
- Single laptop, isolated network, one monitored endpoint and one attacker.
- Active Response was tested only for rule 5712, and the block lasts only 180 seconds.
- The pfSense admin account still uses the default password (acceptable only in an isolated lab).
- pfSense logs are not forwarded to Wazuh.
- A level 11 rootcheck alert on the Ubuntu VM was seen but not investigated.

## 🚀 Next Steps
- Add a Windows VM with Sysmon.
- Write custom rules in `local_rules.xml`.
- Add a network IDS such as Suricata and forward pfSense logs to Wazuh.
- Explore Vulnerability Detection.
- Test more scenarios, such as privilege escalation and persistence.

## 📂 Repository Structure
```
home-siem-lab-wazuh/
├── README.md
├── screenshots/
├── configs/        (ossec.conf snippets written by me)
└── docs/           (project report PDF)
```

## 📚 References
- Wazuh Documentation: https://documentation.wazuh.com
- MITRE ATT&CK: https://attack.mitre.org
- pfSense Documentation: https://docs.netgate.com/pfsense
- Kali Linux Documentation: https://www.kali.org/docs

## 📜 License
This project is for educational purposes only. Use responsibly, and only on systems you own or have permission to test.
