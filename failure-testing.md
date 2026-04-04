# Failure Testing & Validation

## Methodology

All tests were conducted under continuous ICMP traffic (1-second intervals) between an internal host and an external destination. No protocol timers were tuned prior to testing — results reflect baseline convergence behavior. Each test was repeated to confirm reproducibility.

Tests are organized by failure type, not by severity. The goal is to verify that each redundancy mechanism activates correctly and independently, and that no test reveals an unexpected dependency between mechanisms.

---

## Pre-Test Baseline

Before any failure testing, the following baseline connectivity was confirmed:

| Test                                    | Expected | Result |
|-----------------------------------------|----------|--------|
| IT host → internet                      | Pass     | ✅      |
| HR host → IT subnet                     | Deny     | ✅      |
| Guest host → internal subnets           | Deny     | ✅      |
| WAN → DMZ web server (HTTP)             | Pass     | ✅      |
| WAN → internal subnets                  | Deny     | ✅      |
| DMZ server → internal subnets           | Deny     | ✅      |
| IT host → Management subnet             | Pass     | ✅      |
| Management host → IT subnet             | Deny     | ✅      |

Baseline evidence (show commands, pings, ACL hit counters) is in `/tests/connectivity-baseline/`.

---

## Test 1 – Core Primary Failure (HSRP)

**Objective:** Verify that CORE-2 assumes the active HSRP role and traffic resumes without manual intervention when CORE-1 is shut down.

**Procedure:**
1. Confirm CORE-1 is HSRP active for all VLANs
2. Start continuous ICMP from an IT host to an external IP
3. Shut down CORE-1 (`shutdown` on all active interfaces)
4. Monitor ICMP output for loss duration
5. Confirm CORE-2 transitions to HSRP active
6. Restore CORE-1 and verify it preempts after the configured delay

**Observed Results:**
- 5 ICMP packets lost during failover
- CORE-2 transitioned to HSRP active successfully for all VLANs
- Traffic resumed automatically — no manual intervention required
- CORE-1 preempted correctly after the 60-second delay on restoration
- STP root role remained on CORE-1 (re-elected on restoration)

**Root cause of 5-packet loss:** Default HSRP hold timer (10 seconds) + hello interval (3 seconds). CORE-2 must wait for the hold timer to expire before declaring CORE-1 failed. This is expected and documented behavior.

**Evidence:** `/tests/test-1-hsrp-core-failover/`

---

## Test 2 – Active Firewall Failure (FortiGate HA)

**Objective:** Verify that the secondary FortiGate assumes the active role and in-progress sessions are maintained when the primary unit fails.

**Procedure:**
1. Confirm FG-primary (FGVMEV_JBNLZY4B0) is HA active
2. Start continuous ICMP from an internal host to an external IP
3. Power off the primary FortiGate VM
4. Monitor ICMP output for loss duration
5. Confirm the secondary unit (FGVMEVNLIKHEMG61) transitions to active
6. Confirm session state was preserved (no re-establishment required)

**Observed Results:**
- 1 ICMP packet lost during failover
- Secondary unit assumed active role
- Session state preserved (session pickup enabled)
- Core routing table unchanged — failover was transparent to the switching layer
- Virtual MAC preserved — no ARP disruption observed

**Note:** The 1-packet loss reflects the time required for the secondary to detect primary failure via heartbeat and assume the active role. This is significantly faster than HSRP failover because the FG HA subsystem uses a dedicated heartbeat (port8) with a configurable lost-threshold (set to 8 missed heartbeats in this config).

**Evidence:** `/tests/test-2-firewall-ha-failover/`

---

## Test 3 – Access Uplink Failure (Rapid-PVST+)

**Objective:** Verify that when a primary uplink from an access switch to CORE-1 fails, the blocked cross-link to CORE-2 transitions to forwarding without sustained traffic loss.

