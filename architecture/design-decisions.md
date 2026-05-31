# Architecture Design Decisions

**Project:** Healthcare SOC Home Lab  
**Status:** In progress  
**Last updated:** May 2026

---

## Overview

This document explains the reasoning behind every architectural decision made in this project. The goal is not just to build a working lab, but to understand why each component is placed where it is — mirroring the reasoning a security engineer would apply in a real healthcare environment.

## Current architecture diagram

![Architecture Diagram](diagram.png)

*Diagram reflects current build state. LAN internal structure will be updated as the lab progresses into the VLAN segmentation phase.*

---

## 1. Why a dual-firewall design?

The architecture uses two separate firewalls in series — PFSense at the edge and OPNSense internally — rather than a single firewall.

**Reasoning:**

- Defense-in-depth: if the edge firewall is compromised or misconfigured, the internal firewall provides a second enforcement point
- Vendor diversity: using different vendors at each layer means a zero-day or critical vulnerability in one product does not expose the entire network
- Separation of concerns: the edge firewall handles north-south perimeter filtering, while the internal firewall focuses on internal threat detection and will enforce east-west segmentation in a future phase
- In healthcare environments, segmentation between the DMZ (public-facing services) and internal clinical systems is a HIPAA technical safeguard requirement

---

## 2. Why PFSense + Snort at the edge?

PFSense was chosen as the edge firewall with Snort integrated as the IDS/IPS engine.

**Reasoning:**

- Snort has more mature and actively maintained ruleset support for perimeter threats, particularly the Talos Intelligence ruleset from Cisco, which is well-suited for north-south traffic inspection
- PFSense has strong community support, extensive documentation, and is widely deployed in enterprise environments — making it a relevant skill to develop
- Snort's inline IPS mode at the perimeter allows blocking of known malicious traffic before it reaches the internal network

---

## 3. Why OPNSense + Suricata as a co-deployed internal unit?

OPNSense and Suricata are deployed together as a single internal enforcement and detection layer. The diagram reflects this as a combined unit — OPNSense as the firewall and Suricata as the co-resident IDS/IPS and primary NDR engine.

**Reasoning:**

- Suricata is multi-threaded and handles internal traffic inspection more efficiently than Snort in high-throughput environments
- Suricata's protocol analysis capabilities (HTTP, DNS, TLS, SMB, HL7) are more relevant for detecting lateral movement and internal threats in a healthcare network
- Co-deploying Suricata with OPNSense (rather than as a separate VM) reduces architectural complexity while keeping full east-west visibility
- Using a different vendor and IDS engine at the internal layer adds another layer of vendor diversity relative to the PFSense+Snort perimeter
- OPNSense is already positioned to enforce VLAN-based east-west segmentation in a future phase without requiring architectural changes

---

## 4. Why is the primary NDR positioned internally, not just at the perimeter?

This was a deliberate and considered decision based on how real healthcare breaches actually unfold.

**Reasoning:**

- The majority of significant healthcare breaches — ransomware, insider threats, EHR data exfiltration — occur after initial access, during the lateral movement phase inside the network
- A perimeter-only NDR will detect the front door being opened but miss everything that happens inside
- HIPAA and HITECH audit requirements focus heavily on detecting unauthorized access to PHI, which almost always involves internal movement after initial compromise
- MITRE ATT&CK for healthcare threat actors shows that post-compromise techniques (lateral movement, credential dumping, data staging) are where detection opportunities are highest
- The internal firewall sees all traffic passing through it, making OPNSense+Suricata the highest-value detection point in the current architecture

---

## 5. Why was the passive WAN Suricata tap removed?

An earlier version of this architecture included a passive Suricata instance operating in promiscuous mode on the WAN-side segment, before PFSense. This has been removed in the current design.

**Reasoning:**

- The passive WAN tap provided visibility into raw internet traffic (reconnaissance, blocked exploit attempts) but this signal is low-priority relative to the detection value provided by the internal OPNSense+Suricata layer
- Removing it reduces VM count and operational complexity without materially reducing detection capability — the most important threats are post-initial-access events inside the network, not pre-firewall internet noise
- PFSense+Snort at the perimeter still provides north-south detection; adding a pre-firewall passive sensor is a future enhancement once the core SOC stack is stable and generating data
- This is a deliberate simplification for the current build phase, not a permanent omission

---

## 6. Why include a HoneyNet?

A HoneyNet of decoy systems is connected directly off PFSense in its own isolated segment.

**Reasoning:**

