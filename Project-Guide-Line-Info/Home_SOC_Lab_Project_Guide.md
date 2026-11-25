# Home SOC Lab: Detecting & Documenting a Simulated Network Intrusion

**A beginner-friendly, resume-ready SOC analyst project**

This project is scoped for where you're at right now: Security+ certified, CCNA in progress, and hands-on (if light) experience with Splunk, Wireshark, and Snort. It's designed to take those separate tool skills and combine them into one coherent story you can put on a resume, talk through in an interview, and point to on GitHub: *"I built a small SOC, attacked it myself, detected the attacks, and wrote the incident report."*

Estimated time: 2–4 weekends, depending on how deep you go. You can also do it in shorter sessions — the steps are checkpointed so you can stop and resume.

---

## What this demonstrates to an employer

- You can stand up and configure a SIEM (Splunk) and a NIDS (Snort) — not just describe what they do.
- You understand the attacker's side well enough to simulate real techniques (recon, brute force, exploitation).
- You can correlate raw packet data (Wireshark), IDS alerts (Snort), and log analysis (Splunk) into one investigation.
- You can map real activity to the MITRE ATT&CK framework — a language every SOC uses.
- You can write a clear, professional incident report — the deliverable that actually gets read by managers and clients, and the skill you said you want to build.

---

## 1. Lab architecture

You'll build a small isolated network inside VirtualBox with three virtual machines. Nothing here touches your real home network — everything runs on an internal "host-only" adapter so you can attack it safely.

```
 ┌─────────────┐        ┌──────────────────┐        ┌────────────────────┐
 │   Kali Linux │  --->  │  Metasploitable2  │        │  Ubuntu Server VM   │
 │  (Attacker)  │        │    (Victim)       │        │  Snort 3 (NIDS) +   │
 │              │        │                   │        │  Splunk Free (SIEM) │
 └─────────────┘        └──────────────────┘        └────────────────────┘
        \_________________________|_________________________/
                     Host-only internal network
                        (e.g., 192.168.56.0/24)
```

**Why this setup:** the sensor VM sits on the same segment and captures/inspects all traffic between attacker and victim, which is exactly how a NIDS is positioned in a real network (e.g., on a SPAN/mirror port).

### Software you'll need (all free)

