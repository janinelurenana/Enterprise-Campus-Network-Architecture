# HA Logic Explained

## Design Philosophy

The network implements layered redundancy across:

* **Core Layer:** Dual Layer-3 switches using HSRP for default gateway redundancy
* **Edge Layer:** FortiGate Active-Passive (A-P) cluster for perimeter security
* **Interconnections:** Redundant cross-links between Core switches and Firewalls
* **Link Monitoring:** Interface health tracking to influence failover decisions

The goal is not simply device redundancy, but **control-plane and data-plane continuity** during hardware or link failure.

---

## Core Layer Redundancy (HSRP Logic)

### Default Gateway Redundancy

Each VLAN is configured with:

* A **Virtual IP (VIP)** (e.g., `10.10.X.1`)
* An **Active router** (CORE-1, priority 110)
* A **Standby router** (CORE-2, priority 100)
* **Preemption enabled** on both devices

Hosts use the Virtual IP as their default gateway.

### Normal Operation

* CORE-1 owns the HSRP virtual MAC address.
* All ARP requests for the VIP are answered by CORE-1.
* Traffic flows through CORE-1 toward the Firewall cluster.
* CORE-2 listens for HSRP hello packets.

### Failure Detection

If CORE-1 fails:

* CORE-2 stops receiving HSRP hello messages.
* After the hold timer expires, CORE-2 transitions to Active.
* CORE-2 assumes the virtual MAC address.
* Downstream switches update MAC tables.
* Traffic resumes via CORE-2.

No host configuration changes are required.

---

## Firewall Layer Redundancy (FortiGate HA Logic)

The firewall cluster operates in **Active-Passive (A-P)** mode using FGCP.

### Role Assignment

* **FG1**

  * Priority: 200
  * Override: Enabled
  * Default Primary

* **FG2**

  * Priority: 100
  * Override: Disabled
  * Default Secondary

The highest priority unit becomes Primary.

### Configuration Synchronization

* All firewall policies, interfaces, routes, and objects are synchronized from Primary to Secondary.
* The Secondary remains in standby state.
* Session tables are synchronized due to `session-pickup enable`.

The cluster behaves as a single logical firewall.

---

## Link Monitoring & Path Awareness

Each firewall monitors its connections to the Core layer using:

* `set monitor` under HA configuration
* `config system link-monitor`

Monitored interfaces:

* port2
* port3
* port4
* port5

If a monitored interface fails:

* The HA priority is adjusted.
* The cluster may trigger a failover.
* Traffic shifts to alternate physical paths.

This prevents a scenario where a firewall remains Primary but loses upstream connectivity.

---

## Static Routing Strategy

The firewalls use multiple static routes to the internal network (`10.10.0.0/16`) via:

* Direct link to Core-1
* Cross-link to Core-2
* Backup paths with higher administrative priority

### Logic:

* Primary routes have lower priority values.
* Backup routes activate automatically if the primary path fails.
* Upon HA role change, the new Primary activates its locally connected routes.

This ensures continuity even during:

* Core switch failure
* Firewall failure
* Single uplink failure

---

## Session Persistence During Failover

With `session-pickup enable`:

* The session table is synchronized between firewall nodes.
* When failover occurs:

  * The new Primary retains knowledge of active flows.
  * TCP and ICMP sessions continue with minimal interruption.
* Only detection delay (heartbeat timeout) contributes to packet loss.

This reduces user-visible impact.

---

## End-to-End Failover Flow

### Scenario: Firewall Primary Failure

1. FG1 stops sending heartbeat.
2. FG2 detects heartbeat loss.
3. FG2 becomes Primary.
4. FG2 activates its routing table.
5. Sessions continue using synchronized state.
6. Traffic resumes via alternate Core-facing interfaces.

---

### Scenario: Core Active Failure

1. CORE-1 stops sending HSRP hellos.
2. CORE-2 hold timer expires.
3. CORE-2 assumes virtual MAC.
4. Access switches update MAC entries.
5. Traffic flows through CORE-2 to the firewall cluster.

---

## Override & Role Recovery Behavior

Because override is enabled on FG1:

* If FG1 recovers,
* It will reclaim Primary role automatically (if priority remains highest).

For HSRP:

* Since preemption is enabled,
* CORE-1 reclaims Active status after recovery.

This ensures deterministic primary device control.

---

## Design Strengths

* No single point of failure in Core layer
* No single point of failure at Firewall
* Redundant L3 paths between layers
* Deterministic failback behavior
* Session-aware firewall failover
* Link-aware failover logic

---

## Design Trade-Offs

* HSRP default timers introduce ~5–10 second convergence delay
* Static routing requires manual scaling for larger environments
* Active-Passive firewall model does not load-balance traffic

For faster convergence:

* HSRP timers could be tuned
* BFD could be integrated
* Dynamic routing (OSPF) could replace static routes

---

## Summary

This HA implementation provides layered redundancy across both switching and security domains.

Failures at the Core or Firewall layer result in automatic role transition with minimal packet loss and no manual intervention required.

The design prioritizes stability, determinism, and clarity over complexity.
