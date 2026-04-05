# Enterprise Campus Network – Segmentation, Redundancy, and Perimeter Security

## Overview

This project documents the end-to-end design, implementation, and validation of a campus-scale enterprise network built to production standards. The architecture supports multi-department internal users, enforces strict east-west access controls, exposes public-facing services through an isolated DMZ, and eliminates single points of failure across both the switching core and the perimeter firewall.

The network was built entirely in a lab environment using Cisco IOS Layer 2/3 switches and FortiGate VM appliances, with all configurations written, tested, and validated manually. The goal was not to demonstrate tool familiarity, but to reason through the decisions that define how a real network is built: what to segment, what to permit, where redundancy is necessary, and where complexity is a liability.

---

## What This Network Does

The campus network serves four internal departments — IT, Management, HR, and Guest — each isolated into its own VLAN and subnet. A collapsed core/distribution layer performs inter-VLAN routing and enforces east-west access policy via stateless ACLs applied at SVI ingress. A FortiGate Active-Passive cluster handles perimeter inspection, NAT, and public-service exposure. A DMZ segment hosts a public-facing Nginx web server, isolated from internal VLANs at both the firewall and the switch fabric level.

The design reflects several operational priorities: deterministic traffic paths, measurable failover behavior, minimal policy surface on the access layer, and clear separation of trust zones.

---

## Architecture

### Topology Diagram

![Network Topology Diagram](/toplogy/topology.png)

---

### Functional Layers

**Access Layer (ACC-1, ACC-2, ACC-3)**
Strictly Layer 2. Each switch carries only the VLANs relevant to its connected endpoints, enforced via trunk pruning. No routing, no ACLs, no policy logic. Portfast and BPDU Guard are enabled on all host-facing ports. Each access switch uplinks to both core switches (X-mesh), with Rapid-PVST+ managing loop prevention.

**Core Layer (CORE-1, CORE-2)**
Collapsed core/distribution. Both switches host SVIs for all four VLANs and participate in HSRP per-VLAN. CORE-1 is the primary root bridge (STP priority 4096) and primary HSRP active (priority 110). CORE-2 is the secondary (STP priority 8192, default HSRP priority). A Layer 2 Port-Channel using LACP connects the two cores. Each core also maintains a routed uplink to the FortiGate cluster — CORE-1 to FG port2, CORE-2 to FG port5 — with cross-links providing backup paths.

Inbound ACLs applied at each SVI enforce east-west policy at wire speed. Static default routes point to the FortiGate, with tracked object failover via IP SLA monitoring the upstream gateway.

**Perimeter (FortiGate HA Cluster)**
Two FortiGate VM instances in Active-Passive mode. The primary unit (priority 200) handles all traffic; the secondary (priority 100) remains in standby with configuration and session state synchronized. Monitored interfaces trigger automatic failover. The firewall maintains static routes into the internal 10.10.0.0/16 space via both core switches, with primary/backup path preference set by administrative distance.

Firewall zones separate WAN, internal subnets, and the DMZ. Policy is deny-by-default; explicit rules permit internet access from internal VLANs (with NAT) and HTTP/HTTPS access from WAN to the DMZ web server.

**DMZ (VLAN 50)**
A dedicated Layer 2 switch connects the DMZ segment to the FortiGate. The Nginx server is reachable from WAN via explicit firewall policy. No path exists from the DMZ to internal VLANs — denied at the firewall zone level and at the core ACLs. The DMZ switch has no uplink to the core switching fabric.

---

## VLAN and IP Addressing

| VLAN | Name       | Subnet          | Gateway      | Trust Level  |
|-----:|------------|-----------------|--------------|--------------|
| 10   | IT         | 10.10.10.0/24   | 10.10.10.1   | High         |
| 20   | Management | 10.10.20.0/24   | 10.10.20.1   | High         |
| 30   | HR         | 10.10.30.0/24   | 10.10.30.1   | Medium       |
| 40   | Guest      | 10.10.40.0/24   | 10.10.40.1   | Untrusted    |
| 50   | DMZ        | 10.10.50.0/24   | 10.10.50.1   | External     |

### Core ↔ Firewall Transit Links (/30)

| Link           | Subnet           | Core IP       | Firewall IP   |
|----------------|------------------|---------------|---------------|
| CORE-1 ↔ FG-1  | 10.10.255.0/30   | 10.10.255.1   | 10.10.255.2   |
| CORE-2 ↔ FG-2  | 10.10.255.4/30   | 10.10.255.5   | 10.10.255.6   |
| CORE-1 ↔ FG-2  | 10.10.255.8/30   | 10.10.255.9   | 10.10.255.10  |
| CORE-2 ↔ FG-1  | 10.10.255.12/30  | 10.10.255.13  | 10.10.255.14  |

