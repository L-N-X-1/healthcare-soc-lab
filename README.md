# Healthcare SOC Home Lab

## Project Overview
A simulated healthcare SOC environment built on VMware, 
designed to practice real-world threat detection and 
incident response in a HIPAA-relevant context.

## Architecture
- Edge firewall: PFSense + Snort (IDS/IPS)
- Internal firewall + primary NDR: OPNSense + Suricata (east-west)
- Passive NDR: Suricata (north-south, pre-firewall tap)
- Deception: HoneyNet
- SIEM: Wazuh
- Zones: DMZ, LAN, SOC

## Architecture Diagram
![Architecture](architecture/diagram.png)

## Design Decisions
See [design-decisions.md](architecture/design-decisions.md)

## Status
🔧 In progress — currently completing network segmentation phase
