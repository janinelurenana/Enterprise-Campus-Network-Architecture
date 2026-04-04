# Network IP Addressing & Connectivity Plan

## 1. Internal VLAN Architecture
This table defines the segmentation for the access layer. All subnets use a `/24` mask. Core-1 is the Primary Gateway (HSRP Priority 110), and Core-2 is the Standby.

| VLAN | Name | Subnet | Virtual GW (VIP) | Core-1 IP | Core-2 IP | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **10** | IT | 10.10.10.0/24 | 10.10.10.1 | 10.10.10.2 | 10.10.10.3 | Admin / Infra |
| **20** | Management | 10.10.20.0/24 | 10.10.20.1 | 10.10.20.2 | 10.10.20.3 | Exec / Ops |
| **30** | HR | 10.10.30.0/24 | 10.10.30.1 | 10.10.30.2 | 10.10.30.3 | Sensitive Data |
| **40** | Guests | 10.10.40.0/24 | 10.10.40.1 | 10.10.40.2 | 10.10.40.3 | Untrusted |

---

## 2. Core-to-Firewall L3 Links (Transit)
These are point-to-point `/30` routed links connecting the Core switches to the FortiGate HA Cluster.

| Link Path | Core Int | FG-1 Port | FG-2 Port | Core IP | FG IP | Subnet |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Core-1 ↔ FG-1** | G0/0 | Port 2 | - | 10.10.255.1 | 10.10.255.2 | 10.10.255.0/30 |
| **Core-2 ↔ FG-2** | G0/0 | - | Port 4 | 10.10.255.5 | 10.10.255.6 | 10.10.255.4/30 |
| **Core-1 ↔ FG-2** | G2/0 | - | Port 3 | 10.10.255.9 | 10.10.255.10 | 10.10.255.8/30 |
| **Core-2 ↔ FG-1** | G2/0 | Port 5 | - | 10.10.255.13 | 10.10.255.14 | 10.10.255.12/30 |

---

## 3. High Availability (HA) Configuration

### FortiGate HA (Active-Passive)
* **Cluster Name:** FG-CLUSTER
* **Group ID:** 10
* **Mode:** A-P (Active-Passive)
* **Heartbeat Interface:** Port 8 (Priority 50)
* **Monitored Interfaces:** Port 2, Port 3, Port 4, Port 5
* **Primary Priority:** 200 (FG-1)

### Gateway Redundancy (HSRP)
* **HSRP Group:** Matches VLAN ID.
* **Tracking:** * **Core-1:** Tracks IP SLA 10 (Target: FG-1). Decrement: 25.
    * **Core-2:** Tracks physical interface G0/0 and G2/0. Decrement: 20.

---

## 4. Layer 2 & STP Logic


* **STP Mode:** Rapid-PVST
* **Root Bridge:** **Core-1** (Priority 4096)
* **Secondary Root:** **Core-2** (Priority 8192)

| Access Switch | Core-1 Uplink | Core-2 Uplink | Allowed VLANs |
| :--- | :--- | :--- | :--- |
| **ACC-1** | G0/1 | G1/1 | 20, 40 |
| **ACC-2** | G0/2 | G1/2 | 30 |
| **ACC-3** | G0/3 | G1/3 | 10 |
| **Core Interlink** | Po1 (G1/0) | Po1 (G1/0) | ALL (Trunk) |

---

## 5. Security Policy Matrix (ACL Summary)

| Source | Allowed Destinations | Restricted Destinations |
| :--- | :--- | :--- |
| **IT (10)** | MGMT (20), HR (30), Internet | Guest (40) |
| **MGMT (20)** | HR (30), Internet | IT (10), Guest (40) |
| **HR (30)** | Internet | IT (10), MGMT (20), Guest (40) |
| **Guest (40)** | Internet | All Internal (10.10.0.0/16) |

> **Note:** Policies are enforced via Extended ACLs on the Core SVIs and via FortiGate Zone Policies for North-South traffic.