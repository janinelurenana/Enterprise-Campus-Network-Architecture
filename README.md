# Enterprise Campus Network Architecture

**Designing a Resilient, Segmented, and Scalable Enterprise Network**

---

## Project Overview

This repository documents the staged design and implementation of a simulated Enterprise Campus Network.

Rather than building a static lab, the project follows a structured evolution model — mirroring how real corporate networks mature over time:

1. Establish a secure routed baseline
2. Eliminate single points of failure
3. Isolate and harden exposed services
4. Scale across sites
5. Introduce automation and operational visibility

The objective is not just connectivity — but controlled growth, resilience, and measurable operational improvement.

---

## Architecture Roadmap

### 🔹 Phase 1: Foundation – Segmentation & Core Routing

`v1-segmentation-vlan-acl-firewall`

**Focus:** Establishing a secure, routed baseline

* **Topology:** Core-centric Layer 3 campus design
* **Inter-VLAN Routing:** Performed at the Core via SVIs (firewall reserved for policy and NAT)
* **Segmentation:** VLAN-based isolation (Management, HR, Sales, Guest)
* **Access Control:** ACL enforcement at the Core for East-West traffic
* **Documentation:** IP addressing plan, gateway design, verification tests

**Outcome:**
A stable, segmented network with defined trust boundaries and least-privilege enforcement.

---

### 🔹 Phase 2: High Availability & Redundancy

`v2-resilient-core-ha-firewall`

**Focus:** Removing single points of failure

* **Gateway Redundancy:** HSRP for default gateway failover
* **Link Resiliency:** LACP EtherChannels between Access and Core
* **Perimeter Redundancy:** Active/Standby Firewall HA
* **Failure Testing:** Link pulls, device shutdown simulations, convergence observation

**Outcome:**
Gateway and perimeter failover without user-facing disruption. Sub-second recovery under tested conditions.

---

### 🔹 Phase 3: Service Isolation & DMZ

`v3-perimeter-dmz` *(In Progress)*

**Focus:** Controlled service exposure

* Dedicated DMZ segment for public-facing services (e.g., Nginx web servers)
* Separation of internal services from exposed infrastructure
* Strict firewall policy enforcement between Internal Zones and DMZ
* Logging and inspection for inter-zone traffic

**Outcome:**
Clear trust boundary between internal users and externally accessible systems.

---

### 🔹 Phase 4: WAN & Dynamic Routing

`v4-transit-ospf-vpn` *(Planned)*

**Focus:** Multi-site scalability

* Simulated remote branch deployment
* Site-to-Site IPsec VPN over untrusted transport
* OSPF for dynamic routing across sites
* Route summarization to reduce routing table size and improve stability

**Outcome:**
Enterprise-style WAN connectivity with scalable routing design.

---

### 🔹 Phase 5: Automation & Observability

`v5-netops-orchestration-python-docker` *(Planned)*

**Focus:** Operational maturity

* Python (Netmiko) scripts for automated configuration backups
* Dockerized monitoring stack (Zabbix / LibreNMS)
* SNMP traps and Syslog integration
* Monitoring of interface states, HSRP events, and failover transitions

**Outcome:**
Shift from reactive troubleshooting to proactive monitoring and configuration management.

---

## Technical Stack

**Simulation Environment**

* GNS3
* VMware Workstation

**Network Devices**

* Cisco IOSv (Switching & Routing)
* FortiGate (Firewall, NAT, HA)

**Protocols & Technologies**

* OSPF
* HSRP
* LACP (802.3ad)
* 802.1Q Trunking
* NAT / PAT
* IPsec VPN

**Automation & Tooling**

* Python (Netmiko)
* Bash scripting
* Docker-based monitoring stack

---

## Engineering Impact

### Resiliency

Core switches, firewall nodes, or trunk links can fail without interrupting active sessions (validated via failover testing).

### Scalability

Modular VLAN and routing design allows additional departments or branch offices with minimal architectural change.

### Security

Clear trust zones enforced via ACLs and firewall policy. Default-deny posture implemented.

### Observability

Integrated logging, packet captures, and monitoring tools reduce troubleshooting time and provide protocol-level visibility.

---

## Repository Structure

```
Enterprise-Campus-Network-Architecture/
├── v1-segmentation-vlan-acl-firewall/
├── v2-resilient-core-ha-firewall/
├── v3-perimeter-dmz/
├── v4-transit-ospf-vpn/
├── v5-netops-orchestration-python-docker/
└── README.md
```

Each phase contains:

* Topology diagrams
* Design rationale
* Addressing plans
* Sanitized configurations
* Verification and failure test documentation
* Lessons learned

---

## Author

Christelle Janine M. Lureñana

Designed and implemented VLANs, ACLs, HSRP, firewall HA, DMZ, OSPF/VPN, and automation in GNS3 with verified failover.
