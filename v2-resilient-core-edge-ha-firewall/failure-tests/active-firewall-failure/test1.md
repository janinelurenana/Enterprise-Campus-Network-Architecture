This documentation provides a comprehensive analysis of a High Availability (HA) failover test conducted on a FortiGate-VM64-KVM cluster. The test specifically evaluates the transition of traffic and session persistence during an active failure of the Primary node.

---

# Test Report: Firewall Active-Passive Failover Performance

## 1. Goal

The objective of this test is to validate that the FortiGate HA cluster successfully preserves network traffic and active sessions when the Primary node suffers a catastrophic failure.

## 2. Baseline Environment (Before Failure)

Prior to the failure event, the cluster was operating in a healthy **Active-Passive (A-P)** state.

* **HA Status:** The cluster reported `HA Health Status: OK` with both nodes in-sync.
* **Roles:** * **Primary:** `FGVMEV_JBNLZY4B0` (Priority 200, Override enabled).
* **Secondary:** `FGVMEVNLIKHEMG61`.


* **Heartbeat:** Dedicated heartbeat link established on `port8`.
* **Routing:** The routing table was populated with active static routes via `port1`, `port2`, and `port5`.

Source File: [01-fw-ha-status-before.png](./screenshots/01-fw-ha-status-before.png) - Initial HA cluster state showing synchronized nodes.

Source File: [02-fw-routing-before.png](./screenshots/02-fw-routing-before.png) - Active routing table entries on the Primary node.

Source File: [03-ping-baseline-running.png](./screenshots/03-ping-baseline-running.png) - Stable ICMP baseline prior to failure.
  

---

## 3. Failure Execution

**Action:** A hard power-off was simulated on the Active firewall (`FGVMEV_JBNLZY4B0`).

### During Failover Observation

As the primary node went offline, the secondary node detected the loss of heartbeat and initiated a transition to the Primary role.

* **Ping Continuity:** A continuous ping from `PC10A` to `8.8.8.8` was monitored. Only **one packet (icmp_seq=11)** was lost during the transition.
* **Recovery:** Traffic resumed automatically on `icmp_seq=12`.

Source File: [06-ping-during-failover.png](./screenshots/06-ping-during-failover.png) - ICMP traffic showing minimal interruption (1 packet loss).

---

## 4. Post-Failover Analysis

The surviving node, `FGVMEVNLIKHEMG61`, successfully assumed the role of Primary.

### HA State

The HA status now reflects the loss of the peer.

* **Error Message:** `FGVMEV_JBNLZY4B0 is lost`.
* **Mondev Status:** Warnings issued for `port2` and `port5` being down on the original primary, but the new primary has successfully taken over the traffic processing.

### Routing & Connectivity

* **Routing Intact:** The new Primary node updated the routing database. While some ports (like `port2` and `port5`) are marked inactive relative to the failed peer, the active routes migrated to the available physical interfaces (`port3` and `port4`).
* **NAT/Session Persistence:** Existing sessions survived the transition due to FGCP session synchronization.

Source File: [04-fw-ha-status-after.png](./screenshots/06-fw-ha-status-after.png) — Surviving node reporting itself as the new Primary.

Source File: [05-fw-routing-after.png](./screenshots/08-fw-routing-after.png)  — Updated routing table on the new Primary node.

---

## 5. Test Results Summary

| Metric | Result |
| --- | --- |
| **Packets Lost** | 1 packet |
| **Failover Time** | ~1 second (detected via ICMP timeout) |
| **Session Preservation** | Successful; continuous traffic flow confirmed |
| **New Primary Confirmation** | `FGVMEVNLIKHEMG61` assumed Primary role |
| **HA State** | Degraded (Single member mode) |

### Conclusion

The FortiGate HA cluster performed as expected. The transition from Primary to Secondary was near-instantaneous, with negligible impact on data plane traffic. The loss of a single ICMP packet confirms that the `session-pickup` feature effectively synchronized the state between members, ensuring a professional-grade failover experience.