| Component | Where to get it | Notes |
|---|---|---|
| VirtualBox | [virtualbox.org](https://www.virtualbox.org/) | Or VMware Workstation Player if you prefer |
| Kali Linux | [kali.org/get-kali](https://www.kali.org/get-kali/) | Pre-built VirtualBox image is easiest |
| Metasploitable2 | Rapid7 / SourceForge "metasploitable2" | Intentionally vulnerable — never expose it beyond the host-only network |
| Ubuntu Server (22.04 or 24.04 LTS) | [ubuntu.com/download/server](https://ubuntu.com/download/server) | This will be your sensor/SIEM box |
| Snort 3 | [docs.snort.org](https://docs.snort.org/start/installation) | NIDS engine |
| Splunk Free | [splunk.com/en_us/download/splunk-enterprise.html](https://www.splunk.com/en_us/download/splunk-enterprise.html) | Free license: 500 MB/day ingest, single user, no alerting scheduler — plenty for a lab |

---

## 2. Step-by-step build

### Step 1 — Create the isolated network
In VirtualBox: **File → Host Network Manager → Create** a host-only adapter (e.g., `vboxnet0`, subnet `192.168.56.0/24`). You'll attach all three VMs' network adapters to this instead of NAT/Bridged, so the lab is fully isolated from your real network.

### Step 2 — Deploy the attacker (Kali)
Import the Kali VirtualBox image, attach it to the host-only adapter, boot it, confirm it gets an IP in the 192.168.56.x range (`ip a`).

### Step 3 — Deploy the victim (Metasploitable2)
Import Metasploitable2, attach to the same host-only adapter. Default credentials are `msfadmin`/`msfadmin`. Note its IP.

### Step 4 — Deploy the sensor/SIEM (Ubuntu Server)
Install Ubuntu Server, attach it to the same host-only adapter. This box will run both Snort and Splunk. Put its interface into promiscuous mode in VirtualBox settings ("Promiscuous Mode: Allow All") so it can see all traffic on the segment, not just traffic addressed to it.

### Step 5 — Install and configure Snort 3
Follow the official install guide for your Ubuntu version.¹ Key configuration points:
- Set `HOME_NET` in your Snort config to your lab subnet (`192.168.56.0/24`).
- Enable the community rule set as a starting point.
- Run Snort in NIDS (sniffer + logging) mode against your host-only interface, alerting to a local log file.
- **Write at least one custom rule** (e.g., alert on FTP connection attempts, or on a specific Nmap scan pattern). This is a small detail, but "authored custom detection logic" is a genuinely good resume line — most entry-level candidates only ever use default rules.

### Step 6 — Install Splunk Free and ingest Snort alerts (CONTINUE FROM THIS STEP)
Install Splunk Free on the sensor VM (or a 4th VM if you want to keep concerns separated). Splunk Free's 500 MB/day limit is far more than a lab generates.²
- Add a **monitor input** pointing at Snort's alert log file.
- Confirm alerts are landing in Splunk with a simple search: `index=main sourcetype=snort_alert`.
- (Optional, worth doing) Build one simple dashboard panel: alert count over time, top source IPs, top signatures. This gives you a screenshot for your report and your GitHub README.

**Checkpoint:** at this point you should be able to run a basic scan from Kali and watch an alert appear in both Snort's log and Splunk.

---

## 3. Simulate three attack scenarios

Run each of these from Kali against Metasploitable2, and start a Wireshark capture on the sensor VM's interface before each one so you have packet-level evidence.

**Scenario A — Reconnaissance (Nmap scan)**
```
nmap -sS -sV -O 192.168.56.<victim-ip>
```
Maps to MITRE ATT&CK **T1046 – Network Service Discovery**.

**Scenario B — Credential brute force (Hydra)**
```
hydra -l msfadmin -P /usr/share/wordlists/rockyou.txt ssh://192.168.56.<victim-ip>
```
Maps to MITRE ATT&CK **T1110 – Brute Force**.

**Scenario C — Exploitation (vsftpd 2.3.4 backdoor)**
Metasploitable2 ships a deliberately backdoored vsftpd on port 21 — a well-known, safe target for practicing exploitation in Metasploit (`use exploit/unix/ftp/vsftpd_234_backdoor`).
Maps to MITRE ATT&CK **T1190 – Exploit Public-Facing Application**.

Keep notes as you go: exact timestamps (use UTC), source/destination IPs and ports, and what you observed on the attacker side. You'll need these for the report.

---

## 4. Detect and investigate

For each scenario:
1. **Snort** — check whether an alert fired. If a default rule didn't catch it, that's a good opportunity to write a custom rule (see Step 5) and re-run the attack to confirm detection. Document both the miss and the fix — that narrative ("initial detection gap, remediated by custom rule") reads very well in interviews.
2. **Splunk** — pull the matching events. Example searches to build out:
   - `index=main sourcetype=snort_alert | stats count by src_ip, signature`
   - `index=main sourcetype=snort_alert | timechart count by signature`
3. **Wireshark** — open the corresponding pcap and confirm what the traffic looked like at the packet level (e.g., the SYN scan pattern, the repeated failed SSH auth attempts, the FTP backdoor command sequence). Screenshot the key frames.

---

## 5. Map findings to MITRE ATT&CK

| Scenario | Tactic | Technique |
|---|---|---|
| Nmap scan | Reconnaissance | T1046 – Network Service Discovery |
| Hydra brute force | Credential Access | T1110 – Brute Force |
| vsftpd exploit | Initial Access | T1190 – Exploit Public-Facing Application |

Including a table like this in your report signals fluency with the framework SOC teams actually use to communicate — this alone sets a resume apart from someone who's only done tutorials.

---

## 6. Write the incident report

Use the companion template (**SOC_Incident_Report_Template.docx**) sent alongside this guide. Treat the whole exercise as one incident with three observed stages (recon → brute force → exploitation), which is realistic — that's how real intrusions often look.

A few report-writing habits worth building here, since that's a stated goal:
- **Lead with impact, not narrative.** The first paragraph (executive summary) should tell a reader who has 30 seconds what happened and what mattered, before the detailed timeline.
- **Timestamps in UTC, always.** Real SOCs standardize on UTC to avoid timezone confusion across shifts and geographies.
- **Separate fact from assessment.** State what the evidence shows, then clearly label your analysis/recommendation as such.
- **Every claim needs an artifact.** If you say "brute force was detected at 14:32 UTC," there should be a screenshot or log excerpt right there backing it up.
- **Write the remediation section like you're talking to whoever owns the affected system** — specific, actionable, prioritized (e.g., "disable backdoored vsftpd version," not "improve security").

---

## 7. Publish and turn it into resume material

1. Create a GitHub repo (e.g., `home-soc-lab`) with:
   - A README containing the network diagram, a summary of what you built, your custom Snort rule, your Splunk search queries, and 3–5 screenshots (Splunk dashboard, Wireshark capture, Snort alert).
   - The incident report (as a PDF export of the docx) linked from the README.
2. Draft resume bullets, for example:
   - *"Built a segmented home lab (VirtualBox, Kali, Metasploitable2) and deployed Snort 3 NIDS with Splunk SIEM to detect simulated reconnaissance, brute-force, and exploitation activity; authored a custom Snort detection rule and a formal incident report mapping findings to MITRE ATT&CK."*
   - *"Investigated three simulated attack scenarios using Splunk and Wireshark, correlating IDS alerts with packet captures to reconstruct a full attack timeline."*
3. Be ready to walk through it live in an interview — this project holds up well under "walk me through what you built" because you'll have actually done every step yourself.

---

## 8. Optional stretch goals (once the core project is done)

- Swap the standalone Snort+Splunk setup for **Security Onion 3.1** (bundles Suricata, Zeek, and Elastic in one platform) to get exposure to a more production-like SOC toolkit.³
- Add a **phishing email analysis** mini-project — analyze a defanged phishing sample's headers and URLs, extract IOCs, and write it up the same way.
- Automate one small piece of response (e.g., a script that queries an IP against a threat intel API when a Snort alert fires) to dip a toe into SOAR-style thinking.

---

### Sources referenced while building this guide
1. Snort 3 installation guide — [docs.snort.org/start/installation](https://docs.snort.org/start/installation)
2. Splunk Free license details — [help.splunk.com — About Splunk Free](https://help.splunk.com/en/splunk-enterprise/administer/admin-manual/10.4/configure-splunk-licenses/about-splunk-free)
3. Security Onion 3.1 release notes — [blog.securityonion.net](https://blog.securityonion.net/2026/05/security-onion-310-now-available-with.html)