- Honeypots in healthcare environments are an effective low-noise detection mechanism — any connection to a decoy system is inherently suspicious and warrants investigation
- They provide early warning of reconnaissance: an attacker probing the network will eventually hit a decoy
- They generate high-fidelity alerts with very low false positive rates, reducing analyst fatigue
- In a lab context, they also serve as safe targets for simulating attacker behavior without risking actual systems

**Current HoneyNet composition (as deployed):**

- Fake EHR node — mimics a clinical application server; any probe is an immediate IOC
- Fake file server — decoy share with PHI-named files; any access attempt is a guaranteed indicator
- Fake admin endpoint — SSH/RDP listener; any authentication attempt triggers an alert

**Critical PFSense rule:** all outbound traffic from the HoneyNet segment is hard-blocked. A honeypot that can make outbound connections is a liability — if compromised, it must not be usable as a pivot or beacon point.

---

## 7. Why this DMZ composition?

The DMZ is connected off PFSense and contains public-facing services that simulate a realistic healthcare internet presence.

**Current DMZ composition (as deployed):**

- EHR portal (nginx, HTTP/443) — the primary web attack surface; target for credential stuffing, SQL injection, and web application exploitation scenarios
- VPN Gateway — simulates remote clinician access, one of the most common initial access vectors in real healthcare breaches
- Reverse proxy (nginx) — routes traffic cleanly and adds realism for HTTP-based detection scenarios

**Reasoning:** The DMZ is intentionally isolated from the LAN by both PFSense (perimeter) and OPNSense (internal). Any traffic attempting to cross from DMZ to LAN is anomalous by default and triggers detection. This separation mirrors how real healthcare perimeter networks are architected under HIPAA technical safeguard requirements.

---

## 8. Current LAN design

The LAN is currently a flat network behind OPNSense. This is an intentional simplification for the initial build phase — the focus at this stage is getting core detection, logging, and SOC tooling operational before layering in network complexity.

**Current LAN composition (as deployed):**

- Windows workstations — simulate clinician endpoints; Wazuh agents and Sysmon will be deployed for endpoint visibility
- OpenEHR server — simulates the primary clinical application and PHI store; the crown jewel asset in all attack scenarios
- Active Directory / DNS / supporting infrastructure — domain-joined environment necessary for realistic credential-based attack chains (pass-the-hash, Kerberoasting, domain dominance)

**Why OpenEHR?** OpenEHR is an open clinical data standard and platform. It provides a more standards-compliant EHR simulation with realistic clinical data structures, making it a better representation of modern healthcare EHR architecture for detection scenario purposes compared to simpler alternatives.

**Why flat LAN for now?**

- Getting Wazuh agents, Suricata, and the full SOC stack operational and generating real data is the current priority
- A flat network still provides meaningful detection surface — all traffic is visible to OPNSense+Suricata and all endpoints can be monitored by Wazuh
- VLAN segmentation adds operational complexity (OPNSense trunk configuration, inter-VLAN routing rules, agent connectivity) that is better tackled once the baseline is stable and understood

---

## 9. Planned LAN segmentation — future phase

Once the flat LAN baseline is stable, the LAN will be split into dedicated segments enforced by OPNSense. This is a planned architectural evolution, not an oversight in the current design.

**Planned Clinical segment — PHI assets:**

- Windows workstations (Wazuh agent + Sysmon)
- OpenEHR application server
- MySQL/MariaDB database server — deliberately a separate VM from the app server so that direct DB connections from workstations are anomalous and detectable by Suricata
- HAPI FHIR server (port 8080) — HL7/FHIR API endpoint for modern EHR API abuse scenarios
- Windows Server (Active Directory, DNS, DHCP)

**Planned Management segment — SOC tools, no direct PHI:**

- Wazuh SIEM (all-in-one)
- SOC analyst station
- Jump server / bastion — the only OPNSense-permitted path into the clinical segment from management
- Backup server — populated with fake PHI-named files as detection bait for ransomware scenarios
- Full SOAR + CTI toolstack (IRIS, n8n, Cortex, MISP, OpenCTI, Velociraptor)

**Planned IoMT segment — medical device simulation:**

- Passive Suricata sensor (promiscuous mode, not inline — active scanning can crash real medical devices)
- Simulated devices: infusion pump (Python/HL7 v2), PACS server (Orthanc DICOM), patient monitor, legacy Windows XP VM representing unremediable devices
- Strict OPNSense whitelist: each device communicates only with its designated management server; no device-to-device traffic, no internet access

