# Enterprise Campus Network Architecture
**A Deep Dive into Resilient Infrastructure, Redundancy, and Scalable Design.**

---

## Project Overview
This repository documents the end-to-end design and implementation of a modern Enterprise Campus Network. The project is structured as an **Evolutionary Roadmap**, simulating the real-world growth of a corporate environment. 

The goal is to move beyond basic connectivity and solve for **Business Continuity**, **Security Segmentation**, and **Operational Efficiency**.

---

## The Architecture Roadmap

### 🔹 [Phase 1: The Baseline Foundation](./v1-foundation)
*Focus: Connectivity & Segmentation*
* **Topology:** Layer 3 Core-Centric Design.
* **Routing:** Inter-VLAN routing performed at the Core Layer via SVIs (Switch Virtual Interfaces), offloading routing tasks from the firewall.
* **Security:** Implemented VLAN-based segmentation (Management, HR, Sales, Guest) with strict Access Control Lists (ACLs) applied at the Core.
* **Key Achievement:** Established a secure, routed baseline with full documentation and IPAM (IP Address Management).



### 🔹 [Phase 2: High Availability & Redundancy](./v2-high-availability) (In progress)
*Focus: Eliminating Single Points of Failure*
* **Core Redundancy:** Deployed **HSRP (Hot Standby Router Protocol)** for seamless gateway failover.
* **Link Aggregation:** Implemented **LACP (802.3ad)** EtherChannels between Access and Core layers to double bandwidth and provide link-level resiliency.
* **Firewall HA:** Configured an Active/Standby Firewall cluster to ensure perimeter security survives hardware failure.
* **Key Achievement:** Reduced potential downtime from hours to sub-second failover.



### 🔹 [Phase 3: Service Delivery & DMZ](./v3-services-dmz) (Planned)
*Focus: Internal Services & Perimeter Security*
* **DMZ Implementation:** Isolated segment for public-facing assets (Nginx Web Servers).
* **Infrastructure Services:** Migration of DHCP/DNS from network appliances to dedicated Linux nodes.
* **Security Policies:** Fine-grained firewall inspection for East-West traffic between internal zones and the DMZ.

### 🔹 [Phase 4: Scalability & Wide Area Networking (WAN)](./v4-wan-scalability) (Planned)
*Focus: Multi-Site Connectivity & Routing Efficiency*
* **Virtual Branch Office:** Deployment of a remote site with local switching and routing.
* **Site-to-Site IPsec VPN:** Established a secure, encrypted tunnel over an untrusted "Cloud" transport.
* **Dynamic Routing (OSPF):** End-to-end reachability via OSPF with **Route Summarization** to optimize routing table size and CPU overhead.
* **Key Achievement:** Successfully simulated a corporate WAN environment with optimized traffic paths.



### 🔹 [Phase 5: Automation & Observability](./v5-automation-monitoring) (Planned)
*Focus: Modern Network Operations (NetOps)*
* **Configuration Management:** Developed **Python/Netmiko** scripts to automate multi-node configuration backups to a centralized management server.
* **Continuous Monitoring:** Integration of **Zabbix/LibreNMS** via Docker to track interface statistics and HSRP state changes.
* **Incident Response:** Configured SNMP Traps and Syslog alerts to provide real-time visibility into link failures.
* **Key Achievement:** Shifted from reactive troubleshooting to proactive monitoring and "Configuration-as-Code" principles.

---

## Technical Stack
* **Simulation:** GNS3 / VMware Workstation
* **Networking:** Cisco IOSv (Routers/Switches), Cisco ASAv (Firewalls)
* **Protocols:** OSPF, BGP, HSRP, LACP, 802.1Q (Trunking), NAT/PAT
* **Automation:** Python (Netmiko), Bash scripts

---

## Engineering Impact
* **Resiliency:** The network can lose a Core switch, a Firewall, or a trunk link without dropping a single user session.
* **Scalability:** The modular "Building Block" approach allows for adding new departments (VLANs) or branch offices with minimal reconfiguration.
* **Observability:** Integrated logging and capture points (Wireshark) allow for rapid troubleshooting of protocol-level issues.

---

## Repository Structure
```
Enterprise-Campus-Network-Architecture/
├── v1-foundation/             # Topology, diagrams, and initial configs
├── v2-high-availability/      # HSRP/LACP configs and failover tests
├── v3-services-dmz/           # DMZ design and server configs
├── v4-wan-scalability/        # VPN and OSPF summarization details
├── v5-automation-monitoring/  # Python scripts and NMS Docker files
└── README.md                  # Project Master Documentation
```
