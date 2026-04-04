# Firewall Policy Matrix

**Project:** v3 – Perimeter Hardening & Service Isolation

---

## 1. Security Design Objective

This configuration enforces strict service isolation between:

* Public-facing systems (DMZ)
* Internal business users
* Sensitive internal services (HR File Server)

The design follows a **default-deny security posture**, where all inter-zone communication is explicitly permitted based on business requirements.

No implicit trust exists between zones.

---

## 2. Zone Architecture

| Zone          | VLANs                                         | Role                       | Enforcement                                                                                                     |
| ------------- | --------------------------------------------- | -------------------------- | --------------------------------------------------------------------------------------------------------------- |
| ZONE_INTERNAL | 10 (IT), 20 (Management), 30 (HR), 40 (Guest) | Trusted internal network   | Inter-VLAN ACLs enforced at **core switch**; firewall enforces cross-zone restrictions (e.g., guest → internal) |
| ZONE_DMZ      | 50                                            | Public service segment     | Firewall only                                                                                                   |
| ZONE_WAN      | ISP                                           | Untrusted external network | Firewall only                                                                                                   |

> Internal ACLs handle **VLAN-level segmentation** for internal traffic; firewall enforces inter-zone boundaries and prevents unauthorized access from DMZ or WAN.

---

## 3. Security Principles Applied

* Least privilege access
* Explicit allow rules
* Implicit deny-all baseline
* Lateral movement prevention
* Service-based rule restriction (not “any-any”)
* Logged inter-zone enforcement

This configuration prioritizes containment over convenience.

---

## 4. Inter-Zone Access Matrix

### WAN-Originated Traffic

| Source   | Destination   | Service    | Action | Justification               |
| -------- | ------------- | ---------- | ------ | --------------------------- |
| ZONE_WAN | ZONE_DMZ      | HTTP/HTTPS | Allow  | Public web service exposure |
| ZONE_WAN | ZONE_INTERNAL | Any        | Deny   | Protect internal assets     |

---

### DMZ-Originated Traffic

| Source   | Destination   | Service              | Action | Justification                     |
| -------- | ------------- | -------------------- | ------ | --------------------------------- |
| ZONE_DMZ | ZONE_INTERNAL | Any                  | Deny   | Prevent pivoting/lateral movement |
| ZONE_DMZ | ZONE_WAN      | Established sessions | Allow  | Return traffic only               |

> DMZ is semi-trusted; it cannot initiate communication into internal segments.

---

### Internal-Originated Traffic

| Source          | Destination               | Service                 | Action             | Justification                                      |
| --------------- | ------------------------- | ----------------------- | ------------------ | -------------------------------------------------- |
| IT / Management | HR File Server (VLAN 30)  | File Services           | Allow              | Authorized HR access (ACL enforced at core switch) |
| Guest VLAN      | Internal VLANs (10,20,30) | Any                     | Deny               | Firewall prevents guest → internal access          |
| ZONE_INTERNAL   | ZONE_WAN                  | Internet services (NAT) | Allow              | Business requirement                               |
| ZONE_INTERNAL   | ZONE_DMZ                  | Restricted/Admin only   | Allow (controlled) | Administrative access only                         |

> Note: VLAN 10, 20, 30, 40 internal segmentation is **enforced at the core switch**; the firewall prevents guests or other zones from bypassing internal ACLs.

---

## 5. HR File Server Protection Model

The HR file server (VLAN 30) is considered a sensitive internal asset.

Protection mechanisms include:

* VLAN-based segmentation (core switch ACLs)
* Firewall policy restrictions to prevent cross-zone exposure
* Explicit deny from Guest VLAN
* No exposure to WAN
* No access from DMZ

### Access Summary

| Source               | HR Access |
| -------------------- | --------- |
| VLAN 10 (IT)         | Allowed   |
| VLAN 20 (Management) | Allowed   |
| VLAN 40 (Guest)      | Denied    |
| ZONE_DMZ             | Denied    |
| ZONE_WAN             | Denied    |

> Layered enforcement minimizes the blast radius of any compromise and ensures sensitive HR data remains protected.

---

## 6. Logging & Inspection

The firewall logs:

* WAN → DMZ session creation
* Blocked WAN → INTERNAL attempts
* Blocked DMZ → INTERNAL attempts
* Guest → HR denied attempts

> Core switch ACLs log internal segmentation violations where supported; firewall logging ensures inter-zone access enforcement is visible.

---

## 7. Implicit Deny Baseline

All traffic not explicitly allowed is denied by default, ensuring:

* No unintended exposure
* No accidental trust relationships
* No unverified service paths

Rule order ensures **specific allow rules evaluated before broader internet access policies**.

---

## 8. Security Outcome

* Strict DMZ isolation from internal networks
* Internal VLANs segmented via core ACLs, with firewall preventing bypass
* Controlled HR file server access
* Enforced trust boundaries between all security zones
* Reduced attack surface through deliberate service exposure
