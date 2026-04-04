# Design Decisions


## Architectural Model

### Decision: Collapsed Core / Distribution Layer

The network implements a collapsed core design where Layer 3 switching, inter-VLAN routing, and gateway redundancy are centralized at the Core layer.

**Rationale:**

* Suitable for campus-scale environments
* Reduces device count and complexity
* Simplifies routing domain boundaries
* Provides high performance through hardware-based L3 switching

This design balances operational simplicity with redundancy.

---

## Default Gateway Redundancy Strategy

### Decision: HSRP on Core L3 Switches

HSRP was implemented across CORE-1 and CORE-2 to provide a resilient default gateway for all VLANs.

* CORE-1 priority: 110
* CORE-2 priority: 100
* Preemption enabled on both devices

Each VLAN uses a Virtual IP (VIP) as the default gateway.

**Rationale:**

* Hosts remain unaware of physical gateway changes
* No DHCP reconfiguration required during failover
* Deterministic role recovery through preemption
* Simple and predictable convergence model

Alternative protocols such as VRRP were considered functionally equivalent; HSRP was chosen for Cisco-native integration within the lab environment.

---

## Inter-Core Connectivity

### Decision: LACP Port-Channel (CORE_INTERLINK)

Multiple physical links between CORE-1 and CORE-2 were bundled using LACP (802.3ad) into a logical Port-Channel.

**Rationale:**

* Increased aggregate bandwidth
* Link-level redundancy
* Faster failure detection at Layer 2
* Simplified STP topology

Without EtherChannel, STP would block redundant links, wasting available bandwidth.

---

## Access Layer Redundancy Model

### Decision: “X” Mesh Topology

Each Access switch connects to both Core switches, forming an “X” redundant uplink structure.

**Rationale:**

* Eliminates single uplink dependency
* Provides alternate forwarding path
* Increases physical path diversity

This design supports continued operation even if:

* One Core switch fails
* One uplink fails
* One Port-Channel member fails

---

## Loop Prevention Strategy

### Decision: Rapid-PVST+ with Deterministic Root Bridge Placement

To control redundant Layer 2 paths:

* CORE-1 configured as Primary Root Bridge (priority 4096)
* CORE-2 configured as Secondary Root Bridge (priority 8192)

Rapid-PVST+ selected for faster convergence compared to traditional STP.

**Rationale:**

* Deterministic traffic flow
* Predictable blocked uplinks
* Sub-second reconvergence during link failure
* No requirement for Multi-Chassis EtherChannel (MEC)

Secondary “X” links remain in Blocking state but transition to Forwarding if the active uplink fails.

---

## Firewall High Availability Strategy

### Decision: FortiGate Active-Passive (A-P) Cluster

The perimeter security layer uses two FortiGate-VM instances configured in HA:

* FG1: Priority 200, Override enabled
* FG2: Priority 100, Override disabled
* Session pickup enabled
* Dedicated heartbeat interface

**Rationale:**

* Deterministic primary role
* Automatic failover during hardware failure
* Session table synchronization to minimize disruption
* Simplified operational model compared to Active-Active

Active-Passive was selected over Active-Active to avoid asymmetric routing complexity and simplify troubleshooting.

---

## Link Monitoring & Failover Logic

### Decision: Monitored Interfaces in HA Configuration

Core-facing interfaces were included under:

* `set monitor` in HA config
* `config system link-monitor`

**Rationale:**

Prevents “half-alive” scenarios where:

* A firewall remains Primary
* But loses upstream connectivity to the Core layer

If a monitored interface fails, HA logic can trigger role reassignment.

This ensures failover decisions reflect actual forwarding capability, not just device uptime.

---

## Routing Strategy

### Decision: Static Routing with Backup Paths

Static routes to the internal network (`10.10.0.0/16`) were configured with:

* Primary path (lower priority)
* Backup path (higher priority value)

**Rationale:**

* Simple and predictable
* No routing protocol overhead
* Clear failover behavior

Dynamic routing (e.g., OSPF) was not implemented to keep control-plane complexity minimal in this lab iteration.

---

## VLAN Segmentation & Security Posture

### Decision: Functional VLAN Separation

Network segmented into:

* IT
* Management
* HR
* Guest

**Rationale:**

* Reduced broadcast domains
* Logical separation of trust boundaries
* Policy-based firewall enforcement per subnet
* Scalable access control expansion

Guest-to-internal restrictions enforced via firewall policies.

---

## Failback Behavior (Deterministic Recovery)

Both layers are configured for controlled role recovery:

* HSRP preemption ensures CORE-1 reclaims Active role upon recovery
* Firewall override ensures FG1 reclaims Primary role

**Rationale:**

Maintains consistent traffic patterns and predictable operational state after maintenance or transient failures.

---

## Trade-Offs & Limitations

* HSRP default timers introduce multi-second convergence
* Static routing reduces adaptability in larger environments
* Active-Passive firewall model does not provide load sharing
* No dynamic routing between Core and Firewall

These were acceptable trade-offs for clarity, determinism, and lab focus.

---

## Design Outcome

The final architecture achieves:

* No single point of failure at Core layer
* No single point of failure at Firewall layer
* Redundant physical paths
* Deterministic convergence behavior
* Minimal traffic interruption during failure events

The design prioritizes operational clarity and predictable behavior over complexity.
