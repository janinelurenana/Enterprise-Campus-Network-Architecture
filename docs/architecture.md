# Network Architecture

## Purpose

This document describes the logical and functional architecture of the campus enterprise network. It covers how the network is structured, how traffic flows through it, what each layer is responsible for, and where enforcement boundaries exist. It is intended as the primary reference for understanding how the system works as a whole.

---

## Design Philosophy

The network is built around three principles:

**Separation of concerns.** Each layer has a defined role and does not perform functions belonging to another layer. Access switches do Layer 2 only. Core switches do routing and east-west policy enforcement. The firewall does perimeter inspection, NAT, and zone separation. No layer compensates for another's job.

**Defense in depth.** No security control exists in only one place. Guest isolation is enforced by both SVI ACLs and firewall zone policy. DMZ isolation is enforced by both physical topology (no core uplink) and firewall zone rules. A misconfiguration at one layer does not automatically open a path.

**Determinism over cleverness.** Traffic paths are predictable. Failover behavior is measurable. Every static route, every ACL entry, and every STP priority value was set with explicit intent. There are no "it should work out" assumptions in the design.

---

## Topology Overview

```
                        [ ISP / WAN ]
                              |
                        [ WAN-SW-1 ]
                        (VLAN 901)
                         /        \
                      [FG-1]----[FG-2]   FortiGate HA Cluster
                     (Active)  (Standby) Active-Passive, session sync
                         \        /
                    port2  \    / port4/port5
                            \  /
              ┌─────────[CORE-1]════════[CORE-2]─────────┐
              │          (Primary)  LACP  (Secondary)      │
              │           STP 4096        STP 8192         │
              │           HSRP 110        HSRP 100         │
              │                                            │
           Primary uplinks          Secondary (cross) uplinks
              │      │      │              │      │      │
           [ACC-1] [ACC-2] [ACC-3]      [ACC-1] [ACC-2] [ACC-3]
           VL20,40  VL30   VL10
              |       |      |
           Hosts    Hosts  Hosts

                        [ FG-1 or FG-2 ]
                              |
                         [ DMZ-SW ]
                         (VLAN 50)
                              |
                        [ Nginx Server ]
```

The ════ line between CORE-1 and CORE-2 represents the LACP Port-Channel interlink. Each access switch has two physical uplinks — one to each core — forming the X-mesh. Under normal operation, the CORE-1 uplinks forward and the CORE-2 cross-links are in STP blocking state.

---

## Layer Responsibilities

### Access Layer

ACC-1, ACC-2, and ACC-3 are pure Layer 2 devices. Their responsibilities are:

- Terminate end-host connections on access ports
- Carry VLAN traffic to the core over 802.1Q trunks
- Limit trunk scope to only the VLANs present on each switch (pruning)
- Enable PortFast on host-facing ports to eliminate STP delay
- Provide two physical uplinks per switch for path redundancy

Access switches do not perform routing, do not enforce access policy, and do not participate in inter-VLAN decisions. Keeping them policy-free reduces configuration drift and makes the access layer auditable at a glance.

**VLAN assignments per switch:**

| Switch | VLANs  | Connected Departments |
|--------|--------|----------------------|
| ACC-1  | 20, 40 | Management, Guest    |
| ACC-2  | 30     | HR                   |
| ACC-3  | 10     | IT                   |

### Core Layer

CORE-1 and CORE-2 are the routing and enforcement layer. Their responsibilities are:

- Host SVIs for all four internal VLANs (10, 20, 30, 40)
- Provide default gateway addresses via HSRP virtual IPs
- Enforce east-west access policy via inbound ACLs on each SVI
- Route traffic between VLANs (where policy permits) and toward the firewall
- Maintain redundant routing paths to the FortiGate cluster
- Run Rapid-PVST+ to manage loop prevention across the X-mesh topology

CORE-1 is the primary device: STP root (priority 4096), HSRP active (priority 110), and the preferred next-hop for default routes via IP SLA tracking. CORE-2 is the secondary: STP secondary root (priority 8192), HSRP standby (priority 100), and active only when CORE-1 is unavailable or its upstream path is down.