**Critical future OPNSense rule:** Clinical segment can only reach the Management segment on Wazuh agent ports 1514/1515. This prevents a compromised clinical workstation from pivoting directly to the SIEM.

---

## 10. Why is Wazuh the SIEM?

All log sources ship to a central Wazuh instance (Manager + Indexer + Dashboard, all-in-one deployment) before reaching the SOC.

**Reasoning:**

- Individual tools (Snort, Suricata, firewall logs) produce alerts in isolation — Wazuh correlates events across sources to identify multi-stage attacks that no single tool would detect alone
- A SIEM provides the timeline reconstruction capability needed for incident response
- For healthcare compliance, Wazuh provides audit logging and reporting capabilities required by HIPAA Security Rule §164.312(b)
- Wazuh is open source, has native HIPAA compliance dashboards, supports Suricata and Snort log ingestion via Filebeat, and provides endpoint agent deployment for LAN hosts
- Wazuh's built-in anomaly detection activates as data accumulates, providing a latent ML detection layer without requiring a separate platform

**The most critical integration in the architecture:** Wazuh fires webhooks to n8n on rule match, triggering the entire automated SOAR pipeline. Nothing in the SOC automation works without this connection being correctly configured.

---

## 11. Why this SOAR stack?

The SOAR layer consists of IRIS + n8n + Cortex + Velociraptor. Each tool has a single clear responsibility with no overlap.

### n8n — automation engine

n8n is the central orchestrator. When Wazuh fires a webhook, n8n decides what happens next: extract the IOC, call Cortex for external scoring, query MISP for recognition, pull OpenCTI context, create an IRIS case with all enriched data, and if the alert is critical, launch a Velociraptor hunt.

**Why n8n over alternatives (Shuffle, TheHive+Cortex native)?**

- n8n provides visual workflow building that makes the automation logic auditable and educational
- It is not tightly coupled to any single tool — it can call any API, making it resilient to tool changes
- It acts as the single integration hub, keeping direct tool-to-tool connections minimal

### IRIS — case management

IRIS handles incident case creation, timeline reconstruction, and evidence tracking. It receives enriched case data from n8n (not directly from Wazuh) and receives forensic evidence back from Velociraptor after a hunt completes.

**Important:** IRIS does not trigger Velociraptor. n8n triggers Velociraptor. Velociraptor returns evidence to IRIS.

### Cortex — IOC analyzer

Cortex is called by n8n before case creation. It takes a raw IOC (IP, hash, domain) and runs it through external analyzers (VirusTotal, Shodan, AbuseIPDB, PassiveDNS) to produce an enrichment score. That score is returned to n8n and included in the IRIS case. Cortex answers: "is this IOC publicly flagged, and what do external sources say about it?"

### Velociraptor — DFIR and endpoint hunting

Velociraptor is triggered by n8n on critical alerts. It performs live forensics, memory artifact collection, and VQL-based hunting across all LAN endpoints without requiring physical access. Evidence is automatically returned to the IRIS case. In a ransomware scenario, Velociraptor is the first tool reached for post-compromise investigation.

---

## 12. Why this CTI stack?

The CTI layer consists of MISP + OpenCTI + a custom-built CTI dashboard. MISP and OpenCTI are not redundant — they operate at different levels of abstraction.

### MISP — operational threat intelligence

MISP is an IOC database. It stores concrete, actionable indicators and correlates them against ingested feeds. It answers: "is this specific IP/hash/domain known malicious, and have others seen it?"

- Ingests H-ISAC healthcare-specific threat feeds
- Feeds Cortex analyzers with healthcare-relevant IOC context
- First CTI tool to deploy — everything else depends on it having populated data

### OpenCTI — strategic threat intelligence

OpenCTI is a knowledge graph of relationships between threat actors, campaigns, TTPs, and infrastructure. It answers: "who is using this IOC, what campaign does it belong to, and what do they typically do next?"

- STIX2 native — all objects are structured and machine-readable
- ATT&CK kill chain mapping — positions detections within the MITRE framework
- Syncs bidirectionally with MISP
- Add after MISP is stable and data is flowing — OpenCTI without populated MISP is less useful

**Why both?** MISP tells you what is hitting you. OpenCTI tells you who is hitting you and what they will do next. In a healthcare SOC investigation, MISP provides the immediate IOC match in the first five minutes; OpenCTI shapes the threat model and playbook selection. They answer different questions at different points in the investigation.

