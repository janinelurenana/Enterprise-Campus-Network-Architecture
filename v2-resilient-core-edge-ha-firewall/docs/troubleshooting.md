# Troubleshooting Log

This document captures real failure scenarios encountered during the implementation of High Availability (HA) and Core Redundancy in **v2-resilient-core-edge-ha-firewall**.

---

# Case 1: Routing Loop (TTL Expired)

## Symptoms

* End devices (VPCs) unable to reach Internet (e.g., `8.8.8.8`)
* `tracert` / `mtr` shows packets bouncing between:

  * Core Switch
  * FortiGate
* Ping output:

  ```
  ICMP type: 11, code: 0
  TTL expired in transit
  ```

---

## Topology Context

* Two Core switches (HSRP enabled)
* Two FortiGate firewalls (Active/Standby HA)
* Four /30 transit links between Cores and Firewalls
* FortiGate WAN (port1) receives default route via DHCP from ISP/Cloud
* Core switches configured with:

  ```
  ip route 0.0.0.0 0.0.0.0 <FortiGate-transit-IP>
  ```

---

## Root Cause Analysis (RCA)

The issue was caused by a **Static Route Conflict**.

### What Happened

The FortiGate was manually configured with a static default route:

```
0.0.0.0/0 → Core Switch
```

At the same time:

* Core switches had their own default route:

  ```
  0.0.0.0/0 → FortiGate
  ```

This created a bidirectional default route loop.

---

## Packet Flow Breakdown

1. VPC sends traffic to default gateway (HSRP VIP on Core).
2. Core forwards traffic using its default route → FortiGate.
3. FortiGate matches its static default route → sends traffic back to Core.
4. Packet loops indefinitely between Core and FortiGate.
5. TTL decrements to 0.
6. ICMP Type 11 (TTL expired) returned.

This is a textbook Layer 3 routing loop.

---

## Resolution

* Removed manual static default route from FortiGate.
* Allowed FortiGate to use WAN-learned default route via DHCP.
* Verified routing table using:

  ```
  get router info routing-table all
  ```

After correction:

* Core default → FortiGate
* FortiGate default → WAN

Loop resolved immediately.

---

## Lesson Learned

In dual-device edge designs:

* Only one device should own the upstream default route toward the ISP.
* Transit networks must not be used as fallback default routes unless explicitly required.
* Always validate routing tables on both ends when troubleshooting loops.

---

# Case 2: Sub-Optimal Path Selection (Full-Mesh Topology)

## Symptoms

In a topology with:

* 2 Core Switches
* 2 FortiGate Firewalls
* Full-mesh /30 transit links

Traffic occasionally followed a non-direct path:

Example:

* Core-1 → FG-2
  Even though:
* Core-1 → FG-1 link was healthy.

No outage occurred, but traffic symmetry was inconsistent.

---

## Root Cause Analysis (RCA)

FortiOS static routes had:

* Same distance
* Same priority

When multiple static routes have equal distance and priority, FortiOS may load-balance or select routes unpredictably.

This resulted in asymmetric or sub-optimal forwarding paths.

---

## Resolution: Cascading Priority Design

FortiOS uses:

1. Distance (Administrative Distance)
2. Priority (Tie-breaker)
3. Interface/ECMP behavior

To enforce deterministic failover behavior, we implemented:

### Cascading Priority

Example:

| Route          | Distance | Priority |
| -------------- | -------- | -------- |
| Primary Path   | 10       | 10       |
| Secondary Path | 10       | 20       |
| Tertiary Path  | 10       | 30       |

Lower priority value = preferred route.

This creates a strict, ordered failover hierarchy.

---

## Result

* Primary path always selected when healthy.
* Backup path used only when primary fails.
* Traffic symmetry restored.
* Predictable behavior under failover conditions.

---

## Lesson Learned

In redundant, full-mesh topologies:

* Equal-cost static routes can create non-deterministic behavior.
* Always explicitly define route priority.
* Redundancy without control can create instability.

---

# Engineering Reflection

These incidents reinforced key architectural principles:

* Redundancy increases complexity.
* Default routes must be carefully controlled.
* Deterministic path selection is essential in multi-link topologies.
* Always verify actual forwarding behavior — not just interface status.

High Availability is not achieved by adding devices.

It is achieved by controlling behavior under failure.
