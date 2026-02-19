# DMZ Design

**Project:** v3 – Perimeter Hardening & Service Isolation

---

## 1. Overview

The DMZ (Demilitarized Zone) is implemented to **host public-facing services** while protecting internal resources.

Goals:

* Expose web services without compromising internal networks
* Enforce strict trust boundaries between DMZ and internal zones
* Prevent lateral movement in case of compromise
* Provide controlled outbound access for DMZ servers (e.g., DNS, updates)

---

## 2. Network Placement

| Component        | VLAN | Zone                                | Description                      |
| ---------------- | ---- | ----------------------------------- | -------------------------------- |
| Nginx Web Server | 50   | ZONE_DMZ                            | Public-facing web services       |
| DMZ Switch       | 50   | ZONE_DMZ                            | Dedicated L2 switch for DMZ      |
| Firewall         | N/A  | ZONE_DMZ / ZONE_INTERNAL / ZONE_WAN | Enforces all inter-zone policies |

* DMZ is connected to a dedicated L2 switch to **physically and logically separate** it from internal VLANs.
* Firewall mediates all traffic to/from DMZ.

---

## 3. Security Model

The DMZ follows a **semi-trusted security stance**:

* Outbound connectivity: restricted to required services (DNS, HTTPS, PING)
* Inbound connectivity: only allowed to published services via VIP objects (HTTP/HTTPS)
* No DMZ → INTERNAL communication permitted
* Logged inter-zone traffic ensures visibility

Trust boundaries:

```
WAN → DMZ: Allowed (HTTP/HTTPS)
DMZ → INTERNAL: Deny
DMZ → WAN: Controlled (DNS, HTTPS)
```

> Any compromise in the DMZ is contained; internal assets are not exposed.

---

## 4. Integration with Internal Zones

* Internal users (IT, Management) can reach DMZ for administration only
* Guest VLANs have **no access** to DMZ
* HR and other sensitive VLANs remain strictly internal

The DMZ acts as a **buffer between untrusted WAN and trusted internal zones**.

---

## 5. Policy Enforcement

* Firewall policies enforce inbound/outbound rules and deny unauthorized traffic
* VLAN segmentation on core/access switches ensures traffic separation
* Explicit deny rules for DMZ → INTERNAL and WAN → INTERNAL traffic

Example controls:

| Source   | Destination  | Action             | Notes                        |
| -------- | ------------ | ------------------ | ---------------------------- |
| WAN      | DMZ Web VIPs | Allow              | Public web exposure          |
| DMZ      | INTERNAL     | Deny               | Lateral movement containment |
| DMZ      | WAN          | Allow (limited)    | DNS, HTTPS, PING             |
| INTERNAL | DMZ          | Allow (restricted) | Admin access only            |

---

## 6. Design Rationale

* Dedicated VLAN and switch **simplifies monitoring and containment**
* Explicit deny rules reduce risk of **accidental trust paths**
* Restricting outbound traffic minimizes potential data exfiltration
* Logging provides **audit trail and enforcement visibility**

This design mirrors enterprise-standard DMZ architecture suitable for small-scale production networks.

---

## 7. Verification

The following have been tested and verified:

* WAN → DMZ (HTTP/HTTPS) successful
* DMZ → INTERNAL denied
* DMZ → WAN limited to allowed protocols
* Internal → DMZ admin access successful
* Logging captures all inter-zone traffic

Evidence is available in the `verification/` folder.

---

## 8. Security Outcome

* DMZ services are **isolated from internal networks**
* Internal assets (HR, IT, Management) remain protected
* Trust boundaries are clearly defined and enforced
* Blast radius of compromise is minimized