The LACP Port-Channel between the two cores carries inter-core traffic and serves as the backbone for VLANs whose primary uplink runs through one core while the HSRP active gateway for that VLAN is on the other.

### Perimeter Layer (FortiGate HA Cluster)

The FortiGate cluster is the north-south enforcement boundary. Its responsibilities are:

- Perform stateful inspection on all traffic entering or leaving the network
- Apply NAT to outbound traffic from internal VLANs
- Enforce zone-based policy (WAN, ZONE_INTERNAL, ZONE_DMZ)
- Provide internet access to internal VLANs via explicit service policy
- Expose the DMZ web server to WAN via a restricted inbound policy
- Maintain redundant routed adjacencies to both core switches

FG-1 is the active unit (HA priority 200). FG-2 is the standby (HA priority 100). Configuration and session state are continuously synchronized. On failover, FG-2 assumes the active role within approximately one ICMP round-trip of disruption. The shared virtual MAC prevents ARP disruption on connected switches.

### WAN and DMZ Segments

**WAN-SW-1** connects both FortiGate WAN ports and the ISP uplink to a single broadcast domain (VLAN 901). This allows both FG units to share the ISP connection without requiring ISP-side reconfiguration during HA failover.

**DMZ-SW** connects both FortiGate DMZ-facing ports and the Nginx web server. It has no uplink to the core switching fabric — the only routed path into or out of the DMZ is through the FortiGate. This physical isolation is intentional and provides a layer of isolation independent of firewall policy.

---

## Traffic Flow

### Outbound (Internal Host → Internet)

1. Host sends traffic to its VLAN's virtual IP (HSRP VIP, e.g., 10.10.20.1 for Management).
2. The HSRP-active core switch receives the packet on the SVI. The inbound ACL is evaluated — if the destination is permitted (internet-bound traffic passes for all VLANs), routing proceeds.
3. The core routes the packet toward the FortiGate via the static default route (tracked via IP SLA).
4. The FortiGate matches the packet against policy (INTERNAL_to_WAN: permit HTTP/HTTPS/DNS/PING), applies source NAT, records the session, and forwards out port1 to the ISP.

### Inbound (Internet → Internal Host, Return Traffic)

1. The ISP delivers the return packet to the FortiGate's WAN IP.
2. The FortiGate consults its session table, reverses the NAT translation, and routes the packet toward the internal destination via the static 10.10.0.0/16 route.
3. The packet arrives at the core switch's routed interface. The core routes to the correct SVI and delivers to the host.
4. The SVI ACL permits the return packet via the `permit tcp any any established` and `permit icmp any any echo-reply` rules at the top of each ACL.

### East-West (Internal VLAN to Internal VLAN)

1. Host sends traffic to a destination in another VLAN. The packet arrives at the core SVI for the source VLAN.
2. The inbound ACL on the source SVI is evaluated first. If denied (e.g., HR → IT), the packet is dropped and logged here. Traffic never reaches a routing decision.
3. If permitted (e.g., IT → HR), the core routes the packet to the destination SVI and delivers to the host. The firewall is not involved in this path.

### Inbound (WAN → DMZ Web Server)

1. An external client connects to the FortiGate's WAN IP on port 80 or 443.
2. The WAN_to_DMZ firewall policy matches (source: ZONE_WAN, destination: DMZ_SUBNET, service: HTTP/HTTPS). The session is recorded and the packet is forwarded out the DMZ-facing interface.
3. DMZ-SW delivers the packet to the Nginx server on VLAN 50.
4. Return traffic follows the reverse path through the active FortiGate session.

### DMZ → Internal (Blocked)

If the DMZ server attempts to reach an internal VLAN:
1. Traffic arrives at the FortiGate from the DMZ-facing interface.
2. The DMZ_to_INTERNAL_DENY policy matches before any permit rule. The packet is dropped and logged.
3. Even if this policy were absent, there is no Layer 2 path from DMZ-SW to the core — the traffic would never reach the core switches.

