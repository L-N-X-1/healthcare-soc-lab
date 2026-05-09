# Healthcare SOC Home Lab

## Project Overview
A simulated healthcare SOC environment built on VMware, designed to practice
real-world threat detection and incident response in a HIPAA-relevant context.

## Architecture

**Network layer**
- Edge firewall: PFSense + Snort (IDS/IPS, north-south)
- Internal firewall + primary NDR: OPNSense + Suricata (east-west)
- Passive NDR: Suricata (WAN tap, pre-firewall, promiscuous mode)

**Segments**
- DMZ: EHR portal, VPN Gateway, reverse proxy
- HoneyNet: fake EHR node, fake file server, fake admin endpoint
- LAN: OpenEHR server, workstations, AD/DNS *(flat network — VLAN segmentation planned)*

**SOC stack**
- SIEM: Wazuh (Manager + Indexer + Dashboard)
- SOAR: n8n · IRIS · Cortex · Velociraptor
- CTI: MISP · OpenCTI · custom dashboard (FastAPI + React)

## Architecture Diagram
![Architecture](architecture/diagram.png)

## Design Decisions
See [design-decisions.md](architecture/design-decisions.md)

## Status
🔧 In progress — LAN baseline deployed, SOC tooling phase next

## Roadmap
- [ ] Deploy Wazuh agents + Sysmon on all LAN endpoints
- [ ] Connect all log sources to Wazuh
- [ ] Wire Wazuh → n8n webhook and build automation workflows
- [ ] Configure MISP with H-ISAC healthcare feeds
- [ ] Build FastAPI middleware + custom CTI dashboard
- [ ] LAN segmentation: Clinical / Management / IoMT segments
