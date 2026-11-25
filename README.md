# Home SOC Lab: Detecting & Documenting a Simulated Network Intrusion

> **Status:** 🚧 In progress: attacker and victim networks are built; sensor/SIEM build is next.

A hands-on blue-team project: build a small, isolated SOC environment, attack it myself, detect the attacks with a NIDS and SIEM, and write a professional incident report mapped to MITRE ATT&CK.

**Tools:** VirtualBox · Kali Linux · Metasploit · Metasploitable2 · Ubuntu Server · Snort 3 · Splunk Free · Wireshark

---

## Objectives

- Deploy and configure a **NIDS (Snort 3)** and **SIEM (Splunk)** from scratch
- Simulate real attacker techniques: reconnaissance, brute force, and exploitation
- Correlate packet captures (Wireshark), IDS alerts (Snort), and log analysis (Splunk) into a single investigation
- Write **custom detection rules** to close gaps left by default rulesets
- Map findings to **MITRE ATT&CK** and produce a formal incident report

---

## Lab Architecture

All VMs run on an isolated VirtualBox **host-only network**, with no connection to my home network, so attacks stay fully contained.

*NOTE: Ubuntu VM requires two network adapters. One is the **host-only** and the other is **NAT**, because it needs open internet access to install
Snort and Splunk.*

```
 ┌──────────────┐        ┌──────────────────┐        ┌─────────────────────┐
 │  Kali Linux  │  --->  │  Metasploitable2 │        │  Ubuntu Server VM   │
 │  (Attacker)  │        │    (Victim)      │        │  Snort 3 (NIDS) +   │
 │              │        │                  │        │  Splunk Free (SIEM) │
 └──────────────┘        └──────────────────┘        └─────────────────────┘
        \_________________________|_________________________/
                     Host-only network: 192.168.56.0/24
```

| VM | Role | IP Address | Status |
|---|---|---|---|
| Kali Linux | Attacker | `192.168.56.101` | ✅ Deployed |
| Metasploitable2 | Vulnerable victim | `192.168.56.102` | ✅ Deployed |
| Ubuntu Server | Sensor (Snort 3 + Splunk) | `192.168.56.x` | ⏳ Pending |

The sensor VM's adapter will run in **promiscuous mode ("Allow All")** so it sees all traffic on the segment, the same way a NIDS sits on a SPAN/mirror port in a real network.

---

## Progress

### ✅ Phase 1: Isolated Network & Attack/Victim Hosts
- [x] **Step 1:** Created a VirtualBox host-only adapter on `192.168.56.0/24` to isolate the lab from my real network
- [x] **Step 2:** Deployed **Kali Linux** (attacker), attached to the host-only network → `192.168.56.101`
- [x] **Step 3:** Deployed **Metasploitable2** (intentionally vulnerable victim) → `192.168.56.102`
- [x] Verified connectivity: Kali ↔ Metasploitable2 ping successfully

**Troubleshooting note:** I hit a connectivity problem twice when VMs were attached to *different* host-only adapters. I fixed it by checking that every VM's **Settings → Network → Adapter 1 → Name** matched exactly. The lesson: in a virtual network, "host-only" is not enough. The VMs have to be on the *same* adapter.

### ⏳ Phase 2: Sensor & SIEM Build
- [ ] **Step 4:** Deploy Ubuntu Server sensor VM (host-only, promiscuous mode, verify 3-way connectivity)
- [ ] **Step 5:** Install Snort 3, set `HOME_NET` to `192.168.56.0/24`, enable community rules, run in NIDS mode
- [ ] **Step 5b:** Author a **custom Snort rule**
- [ ] **Step 6:** Install Splunk Free, add a monitor input for Snort's alert log, verify with `index=main sourcetype=snort_alert`
- [ ] Build a Splunk dashboard (alerts over time, top source IPs, top signatures)
- [ ] ✔ Checkpoint: Nmap scan from Kali → alert in Snort log → event searchable in Splunk

### ⬜ Phase 3: Attack Simulation
- [ ] Scenario A: Reconnaissance (Nmap)
- [ ] Scenario B: Credential brute force (Hydra → SSH)
- [ ] Scenario C: Exploitation (vsftpd 2.3.4 backdoor via Metasploit)

### ⬜ Phase 4: Detection, Investigation & Reporting
- [ ] Investigate each scenario across Snort, Splunk, and Wireshark
- [ ] Document detection gaps and custom-rule fixes
- [ ] Write the incident report (PDF)

---

## Planned Attack Scenarios & MITRE ATT&CK Mapping

| Scenario | Tool | Tactic | Technique |
|---|---|---|---|
| A: Recon | `nmap -sS -sV -O` | Discovery | T1046: Network Service Discovery |
| B: Brute force | `hydra` (SSH) | Credential Access | T1110: Brute Force |
| C: Exploitation | Metasploit `vsftpd_234_backdoor` | Initial Access | T1190: Exploit Public-Facing Application |

---

## Detection Engineering

### Custom Snort Rule
```
# TODO: add custom rule here
```

### Splunk Queries
```spl
index=main sourcetype=snort_alert
index=main sourcetype=snort_alert | stats count by src_ip, signature
index=main sourcetype=snort_alert | timechart count by signature
```

---

## Screenshots

<!-- Add screenshots to a /screenshots folder and link them here -->
| Description | Screenshot |
|---|---|
| VirtualBox host-only network config | _TODO_ |
| Kali `ip a` (192.168.56.101) | _TODO_ |
| Metasploitable2 `ip a` (192.168.56.102) | _TODO_ |
| Kali → Metasploitable2 ping | _TODO_ |
| Snort alert | _TODO_ |
| Splunk dashboard | _TODO_ |
| Wireshark capture | _TODO_ |

---

## Incident Report

_Coming soon: will be linked here as a PDF once the attack scenarios are complete._

---

## Key Takeaways (so far)

- **Network isolation comes first.** Running intentionally vulnerable machines like Metasploitable2 requires strict containment on a host-only network.
- **Verify every layer.** Checking IPs (`ip a`) and connectivity (`ping`) after each VM deployment caught adapter misconfigurations early, before they could silently break detection later.

---

## ⚠️ Disclaimer

This lab is for educational purposes only. All attacks were performed against intentionally vulnerable systems I own, on an isolated network. Never run these techniques against systems without explicit authorization.
