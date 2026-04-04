# Firewall Policy: North–South Control

## Purpose

This document defines the **north–south security policy** enforced by the FortiGate firewall in this lab. The firewall acts as the **only trust boundary** between internal VLANs and the Internet.

East–west (inter-VLAN) traffic is intentionally **not** handled here and is enforced at the core switch using ACLs.

---

## Firewall Role in the Architecture

* Stateful inspection for Internet-bound traffic
* NAT for outbound connectivity
* Enforcement of guest isolation
* Logging and visibility of security decisions

The firewall does **not** perform:

* Inter-VLAN routing
* Internal segmentation
* East–west policy enforcement

---

## Interface Trust Model

| Interface | Zone          | Trust Level | Description                |
| --------: | ------------- | ----------- | -------------------------- |
|     port1 | ZONE_WAN      | Untrusted   | Internet / Cloud           |
|     port2 | ZONE_INTERNAL | Trusted     | Core L3 switch (all VLANs) |

All internal VLANs enter the firewall through a single trusted interface. Logical separation is achieved through address objects and policy, not subinterfaces.

---

## Address Objects

The following address objects represent routed VLANs behind the core switch:

* IT_SUBNET — 10.10.10.0/24
* MGMT_SUBNET — 10.10.20.0/24
* HR_SUBNET — 10.10.30.0/24
* GUEST_SUBNET — 10.10.40.0/24

These objects allow policies to remain readable, scalable, and auditable.

---

## Routing Dependencies

The firewall relies on static routing for reachability:

* Default route → WAN (port1)
* Internal summary route → 10.10.0.0/16 via Core (port2)

Without the internal route, return traffic would be dropped regardless of policy.

---

## NAT Strategy

* NAT is enabled **only** for traffic exiting to the WAN
* No internal or inter-VLAN NAT is performed

This ensures:

* Predictable internal addressing
* Simplified troubleshooting
* Clean separation of routing and translation responsibilities

---

## Policy Evaluation Model

Policies are evaluated **top-down, first match wins**.

Rules are ordered from:

1. Most specific and restrictive
2. To more general Internet access

Implicit deny applies to all traffic not explicitly permitted.

---

## Firewall Policy Table

| Order | Policy Name            | Source       | Destination | Service          | Action | Notes            |
| ----: | ---------------------- | ------------ | ----------- | ---------------- | ------ | ---------------- |
|     1 | IT_to_WAN              | IT_SUBNET    | Internet    | ANY              | ALLOW  | Full access      |
|     2 | MGMT_to_WAN            | MGMT_SUBNET  | Internet    | HTTP, HTTPS, DNS | ALLOW  | Restricted       |
|     3 | HR_to_WAN              | HR_SUBNET    | Internet    | HTTP, HTTPS, DNS | ALLOW  | Restricted       |
|     4 | GUEST_to_WAN           | GUEST_SUBNET | Internet    | HTTP, HTTPS, DNS | ALLOW  | Internet-only    |
|     5 | GUEST_to_INTERNAL_DENY | GUEST_SUBNET | Internal    | ANY              | DENY   | Defense-in-depth |

All allow rules perform source NAT.

---

## Guest Traffic Handling

Guest traffic is treated as **untrusted** despite entering via the internal interface:

* Blocked from all internal subnets
* Allowed Internet access only
* Logged explicitly for visibility

Guest isolation is enforced at **two layers**:

1. Core ACLs (east–west)
2. Firewall deny policy (north–south)

This layered approach prevents policy bypass.

---

## Logging and Visibility

* All firewall policies log traffic decisions
* Deny rules provide confirmation of enforcement
* Logs are used for validation, not alerting

Logging confirms policy behavior without overwhelming signal.

---

## Security Rationale


   * **Distributed Enforcement:** By using Core ACLs for east-west traffic and the FortiGate for north-south, we avoid the "hairpinning" effect, where internal traffic unnecessarily consumes firewall CPU cycles.

   * **Stateless + Stateful Layering:** Core ACLs provide fast, stateless drops. The firewall provides stateful inspection, ensuring that even if a packet is "allowed" by a basic ACL, its behavior must be valid for the protocol.

   * **Guest "Double-Lock":** Guest traffic is blocked at the Core (VLAN 40 SVI) and explicitly denied again at the Firewall, ensuring total isolation even in the event of a configuration error at one layer.

---

## Summary

The FortiGate firewall enforces a clear and minimal north–south policy:

* Internal users reach the Internet based on role
* Guest users reach only the Internet
* No internal VLAN relies on the firewall for segmentation

This separation of responsibility simplifies operations, improves security, and scales cleanly.