---

## East-West Access Policy

ACLs are applied inbound on each SVI. Policy is expressed in terms of what is permitted; all other traffic is dropped and logged.

| Source \ Destination | IT  | Management | HR  | Guest | Internet |
|----------------------|-----|------------|-----|-------|----------|
| IT                   | —   | ✅          | ✅   | ❌     | ✅        |
| Management           | ❌   | —          | ✅   | ❌     | ✅        |
| HR                   | ❌   | ❌          | —   | ❌     | ✅        |
| Guest                | ❌   | ❌          | ❌   | —     | ✅        |

Return traffic is permitted via explicit `permit tcp any any established` and `permit icmp any any echo-reply` rules at the top of each ACL. ACLs are stateless; the firewall handles stateful inspection for perimeter traffic.

---

## Redundancy Mechanisms

### HSRP (Default Gateway Redundancy)

One HSRP group per VLAN. Virtual IPs use the .1 address consistently across all subnets. CORE-1 holds active for all groups under normal operation. Preemption is enabled with a 60-second delay on CORE-1 to prevent flapping during startup. HSRP tracking via IP SLA decrements CORE-1's priority by 25 if the upstream gateway becomes unreachable, triggering failover to CORE-2.

**Measured convergence:** 5 ICMP packets lost during CORE-1 shutdown under default hello/hold timers. End hosts required no reconfiguration.

### LACP Inter-Core Backbone

Physical links between CORE-1 and CORE-2 are bundled into a Port-Channel using 802.3ad LACP. This provides both bandwidth aggregation and link-level redundancy on the backbone segment. Individual link failure does not require routing reconvergence.

### Rapid-PVST+ and the X-Mesh Design

Each access switch uplinks to both CORE-1 and CORE-2, creating deliberate Layer 2 loops. Rapid-PVST+ manages loop prevention per-VLAN. Primary uplinks forward; cross-links remain blocking until needed. On link failure, Rapid-PVST+ transitions the blocked cross-link to forwarding without requiring a full topology recalculation.

CORE-1 is configured as root bridge for all VLANs (priority 4096). CORE-2 serves as secondary root (priority 8192). This deterministic configuration ensures that STP behavior is predictable under any single-link failure scenario.

### FortiGate Active-Passive HA

Session synchronization is enabled, so in-progress connections survive failover. The primary unit monitors four interface groups; failure of any monitored interface triggers failover to the secondary. A shared virtual MAC is used, so ARP tables on connected devices do not need to update.

**Measured convergence:** 1 ICMP packet lost during active firewall shutdown.

---

## Security Design

### Segmentation Philosophy

Segmentation is enforced at multiple layers, not just at one control point. The access layer limits VLAN scope per switch. The core enforces inter-VLAN policy at SVI ingress. The firewall enforces zone-to-zone policy. This means no single misconfiguration opens a path between trust zones.

### DMZ Isolation

The DMZ is not connected to the core switching fabric. The only path into or out of the DMZ is through the FortiGate. Firewall policy permits WAN-to-DMZ traffic on HTTP and HTTPS only. DMZ-to-internal traffic is denied at the firewall zone level. If the DMZ host is compromised, the attacker has no routed path to internal VLANs.

### Guest VLAN Hardening

Guest traffic (VLAN 40) is permitted to reach the internet but denied access to all internal subnets. ICMP unreachables are suppressed on the Guest SVI, so probes into internal space result in timeouts rather than explicit rejections. This forces an attacker into slower, less certain reconnaissance.

### ACL Design Decisions

ACLs use named extended lists applied inbound at the SVI. This placement intercepts traffic as it enters the routed domain, before it can be forwarded to any destination. Stateless ACLs require explicit return-path allowances, which are included at the top of each list. The explicit final `deny ip any any` documents intent and ensures logging captures any unexpected traffic.

### Firewall Policy

The GUEST_to_INTERNAL_DENY rule blocks Guest-to-internal traffic at the perimeter as a second enforcement point, independent of the ACL. The INTERNAL_to_WAN rule restricts outbound services to HTTP, HTTPS, DNS, and ICMP — no blanket internet access. NAT is applied outbound, masking internal addressing from external observers.

---

## Failure Testing

