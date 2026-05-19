# SOC Home Lab — Distributed Architecture & Monitoring

![Wazuh](https://img.shields.io/badge/SIEM-Wazuh%204.x-blue)
![pfSense](https://img.shields.io/badge/Firewall-pfSense-lightgrey)
![AD](https://img.shields.io/badge/Directory-Active%20Directory-blue)
![Status](https://img.shields.io/badge/Status-Operational-success)

A self-built, two-machine SOC home lab running a realistic mini-enterprise
environment. Built and debugged from scratch — no cloud shortcuts, no
pre-configured images.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Network Design](#network-design)
- [Components](#components)
- [What Was Built and Configured](#what-was-built-and-configured)
- [Build Challenges and How They Were Solved](#build-challenges-and-how-they-were-solved)
- [Log Sources and Event IDs Monitored](#log-sources-and-event-ids-monitored)
- [Configurations](#configurations)
- [Detection Work Built on this Lab](#detection-work-built-on-this-lab)

---

## Architecture Overview

```
[Internet]
     |
[Home Router]  192.168.8.1
     |
     +─── Machine A  (Windows 11 Host)
     |     ├── pfSense VM
     |     │     WAN  : 192.168.8.224   (uplink)
     |     │     LAN  : 10.0.0.1/24     (production)
     |     │     OPT1 : 10.0.10.1/24    (management / Wazuh)
     |     │     OPT2 : 10.0.20.1/24    (attack — isolated)
     |     └── Wazuh Manager+Indexer    10.0.10.11
     │
     │  [Physical Ethernet Bridge]
     │
     +─── Machine B  (Windows 10 Host)
           ├── DC01    (Windows Server 2019)  10.0.0.10
           ├── WKSTNO1 (Windows 10 Enterprise) 10.0.0.100
           └── Kali Linux                      10.0.20.11
```

---

## Network Design

Three isolated segments enforce network separation between the attacker, the
simulated enterprise, and the SIEM management plane.

| Interface | Subnet | Purpose |
|---|---|---|
| LAN | 10.0.0.0/24 | Production — DC01 and WKSTNO1 |
| OPT1 (labnet) | 10.0.10.0/24 | Management — Wazuh server only |
| OPT2 (attack) | 10.0.20.0/24 | Attack — Kali Linux, isolated |

**Firewall policy:**
- Default-deny on all inter-segment traffic.
- Explicit allow: Wazuh agents on LAN/OPT2 → Wazuh manager on OPT1 (ports 1514, 1515).
- Explicit allow: pfSense syslog → Wazuh OPT1 (UDP 514).
- Explicit allow: OPT2 → WAN (NAT) for Kali internet access.
- LAN → OPT2 and OPT2 → LAN are blocked by default; OPT2 → LAN traffic is allowed
  only for simulated attack scenarios and explicitly logged for detection.

See [`network/ip-schema.md`](network/ip-schema.md) for the full addressing table.

---

## Components

| Component | Technology | Version | Role |
|---|---|---|---|
| Firewall / Router | pfSense | 2.7.x | Segmentation, NAT, syslog forwarding |
| SIEM / XDR | Wazuh Manager + Indexer | 4.x | Alerting, correlation, FIM, SCA |
| Domain Controller | Windows Server 2019 | DC01 | AD DS, DNS, GPO |
| Endpoint | Windows 10 Enterprise LTSC | WKSTNO1 | Monitored workstation |
| Attack Platform | Kali Linux | Rolling | Red team simulations |
| Endpoint Telemetry | Sysmon | v15 | Process, network, file events |
| Hypervisor | VirtualBox | 7.x | Both hosts |

---

## What Was Built and Configured

### pfSense
- Multi-interface routing: WAN / LAN / OPT1 (labnet) / OPT2 (attack)
- Default-deny inter-VLAN with explicit allow rules per traffic type
- NAT rule for Kali internet access from OPT2
- Syslog forwarding to Wazuh (UDP 514) for firewall-event visibility

### Active Directory
- Domain: `SOCLAB.local`
- DC01 promoted as domain controller; WKSTNO1 domain-joined
- Simulation users created: `jkamau`, `gwanjiku`, `itsupport_backup`
- Audit policy configured for: logon events, account management, policy change,
  privilege use, directory service access

### Wazuh
- Agents deployed on DC01 and WKSTNO1, reporting to manager on OPT1
- Sysmon event channel added to agent `ossec.conf` for enriched process telemetry
- pfSense syslog ingested via remote syslog listener (UDP 514)
- Custom detection rules in `etc/rules/local_rules.xml` (see
  [Financial-Fraud-Cyber-Threat-Detection-Pipeline](https://github.com/Moses-Kinuthia/Financial-Fraud-Cyber-Threat-Detection-Pipeline))
- Alerts confirmed firing: Event IDs 4625, 4624, 4720, 4728, 4740, 1102

### Sysmon
- Config deployed to both Windows machines via GPO (no manual per-host install)
- Covers: process creation (Event ID 1), network connections (3), file creation (11),
  registry changes (12/13/14), PowerShell script-block correlation

---

## Build Challenges and How They Were Solved

This section documents the real problems encountered during build — the kind of
detail that doesn't appear in tutorials. These are the proofs that the lab was
genuinely built, not followed from a guide.

---

### 1. Wazuh agents not generating Windows Security alerts

**Symptom:** Wazuh was running, agents were connected, but Windows Security
events (4625, 4624, etc.) were not appearing in alerts.

**Root cause:** The default `ossec.conf` on the agent included a `<query>` filter
in the `Microsoft-Windows-Security-Auditing` localfile block. This filter was
too restrictive and was silently dropping most Security channel events before
they reached the manager.

**Fix:** Removed the `<query>` filter from the Security event channel block in
`ossec.conf` on both agents, leaving only the bare `<location>` and
`<log_format>eventchannel</log_format>` entries. Restarted the Wazuh agent
service. Events 4625 and 4624 began appearing in `alerts.json` immediately.

---

### 2. DC audit policy not persisting — no logon events from DC01

**Symptom:** The Windows Security audit policy was set via Group Policy, but
DC01 was not generating 4625/4624 events. `auditpol /get /category:*` showed
the policy was applied, but events did not appear in the Security log.

**Root cause:** Windows Server 2019 domain controllers have a legacy audit
compatibility issue. When both the old-style Security Settings audit policy
and the newer Advanced Audit Policy (subcategory-level settings) are present,
the legacy policy wins and suppresses the subcategory settings. The relevant
registry key was missing:

```
HKLM\SYSTEM\CurrentControlSet\Control\Lsa
Value: SCENoApplyLegacyAuditPolicy = 1 (DWORD)
```

**Fix:** Added `SCENoApplyLegacyAuditPolicy = 1` via `regedit` on DC01.
Forced a `gpupdate /force`. Logon events immediately started appearing in
the Security event log and in Wazuh.

---

### 3. Kali could not reach the internet from OPT2

**Symptom:** After placing Kali on the OPT2 (attack) segment, it had no
internet connectivity — `apt update` timed out, tools could not pull updates.

**Root cause:** pfSense had a NAT outbound rule for LAN → WAN but not for
OPT2 → WAN. Kali traffic was hitting the firewall and being dropped at the
NAT stage.

**Fix:** Added a manual outbound NAT rule for the OPT2 (10.0.20.0/24) subnet
mapping to the WAN interface IP. Also added an OPT2 firewall pass rule to
allow OPT2 traffic to the WAN gateway. Kali internet access restored.

---

### 4. Kali could not reach LAN hosts for attack simulations

**Symptom:** After fixing internet access, Kali still could not connect to
DC01 (10.0.0.10) or WKSTNO1 (10.0.0.100) for simulated attacks.

**Root cause:** pfSense blocks all inter-segment traffic by default. There was
no rule permitting OPT2 → LAN. The NAT fix above only addressed OPT2 → WAN.

**Fix:** Added a pfSense firewall rule on the OPT2 interface allowing traffic
from 10.0.20.0/24 to 10.0.0.0/24 for the duration of attack simulations.
All OPT2 → LAN traffic is logged; this logging is what enables the lateral
movement detection rules to fire on Wazuh.

---

### 5. Sysmon deployment — manual install did not survive reboots cleanly

**Symptom:** Sysmon was installed manually on WKSTNO1, but after a reboot the
Sysmon service would sometimes fail to start, causing process-creation events
to disappear from Wazuh alerts.

**Fix:** Moved Sysmon deployment to a GPO startup script that checks for the
service state and reinstalls if absent. Both Windows machines now receive
Sysmon consistently via GPO, and the service state is stable across reboots.

---

## Log Sources and Event IDs Monitored

| Source | Event IDs | Detection Purpose |
|---|---|---|
| Windows Security | 4624 | Successful logon |
| Windows Security | 4625 | Failed logon |
| Windows Security | 4720 | New user account created |
| Windows Security | 4728 / 4732 | Member added to privileged group |
| Windows Security | 4740 | Account locked out |
| Windows Security | 4769 | Kerberos service ticket request |
| Windows Security | 1102 | Security audit log cleared |
| Sysmon | 1 | Process creation |
| Sysmon | 3 | Network connection |
| Sysmon | 11 | File creation |
| PowerShell | 4104 | Script-block logging |
| pfSense syslog | — | Firewall allow/deny, network scan detection |

---

## Configurations

| File | Description |
|---|---|
| [`configs/ossec.conf`](configs/ossec.conf) | Wazuh manager configuration (redacted) |
| [`configs/wazuh-custom-rules.xml`](configs/wazuh-custom-rules.xml) | Custom detection rules (same file as Financial-Fraud repo) |

---

## Detection Work Built on this Lab

- [**Financial-Fraud-Cyber-Threat-Detection-Pipeline**](https://github.com/Moses-Kinuthia/Financial-Fraud-Cyber-Threat-Detection-Pipeline) —
  three-scenario detection engineering project (ATO, insider threat, lateral movement) built entirely in this environment.

- [**simulation-writeups**](https://github.com/Moses-Kinuthia/simulation-writeups) —
  incident-style investigation reports for each simulated attack.
