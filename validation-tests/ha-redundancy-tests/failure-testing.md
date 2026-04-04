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

## Observations

The dual-failure test was the most informative. The expectation was that two simultaneous failures would produce additive complexity. The actual result was that they were genuinely independent — CORE-2's HSRP and IP SLA did not depend on the primary FortiGate being reachable, and the secondary FortiGate did not depend on CORE-1 being active. This validates the design goal of layered redundancy with no shared state between mechanisms.

The 5-packet HSRP loss is the most visible limitation of this design. It is a direct consequence of default timers and is not hidden or minimized in this documentation. Any deployment with latency-sensitive workloads should evaluate BFD-assisted HSRP or subsecond timer tuning before production use.
