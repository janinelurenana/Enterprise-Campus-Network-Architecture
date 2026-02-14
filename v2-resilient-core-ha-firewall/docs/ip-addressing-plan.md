# IP Addressing & Network Documentation Plan

## VLAN & Subnet Architecture

This table defines the internal segmentation for the user access layers. All subnets use a `/24` mask to allow for future growth and simplified management.

| VLAN | Name | Subnet | HSRP Virtual GW (VIP) | Core-1 IP | Core-2 IP | Purpose |
| --- | --- | --- | --- | --- | --- | --- |
| **10** | IT | 10.10.10.0/24 | 10.10.10.1 | 10.10.10.2 | 10.10.10.3 | Admin / Infra |
| **20** | Management | 10.10.20.0/24 | 10.10.20.1 | 10.10.20.2 | 10.10.20.3 | Exec / Ops |
| **30** | HR | 10.10.30.0/24 | 10.10.30.1 | 10.10.30.2 | 10.10.30.3 | Sensitive Data |
| **40** | Guests | 10.10.40.0/24 | 10.10.40.1 | 10.10.40.2 | 10.10.40.3 | Untrusted |

---

## Core to Firewall (L3 PtP Links)

These are point-to-point `/30` routed links between the Core switches and the FortiGate High Availability cluster.

| Link Description | Core Interface | Core IP | FG IP | Network (/30) |
| --- | --- | --- | --- | --- |
| **Core-1 → FG-1** | G0/0 | 10.10.255.1 | 10.10.255.2 | 10.10.255.0/30 |
| **Core-2 → FG-2** | G0/0 | 10.10.255.5 | 10.10.255.6 | 10.10.255.4/30 |
| **Core-1 → FG-2** | G2/0 | 10.10.255.9 | 10.10.255.10 | 10.10.255.8/30 |
| **Core-2 → FG-1** | G2/0 | 10.10.255.13 | 10.10.255.14 | 10.10.255.12/30 |

---

## Redundancy & High Availability (HSRP)

* **HSRP Group Naming:** Group numbers match the VLAN ID (e.g., VLAN 10 uses Group 10).
* **Priority:** * **CORE-1:** Primary (Priority 110).
* **CORE-2:** Standby (Priority 100/Default).


* **Tracking:** Both cores use IP SLA tracking on their uplinks to the FortiGates. If the upstream link fails, the HSRP priority decrements to allow the other core to take over.

---

## Layer 2 Trunking / Access

| Downstream Switch  | Physical Uplink          | Allowed VLANs |
| ------------------ | ------------------------ | ------------- |
| **ACC-1**          | Core-1 G0/1, Core-2 G1/1 | 20, 40        |
| **ACC-2**          | Core-1 G0/2, Core-2 G1/2 | 30            |
| **ACC-3**          | Core-1 G0/3, Core-2 G1/3 | 10            |
| **Core Interlink** | Port-channel 1           | ALL (1–4094)  |


---

## Security Policy Matrix (Summary)

| Source VLAN | Destination | Policy |
| --- | --- | --- |
| **IT (10)** | MGMT, HR, WAN | **Permit** |
| **IT (10)** | GUEST | **Deny** |
| **MGMT (20)** | HR, WAN | **Permit** |
| **HR (30)** | WAN | **Permit** (Isolated from other VLANs) |
| **GUEST (40)** | WAN | **Permit** (Isolated from all internal 10.10.0.0/16) |