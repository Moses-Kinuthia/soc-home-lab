# 🛡️ Enterprise-grade security monitoring environment built from the ground up.

# SOC Home Lab: Distributed Architecture & Monitoring
This repository contains the technical documentation, network schemas, and configuration files for my distributed security operations lab.

## 🏗️ Lab Architecture
The environment is built across two physical hosts using VirtualBox and a physical ethernet bridge to simulate a corporate network.
- **Perimeter Security:** pfSense (Firewall/Router/DHCP)
- **SIEM/XDR:** Wazuh 4.x (Manager & Indexer)
- **Endpoints:** Windows Server 2019 (DC) & Windows 10 LTSC (client).

## ⚙️ Core Configurations
- **Segmentation:** Isolated segments (Attack, Management, Production) using pfSense Rules.
- **Telemetry:** Advanced logging via Sysmon and Windows Event Forwarding.
- **Hardening:** Implementation of CIS Benchmarks for Windows Server 2019.

---

## 🧪 Current Focus & Research
I am currently simulating enterprise-level threats to validate detection logic:
1. **Endpoint Monitoring:** Analyzing PowerShell execution and unusual process parenting on the Windows Host.
2. **Network Visibility:** Monitoring cross-device communication via the Bridged network interface.
3. **FIM (File Integrity Monitoring):** Tracking unauthorized changes to sensitive system directories.

---

## 📂 Documentation
- `/network`: Diagrams and IP addressing schema.
- `/configs`: Exported firewall rules and Wazuh agent configurations.

---

## 📊 Dashboard Preview

<img width="1910" height="895" alt="image" src="https://github.com/user-attachments/assets/03d75751-9140-467e-80f9-b636e645df13" />

---

## 📜 Disclaimer
This project is for defensive research and educational purposes. All monitoring is performed on owned hardware with explicit consent for data collection.