---

## Redundancy Architecture

### Gateway Redundancy (HSRP)

One HSRP group per VLAN. The virtual IP (.1 address) is the configured default gateway on all hosts. CORE-1 holds active for all groups under normal conditions (priority 110 vs. CORE-2's 100). If CORE-1's upstream gateway (the FortiGate) becomes unreachable, IP SLA tracking decrements CORE-1's HSRP priority by 25, dropping it below CORE-2's 100 and triggering failover. If CORE-1 itself goes down, CORE-2 assumes active immediately via preempt.

The 60-second preempt delay on CORE-1 prevents it from reclaiming active immediately after a restart — long enough for routing tables and STP to fully converge before HSRP traffic shifts back.

### Path Redundancy (IP SLA + Static Routes)

Both core switches maintain tracked and untracked static default routes to the FortiGate. The tracked route is preferred (lower administrative distance). If the IP SLA probe fails — indicating the FortiGate gateway is unreachable — the tracked route is withdrawn and traffic shifts to the next available path. Multiple /30 host routes to each transit link ensure that path diversity exists even when the summary route might point to a failed next-hop.

### Link Redundancy (LACP + X-Mesh + Rapid-PVST+)

The LACP Port-Channel between cores provides redundancy on the inter-core backbone. If a single physical member link fails, traffic continues across remaining members without routing reconvergence.

The X-mesh design gives each access switch two uplinks — one to each core. Rapid-PVST+ keeps cross-links in blocking state to prevent loops. On primary uplink failure, the cross-link transitions to forwarding in sub-second time (Rapid-PVST+ uses proposal/agreement rather than the 30-second STP timer sequence).

### Firewall Redundancy (FortiGate Active-Passive HA)

FG-1 and FG-2 share a dedicated heartbeat link (port8). FG-1 monitors four internal-facing interfaces; failure of any monitored interface triggers evaluation of whether FG-2 is in a better state. If FG-2 has all monitored interfaces up, it assumes the active role. Session synchronization means in-progress TCP connections are not reset during failover. The shared virtual MAC prevents ARP cache updates on connected switches, eliminating the ARP-flush delay that would otherwise add to failover time.

---

## Enforcement Boundaries Summary

| Boundary              | Mechanism                          | Layer  | Stateful |
|-----------------------|------------------------------------|--------|----------|
| VLAN isolation        | 802.1Q, trunk pruning              | L2     | No       |
| East-west access      | SVI inbound ACLs (core switches)   | L3     | No       |
| Guest → internal deny | ACL + FortiGate zone policy        | L3/L4  | No / Yes |
| North-south inspection| FortiGate stateful firewall        | L3–L7  | Yes      |
| DMZ → internal deny   | FortiGate zone policy + physical   | L2/L3  | No       |
| WAN → internal deny   | FortiGate implicit deny            | L3–L7  | Yes      |
| Outbound NAT          | FortiGate SNAT                     | L3     | Yes      |

---

## Known Architectural Constraints

**Static routing.** The absence of a dynamic routing protocol means topology changes require manual route updates. IP SLA provides basic tracking but is not a substitute for protocol-driven convergence. OSPF would be the appropriate next step for a network that grows beyond two core switches or adds a second site.

**Stateless east-west ACLs.** Core ACLs are stateless. Return traffic is explicitly permitted via established/echo-reply rules rather than tracked session state. This is operationally simple but introduces a theoretical gap around crafted TCP flag abuse. Moving east-west inspection to the firewall resolves this at the cost of additional firewall load and routing complexity.

**Default HSRP timers.** Five ICMP packets are lost during CORE-1 failover under default timers. This is acceptable for most workloads but would be a problem for latency-sensitive applications. BFD-assisted HSRP with subsecond timers is the standard mitigation.

**Single ISP.** The current design has one ISP uplink through WAN-SW-1. Dual-ISP with policy-based routing or BGP is the next logical redundancy increment for the WAN edge.