All tests were conducted under continuous ICMP traffic with default protocol timers. No tuning was applied; results reflect baseline convergence behavior.

| Test Scenario                    | Mechanism               | Packets Lost | Recovery        |
|----------------------------------|-------------------------|-------------|-----------------|
| CORE-1 shutdown                  | HSRP failover           | 5           | Automatic       |
| Active FortiGate shutdown        | FG HA failover          | 1           | Automatic       |
| Access uplink failure            | Rapid-PVST+ reconvergence | <3         | Sub-second      |
| CORE-1 + Active FG simultaneous  | Layered redundancy      | Measured    | Independent     |

The dual-failure test validates that HSRP and FG HA operate independently — a simultaneous core and firewall failure does not create a cascading outage, because the two mechanisms do not share state or timers.

Full test logs, packet captures, and screenshots are in `/tests/`.

---

## Design Tradeoffs

**Static routing vs. dynamic routing.** Static routes were chosen deliberately to maintain deterministic, auditable traffic paths during redundancy testing. Dynamic routing (OSPF) would reduce operational overhead at scale but would introduce convergence behavior that complicates failover validation. This is documented as a known limitation for environments that grow beyond two core switches.

**Default HSRP timers.** No timer tuning was applied. The 5-packet loss measured during HSRP failover reflects the real cost of default hello (3s) and hold (10s) timers. Tuning to subsecond values (e.g., 200ms/700ms) would reduce this significantly, but adds complexity and increases control-plane overhead. The tradeoff is documented, not hidden.

**Spanning Tree vs. MEC.** Multi-Chassis EtherChannel was not implemented. Rapid-PVST+ was optimized instead. This avoids the operational complexity of MEC while still providing fast convergence. The tradeoff is that blocking cross-links represent unused bandwidth under normal operation.

**Stateless ACLs.** The ACL-based east-west policy is fast and simple but requires explicit return-path rules. Any policy change must account for both directions. This is a known operational burden that would be addressed by moving east-west inspection to the firewall in a more complex deployment.

---

## Repository Structure

```
enterprise-campus-network/
├── README.md
|
├── topology/
│   ├── logical-topology.png
│   └── physical-topology.png
|
├── configs/
│   ├── core/
│   │   ├── core-1.txt
│   │   └── core-2.txt
│   ├── access/
│   │   ├── acc-1.txt
│   │   ├── acc-2.txt
│   │   └── acc-3.txt
│   ├── firewall/
│   │   ├── fg-primary.txt
│   │   └── fg-secondary.txt
│   └── wan-dmz/
│       ├── wan-sw.txt
│       └── dmz-sw.txt
|
├── docs/
│   ├── architecture.md
│   ├── design-decisions.md
│   ├── ip-addressing-plan.md
│   ├── firewall-policy-matrix.md
│   ├── acl-reference.md
│   └── threat-model.md
|
├── validation-tests/
│   ├── ha-redundancy-tests/         # simulated hsrp core and active firewall failover
│   ├── acl-segmentation-tests/      # validated acl segmentation
│   └── service availability-tests/  # DMZ web server and HR file server 
|
└── notes/
    ├── ha-logic-explained.md
    ├── troubleshooting-log.md
    └── lessons-learned.md
```

---

## Lessons Learned

**Segmentation without enforcement is cosmetic.** VLANs create broadcast isolation, not security. Without ACLs at the SVI and firewall zone policies, a single misrouted packet crosses any boundary. Every trust boundary in this network has at least two independent enforcement points.

**Redundancy must be tested under load, not assumed.** Configuring HSRP and LACP does not guarantee the failover works. The 5-packet HSRP loss and 1-packet FG HA loss were only measurable because traffic was running continuously during the test. Silent failures — where the mechanism activates but traffic still drops — are only visible with active monitoring.

**ACL direction determines what you actually enforce.** An outbound ACL on a VLAN SVI is almost useless because traffic may have already been switched. Inbound ACLs at the ingress SVI catch traffic before any routing decision, which is the correct enforcement point.

**Firewall NAT success does not validate internal routing.** On multiple occasions during build, internet connectivity appeared to work while internal routing was misconfigured. The NAT session table was masking the problem. Validating each layer independently — L2, L3, ACL, firewall — prevents this confusion.

**Simplicity improves auditability.** Every configuration decision in this network can be traced to a specific requirement. There are no "just in case" rules, no undocumented ACL entries, and no routing entries that exist for unclear reasons. An auditor or a new engineer can read the configuration and reconstruct the intent.
