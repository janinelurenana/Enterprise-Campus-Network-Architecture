This documentation provides a comprehensive analysis of the high availability (HA) convergence performance within the core switching layer, specifically focusing on the HSRP failover mechanism during a simulated hardware failure.

---

## Test Report: Core Switch HSRP Failover Performance

### 1. Goal

The objective of this test is to validate that the HSRP configuration across the Core switches successfully preserves default gateway availability and network traffic when the **Active** Core switch suffers a catastrophic failure.

### 2. Baseline Environment (Before Failure)

Prior to the failure event, the network was operating in a stable state with **CORE-1** acting as the primary gateway for all internal VLANs.

* **HSRP Roles:**
* **Active:** CORE-1 (Priority 110, Preemption Enabled).
* **Standby:** CORE-2 (Priority 100, Preemption Enabled).


* **Virtual IP (VIP):** Configured as `10.10.X.1` across VLANs 10, 20, 30, and 40.
* **Infrastructure:** EtherChannel (Po1) using LACP was operational, and the routing table on CORE-1 was fully populated with a Gateway of Last Resort via `10.10.255.2`.

**Reference Images:**

> **Figure 1:** `01-core1-hsrp-brief-before.png` — Initial HSRP brief showing CORE-1 as Active.
> **Figure 2:** `02-core2-hsrp-detail-before.png` — CORE-2 detail confirming it is in Standby mode.
> **Figure 3:** `03-core1-routing-before.png` — Baseline routing table on the Active Core.
> **Figure 4:** `05-ping-baseline-running.png` — Continuous ping from PC10A showing 0% packet loss.

---

### 3. Failure Execution

**Action:** A simulated hardware failure/link loss was performed on **CORE-1**.

#### **During Failover Observation**

As CORE-1 went offline, CORE-2 stopped receiving HSRP hello packets. After the hold timer expired, CORE-2 transitioned to the Primary role.

* **Ping Continuity:** A continuous ping from **PC10A** to `8.8.8.8` was monitored. Traffic was interrupted for a brief window as the standby router moved to the active state.
* **Recovery:** Traffic resumed automatically after 5 packets were lost (`icmp_seq=6` through `icmp_seq=10`).

**Reference Image:**

> **Figure 5:** `10-ping-during-failover.png` — ICMP traffic showing the 5-packet interruption during failover.

---

### 4. Post-Failover Analysis

The surviving node, **CORE-2**, successfully assumed the role of **Active**.

#### **HA State**

The HSRP status now reflects CORE-2 as the local Active node.

* **Active Router:** `local` (CORE-2).
* **Standby Router:** `10.10.X.2` (CORE-1 is now seen as the peer standby, or "unknown" if completely powered down).

#### **Layer 2 & ARP Convergence**

* **ARP Ownership:** CORE-2 began responding to requests for the Virtual MAC addresses (`0000.0c07.acXX`).
* **MAC Table Update:** Downstream switches updated their MAC address tables to point traffic toward the Port-Channel (Po1) or physical interfaces leading to CORE-2.

**Reference Images:**

> **Figure 6:** `07-core2-hsrp-brief-after.png` — CORE-2 confirming its transition to Active state.
> **Figure 7:** `08-core2-arp-after.png` — ARP table showing CORE-2 maintaining the Virtual IP-to-MAC mapping.
> **Figure 8:** `09-core2-mactable-after.png` — MAC address table on CORE-2 showing host reachability via Port-Channel 1.

---

### 5. Test Results Summary

| Metric | Result |
| --- | --- |
| **Packets Lost** | 5 packets |
| **Failover Time** | ~15 seconds (aligned with HSRP default hold timers) |
| **Gateway Reachability** | Successful; VIP `10.10.10.1` remained reachable post-failover |
| **New Active Confirmation** | CORE-2 assumed the Active role for all VLANs |
| **User Impact** | Minimal; momentary pause in traffic with automatic recovery |

### Conclusion

The HSRP implementation performed as expected. The transition from CORE-1 to CORE-2 was successful, ensuring that the internal hosts maintained their path to the Internet and Firewall. While HSRP default timers result in a slightly longer convergence time (~15s) compared to specialized HA protocols, the failover was reliable and required no manual intervention.
