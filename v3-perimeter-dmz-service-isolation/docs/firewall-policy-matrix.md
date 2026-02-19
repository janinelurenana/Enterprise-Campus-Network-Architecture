# Firewall Policy Matrix

**Project:** v3 – Perimeter Hardening & Service Isolation

---

## 1. Security Design Objective

This firewall configuration enforces strict service isolation between:

* Public-facing systems (DMZ)
* Internal business users
* Sensitive internal services (HR File Server)

The design follows a **default-deny security posture**, where all inter-zone communication is explicitly permitted based on business requirement.

No implicit trust exists between zones.

---

## 2. Zone Architecture

| Zone          | VLANs                                         | Role                       |
| ------------- | --------------------------------------------- | -------------------------- |
| ZONE_INTERNAL | 10 (IT), 20 (Management), 30 (HR), 40 (Guest) | Trusted internal network   |
| ZONE_DMZ      | 50                                            | Public service segment     |
| ZONE_WAN      | ISP                                           | Untrusted external network |

Zone separation is enforced at the firewall.
Inter-VLAN controls inside ZONE_INTERNAL are further restricted using core ACLs.

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

External access is limited strictly to published services in the DMZ.

---

### DMZ-Originated Traffic

| Source   | Destination   | Service              | Action | Justification                     |
| -------- | ------------- | -------------------- | ------ | --------------------------------- |
| ZONE_DMZ | ZONE_INTERNAL | Any                  | Deny   | Prevent pivoting/lateral movement |
| ZONE_DMZ | ZONE_WAN      | Established sessions | Allow  | Return traffic only               |

The DMZ is treated as a semi-trusted zone and is not allowed to initiate communication into internal segments.

---

### Internal-Originated Traffic

| Source          | Destination              | Service                 | Action             | Justification              |
| --------------- | ------------------------ | ----------------------- | ------------------ | -------------------------- |
| IT / Management | HR File Server (VLAN 30) | File Services           | Allow              | Authorized HR access       |
| Guest VLAN      | HR File Server           | Any                     | Deny               | Protect sensitive data     |
| ZONE_INTERNAL   | ZONE_WAN                 | Internet services (NAT) | Allow              | Business requirement       |
| ZONE_INTERNAL   | ZONE_DMZ                 | Restricted/Admin only   | Allow (controlled) | Administrative access only |

No internal VLAN has blanket access to all internal resources.
HR access is restricted by VLAN and enforced via ACL + firewall policy.

---

## 5. HR File Server Protection Model

The HR file server (VLAN 30) is considered a sensitive internal asset.

Protection mechanisms include:

* VLAN-based segmentation
* Core switch ACL enforcement
* Firewall policy restrictions
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

This layered enforcement reduces risk of unauthorized access and limits blast radius in case of compromise.

---

## 6. Logging & Inspection

The firewall logs:

* WAN → DMZ session creation
* Blocked WAN → INTERNAL attempts
* Blocked DMZ → INTERNAL attempts
* Guest → HR denied attempts

This provides visibility into policy enforcement and validates segmentation integrity.

---

## 7. Implicit Deny Baseline

All traffic not explicitly allowed is denied by default.

This ensures:

* No unintended exposure
* No accidental trust relationships
* No unverified service paths

Policy order is structured to ensure specific rules are evaluated before broader internet access policies.

---

## 8. Security Outcome

This implementation achieves:

* Strict DMZ isolation
* Protection of internal VLANs from external and semi-trusted zones
* Controlled HR file server access
* Enforced trust boundaries between all security zones
* Reduced attack surface through intentional service exposure

---

## 9. Professional Relevance

This project demonstrates applied understanding of:

* Zone-based firewall architecture
* DMZ deployment models
* Inter-VLAN segmentation
* Policy-based access control
* Security-first network design
* Lateral movement containment strategies

The configuration reflects real-world enterprise perimeter design principles implemented in a lab environment.