**Procedure:**
1. Confirm primary uplinks are forwarding; cross-links are blocking (via `show spanning-tree`)
2. Start continuous ICMP from a host on the affected access switch
3. Shut down the primary uplink interface on the access switch
4. Monitor ICMP for loss duration
5. Confirm cross-link transitions from blocking to forwarding
6. Restore the primary link and confirm topology returns to baseline

**Observed Results:**
- Sub-second reconvergence (fewer than 3 ICMP packets lost)
- Cross-link transitioned from blocking to forwarding
- Rapid-PVST+ PortFast/Edge configuration on host ports prevented unnecessary TCN flooding
- Primary link restoration triggered expected topology recalculation; cross-link returned to blocking

**Evidence:** `/tests/test-3-stp-uplink-failover/`

---

## Test 4 – Dual Failure (CORE-1 + Active FortiGate)

**Objective:** Verify that simultaneous failure of CORE-1 and the active FortiGate does not produce a cascading outage — that HSRP and FG HA operate independently and both recover without manual intervention.

**Procedure:**
1. Confirm CORE-1 is HSRP active; FG-primary is HA active
2. Start continuous ICMP from an internal host to an external IP
3. Simultaneously shut down CORE-1 and power off the active FortiGate
4. Monitor ICMP for total loss duration and recovery
5. Confirm CORE-2 is HSRP active AND secondary FortiGate is HA active

**Observed Results:**
- Combined loss was bounded by the slower of the two mechanisms (HSRP, ~5 packets)
- Both mechanisms recovered independently — no deadlock or dependency observed
- Traffic resumed without manual intervention
- CORE-2 routing table remained intact; cross-links to both cores were available

**Key validation:** The IP SLA on CORE-1 monitors the FortiGate gateway. On CORE-1 shutdown, the SLA tracking object was irrelevant — HSRP failover occurred due to interface failure, not gateway reachability. On CORE-2, the IP SLA continued to track the new active FortiGate address without interruption.

**Evidence:** `/tests/test-4-dual-failure/`

---

## ACL Validation

ACL enforcement was validated by testing each deny rule explicitly:

| Traffic Scenario                           | ACL Applied        | Expected | Result |
|--------------------------------------------|--------------------|----------|--------|
| Guest (40) → IT subnet (10)                | ACL_GUEST_IN       | Deny     | ✅      |
| Guest (40) → HR subnet (30)                | ACL_GUEST_IN       | Deny     | ✅      |
| HR (30) → IT subnet (10)                   | ACL_HR_IN          | Deny     | ✅      |
| HR (30) → Management subnet (20)           | ACL_HR_IN          | Deny     | ✅      |
| Management (20) → IT subnet (10)           | ACL_MGMT_IN        | Deny     | ✅      |
| IT (10) → Guest subnet (40)                | ACL_IT_IN          | Deny     | ✅      |
| IT (10) → HR subnet (30)                   | ACL_IT_IN          | Permit   | ✅      |
| IT (10) → Management subnet (20)           | ACL_IT_IN          | Permit   | ✅      |
| Internal host → internet (HTTP)            | FW policy          | Permit   | ✅      |
| Return traffic (established TCP)           | ACL (all VLANs)    | Permit   | ✅      |

ACL hit counters (`show ip access-lists`) were inspected after each test to confirm the correct rule matched.

---

## Observations

The dual-failure test was the most informative. The expectation was that two simultaneous failures would produce additive complexity. The actual result was that they were genuinely independent — CORE-2's HSRP and IP SLA did not depend on the primary FortiGate being reachable, and the secondary FortiGate did not depend on CORE-1 being active. This validates the design goal of layered redundancy with no shared state between mechanisms.

The 5-packet HSRP loss is the most visible limitation of this design. It is a direct consequence of default timers and is not hidden or minimized in this documentation. Any deployment with latency-sensitive workloads should evaluate BFD-assisted HSRP or subsecond timer tuning before production use.
