# Campus Network with VLAN Segmentation and Firewall Policy

## Executive Summary

This lab implements a **single-building enterprise campus network** with:

* Layer 2 VLAN segmentation
* Centralized Layer 3 inter-VLAN routing
* East–west access control via ACLs
* North–south stateful inspection via FortiGate

The design enforces **least privilege, clear role separation, and predictable traffic flow**.

This is a foundational architecture intended to scale into a high-availability design in V2.

---

## Architectural Goals

* Enforce least privilege between departments
* Prevent lateral movement at the distribution layer
* Centralize north–south inspection
* Keep access layer free of routing or policy logic
* Maintain deterministic traffic paths

---

## Assumptions

* One VLAN per department
* /24 subnet per VLAN for simplicity and readability
* Core switch owns all SVIs and default gateways
* Firewall handles NAT and Internet access only
* Guest traffic must never access internal VLANs

---

## Topology

![Network Topology](./topology/topology.png)

The core switch acts as the routing and internal policy enforcement point.
The firewall enforces perimeter security and NAT.
Access switches remain strictly Layer 2 devices.

---

## VLAN & IP Addressing Plan

### Addressing Philosophy

* One VLAN = one subnet
* Gateways use `.1` consistently
* Transit links use /30 subnets
* No overlapping or ambiguous ranges

### VLAN Table

| VLAN | Name       | Subnet        | Gateway    | Purpose          |
| ---: | ---------- | ------------- | ---------- | ---------------- |
|   10 | IT         | 10.10.10.0/24 | 10.10.10.1 | Admin / Infra    |
|   20 | Management | 10.10.20.0/24 | 10.10.20.1 | Exec / Ops       |
|   30 | HR         | 10.10.30.0/24 | 10.10.30.1 | Sensitive Data   |
|   40 | Guest      | 10.10.40.0/24 | 10.10.40.1 | Untrusted Access |

Addressing follows a predictable schema (10.10.<VLAN>.0/24) to improve operational clarity and troubleshooting speed.

### Core ↔ Firewall Transit

| Link | Subnet         | Core IP     | Firewall IP |
| ---: | -------------- | ----------- | ----------- |
|  P2P | 10.10.255.0/30 | 10.10.255.1 | 10.10.255.2 |

---

## Layer 2 Design (Access Layer)

**Trunk VLAN assignments:**

| Switch | Allowed VLANs |
| -----: | ------------- |
|    SW1 | 20, 40        |
|    SW2 | 30            |
|    SW3 | 10            |

* VLAN pruning reduces unnecessary broadcast propagation

* Access layer remains policy-free to avoid configuration drift

* Trunks carry only required VLANs

---

## Core Layer 3 Responsibilities

The core switch is the routing brain of the network:

* Hosts all SVIs
* Performs inter-VLAN routing
* Enforces east–west security via inbound ACLs

### Why ACLs on SVIs?

* Wire-speed enforcement
* Simple, explicit policy control
* Prevents lateral movement before traffic ever reaches the firewall

ACLs are **stateless**, so return traffic is explicitly permitted where required. Stateful inspection is intentionally deferred to the firewall.

---

## Inter-VLAN Access Policy (East–West)

| Source \ Destination | IT | Management | HR | Internet |
| -------------------- | -- | ---------- | -- | -------- |
| IT                   | —  | ✅          | ✅  | ✅        |
| Management           | ❌  | —          | ✅  | ✅        |
| HR                   | ❌  | ❌          | —  | ✅        |
| Guest                | ❌  | ❌          | ❌  | ✅        |

Default stance is **deny**. Inter-department communication requires explicit justification.

---

## Stealth Filtering (RFC 1812 Compliance)

ICMP unreachables are disabled on internal SVIs to reduce reconnaissance feedback from untrusted VLANs.

* Implementation: The no ip unreachables command is applied to all internal SVIs, specifically VLAN 40 (Guest).

* Security Outcome: Attempts to probe internal subnets from the Guest VLAN result in a silent "Request Timed Out" rather than an explicit rejection. This forces an attacker to guess whether a host is offline or protected by a firewall, significantly slowing down the reconnaissance phase.