### Custom CTI dashboard — analyst-facing single pane of glass

A custom React dashboard backed by a FastAPI middleware layer. It surfaces the most operationally relevant output from MISP, OpenCTI, and Wazuh active alerts in one view, without requiring the analyst to navigate three separate UIs during an active incident.

**Why a FastAPI middleware?**

- MISP and OpenCTI use different API schemas (REST vs GraphQL/STIX2); the middleware normalizes both into clean JSON
- API keys and tokens are stored server-side only — never in browser-accessible code
- Response caching (60s for IOCs, 5min for actors/campaigns) keeps the dashboard responsive without hammering CTI tools
- The dashboard exposes three endpoint types: `/api/iocs`, `/api/actors`, `/api/alerts/active`

**Why not Streamlit?** Streamlit is a dashboard framework — it would require rebuilding what OpenCTI already provides natively. OpenCTI is the professional-grade CTI platform; the custom dashboard is the operational analyst view on top of it, not a replacement for either CTI tool.

---

## 13. Why no dedicated ML platform?

ML for anomaly detection is not a separate platform in this architecture — it is a latent capability inside existing tools.

**Reasoning:**

- ML requires data volume and historical baseline to be useful; in the early phases of the lab there is insufficient data to train or tune anything meaningfully
- Wazuh's built-in anomaly detection activates as logs accumulate and can be tuned without a separate tool
- OpenCTI includes an ML-based IOC confidence scoring module that activates once MISP sync is populated

**Future addition:** once 3-6 months of lab traffic exists, Malcolm (CISA's network traffic analysis platform — Zeek + Arkime + ML anomaly detection) can be added alongside the Suricata NDR. It is designed exactly for this position in the architecture.

---

## 14. SOC build order

The SOC layer has internal dependencies. Build in this order to avoid integration failures:

1. **Wazuh all-in-one** — everything depends on it; deploy first, connect all log sources before proceeding
2. **MISP** — get H-ISAC healthcare feeds ingesting and verify IOC population before adding analysis tools
3. **IRIS + n8n + Cortex** — wire the Wazuh webhook → n8n → IRIS case creation + Cortex IOC enrichment pipeline
4. **Velociraptor** — deploy agents on all LAN endpoints, wire n8n trigger on critical alert rule match
5. **OpenCTI** — sync bidirectionally with MISP, build ATT&CK dashboards
6. **Custom CTI dashboard** — deploy FastAPI middleware, wire MISP + OpenCTI + Wazuh alert feeds, build analyst UI last

---

## Network segment summary

| Segment | Firewall | Status | Key assets |
|---|---|---|---|
| HoneyNet | PFSense | Deployed | Fake EHR node, fake file server, fake admin endpoint |
| DMZ | PFSense | Deployed | EHR portal, VPN Gateway, reverse proxy |
| LAN | OPNSense + Suricata | Deployed — flat network | Workstations, OpenEHR server, AD/DNS |
| Clinical segment | OPNSense + Suricata | Planned — future phase | OpenEHR, MySQL, FHIR, AD, workstations |
| Management segment | OPNSense + Suricata | Planned — future phase | Wazuh, IRIS, n8n, Cortex, MISP, OpenCTI, Velociraptor, jump server, backup |
| IoMT segment | OPNSense + Suricata | Planned — future phase | Device simulators, Orthanc PACS, legacy XP VM |

---

## Open questions / future decisions

- [ ] Deploy Wazuh agents + Sysmon on all LAN endpoints
- [ ] Connect all log sources to Wazuh (Suricata, Snort, PFSense logs, OPNSense logs)
- [ ] Wire Wazuh → n8n webhook and build initial automation workflows
- [ ] Configure H-ISAC feed ingestion into MISP
- [ ] Build FastAPI middleware and custom CTI dashboard
- [ ] Plan and execute LAN segmentation into Clinical / Management / IoMT segments
- [ ] Define OPNSense ruleset for inter-segment traffic once segmentation is in place
- [ ] Select Suricata rulesets for healthcare protocol coverage (HL7, FHIR, DICOM)
- [ ] Define SOC playbooks for key detection scenarios (ransomware, credential dump, PHI exfiltration, IoMT anomaly)
- [ ] Define Velociraptor VQL hunt queries for ransomware and lateral movement IOCs
- [ ] Evaluate Malcolm addition after 3-6 months of lab data for ML anomaly detection
- [ ] Evaluate re-adding passive WAN Suricata tap once core SOC stack is stable