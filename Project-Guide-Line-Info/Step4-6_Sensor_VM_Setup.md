# Setting Up the Sensor VM (Ubuntu Server + Snort 3 + Splunk Free)

Pick up here whenever you're ready. Kali (`192.168.56.101`) and Metasploitable2 (`192.168.56.102`) are already confirmed working and pinging each other — this VM is the third leg: the box that *watches* their conversation.

---

## Step 4 — Deploy the Ubuntu Server VM

**Why Ubuntu Server (not Desktop):** you don't need a GUI on a sensor — it just burns RAM and disk you'd rather give to Snort/Splunk. Everything you'll do here is command-line.

1. Download **Ubuntu Server 22.04 or 24.04 LTS** ISO from ubuntu.com/download/server.
2. **Machine → New** in VirtualBox. This time you *do* want the ISO — you're installing a fresh OS, not attaching a pre-built disk (opposite situation from Metasploitable2).
3. Point the wizard's ISO field at the Ubuntu Server ISO. Give it more resources than the other two — **2–4GB RAM minimum**, ideally more if your host can spare it. Splunk in particular is not lightweight; this is the one VM in the lab where under-provisioning will actually slow you down.
4. Walk through the installer defaults. When it asks about networking during install, don't worry about getting it perfect — you'll fix/confirm this in step 5 below regardless.
5. Create a user account you'll remember (write it down — no default creds to fall back on here, unlike Kali or Metasploitable2).

**Once installed and booted, before anything else — fix networking and promiscuous mode:**

6. Shut the VM down, then in VirtualBox: **Settings → Network → Adapter 1**
   - **Attached to:** Host-only Adapter
   - **Name:** the *same* adapter Kali and Metasploitable2 are on (double-check the name matches — this is the exact mistake you already caught twice, so you know what to look for)
   - **Advanced → Promiscuous Mode: Allow All** — this is the one VM in the lab where this setting matters, per the reasoning from earlier (it needs to see frames not addressed to its own MAC).
7. Boot back up, run `ip a`, confirm you land in `192.168.56.0/24`.
8. From Kali, `ping` this new address to confirm three-way connectivity.

---

## Step 5 — Install and Configure Snort 3

**What Snort actually does, briefly:** it's a Network Intrusion Detection System (NIDS) — it reads raw traffic off an interface and compares it against a set of rules (signatures of known-bad patterns), generating an alert when something matches. This is distinct from a firewall — Snort in NIDS mode *observes and logs*, it doesn't block anything by default.

1. Follow the official install guide for your Ubuntu version: **docs.snort.org/start/installation**. The steps vary slightly by Ubuntu release, so working from the source here is more reliable than a paraphrase.
2. Once installed, edit Snort's config (`snort.lua` in Snort 3):
   - Set **`HOME_NET`** to `192.168.56.0/24`. This tells Snort "this is my protected network" — it uses this to distinguish inbound/outbound traffic direction in rules, so getting it wrong silently breaks a lot of rule logic.
   - Enable the **community rule set** as your baseline (free, maintained, decent coverage of common attack signatures).
3. Run Snort in **NIDS mode** against your `eth0` (or whatever the host-only interface is named), logging alerts to a local file. The install guide will have the exact command syntax for your version.
4. **Write one custom rule.** A good beginner target: alert on any SYN-only Nmap-style scan pattern, or on any connection attempt to port 21 (FTP) — both map directly to Scenario A/C later. This is the step the guide called out as a genuine resume differentiator, so don't skip it even though it's optional-feeling.

**Checkpoint for this step:** run a basic `nmap` scan from Kali against Metasploitable2, and confirm an alert lands in Snort's log file. If nothing appears, that's useful information too — it might mean `HOME_NET` is misconfigured, the interface Snort is listening on is wrong, or the community ruleset doesn't cover that specific scan flag combination (worth testing a plain `nmap -sS` first before more exotic flags).

---

## Step 6 — Install Splunk Free and Ingest Snort Alerts

**What Splunk is doing here, conceptually:** Snort's alert log is just a text file. Splunk's job is to ingest that file, index it so it's searchable, and let you correlate/aggregate across events — turning "a pile of log lines" into "an investigation."

1. Download **Splunk Free** from splunk.com — no license purchase needed, the free tier (500MB/day ingest) is far more than a home lab generates.
2. Install it directly on this same sensor VM (simplest — one less VM to manage and network correctly).
3. Log into Splunk's web interface (default port 8000) and add a **monitor input**:
   - Settings → Data Inputs → Files & Directories → point it at Snort's alert log file path.
4. Confirm ingestion with a simple search:
   ```
   index=main sourcetype=snort_alert
   ```
   If this returns nothing, check: is the sourcetype actually named `snort_alert`, or did Splunk auto-assign something else during setup? (Check under Settings → Data Inputs to see what it actually picked.)
5. **Optional but worth doing:** build one dashboard panel — alert count over time, top source IPs, or top signatures. This becomes your first real screenshot for the GitHub README.

**Checkpoint for the whole section:** run a scan from Kali → see it in Snort's raw log → see the same event searchable in Splunk. That chain (attack → detection → indexed evidence) is the technical spine of the entire project — once it works end to end, Section 3 (running the three actual attack scenarios) is mostly repeating this same loop three times with different techniques.

---

### When you're back
Send a screenshot of `ip a` once the Ubuntu Server VM is up and networked, and we'll verify the subnet before you get into the Snort install — same habit as the last two VMs.