---

## Firewall Design (North–South)

### Interface Roles

* `port1` → WAN / Cloud
* `port2` → Internal (Core L3 switch)

### Perimeter Security Model

* Firewall performs NAT
* Firewall performs stateful inspection
* Firewall does **NOT** perform inter-VLAN routing

---

## Routing Plan

### Core Switch

* Default route → FortiGate

```
0.0.0.0/0 → 10.10.255.2
```

### FortiGate

* Static route back to internal networks

```
10.10.0.0/16 → 10.10.255.1
```

Dynamic routing is intentionally excluded to preserve architectural simplicity at this stage.

---

## DHCP Strategy
The network utilizes a hybrid DHCP approach to balance administrative control with security isolation:

* VLANs 10/20/30 (Internal): Primarily managed via Static IP assignments for infrastructure and known endpoints to ensure auditability. Optional DHCP scopes are configured on the Core L3 Switch for temporary devices, with exclusions reserved for management interfaces
* VLAN 40 (Guest): DHCP is provided exclusively by the FortiGate Firewall. This ensures that untrusted "plug-and-play" devices are onboarded directly at the security boundary, facilitating easier logging and immediate stateful inspection.

**Design Justification:** By delegating the Guest scope to the firewall and keeping internal scopes on the core switch, we maintain a clear "separation of concerns". The Core switch focuses on high-speed internal distribution, while the FortiGate handles the overhead of dynamic, untrusted endpoint management.

---
## Packet Flow Validation

### Egress (Outbound):

* **VPC → Core:** Forwards traffic to the SVI (Default Gateway).

* **Core → Firewall:** Routes unknown destinations via the Gateway of Last Resort (0.0.0.0/0).

* **Firewall → Internet:** Matches security policy, applies Source NAT, and records the session.

### Ingress (Return):

* **Internet → Firewall:** Delivers response to the Public WAN IP.

* **Firewall → Core:** Consults the Session Table to "Un-NAT" the packet, then uses a Static Route (10.10.0.0/16) to find the Core Switch.

* **Core → VPC:** Delivers the packet to the specific VLAN host.

---

## Validation & Proof

### Validation confirms:

* VLAN membership and trunk integrity
* Correct SVI operation
* ACL enforcement under permitted and denied scenarios
* Successful NAT and stateful inspection
* Guest VLAN isolation

All validation artifacts are available under [/verification](./verification)

---

## Design Tradeoffs

* Single core switch represents a single point of failure (addressed in V2)

* Stateless ACLs require explicit return-path allowances

* Static routing limits scalability

---

## Repository Structure

```
v1-segmentation-fabric-vlan-acl-firewall/
|
├── topology/
│   └── topology.png
├── configs/
│   ├── core/
│   │   ├── core-l3.txt
│   │   └── core-acls.txt
│   ├── access/
│   │   ├── access-sw1.txt
│   │   ├── access-sw2.txt
│   │   └── access-sw3.txt
│   └── firewall/
│       └── fortigate.txt
├── verification/
│   ├── l2/
│   │   ├── show-vlan-brief.png
│   │   └── show-interfaces-trunk.png
│   ├── l3/
│   │   ├── show-ip-route.png
│   │   └── acl-core-show-access-lists.png
│   ├── firewall/
│   │   ├── fw-policy-table.png
│   │   └── fw-routing-table.png
│   └── e2e/
│       ├── internal-to-internet-ping.png
│       └── guest-to-internal-deny.png
├── docs/
│   ├── firewall-policy.md
│   ├── troubleshooting.md
│   └── lessons-learned.md
└── README.md
```

---

## Lessons Learned

* Segmentation without enforcement is cosmetic
* ACL direction and order determine effective security
* Firewall NAT success does not validate internal routing
* Simplicity improves auditability

---

## Architectural Evolution (See v2)

This design evolves in V2 to include:

* Dual core redundancy (HSRP)
* High-availability firewall cluster
* Failure testing validation
* Path resiliency

