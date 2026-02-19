# v3 – Perimeter Hardening & Service Isolation (DMZ Implementation)

## Overview

This phase introduces perimeter hardening through DMZ isolation and stricter inter-zone firewall enforcement.

Unlike previous versions focused on segmentation (v1) and high availability (v2), this version focuses on:

* Controlled public service exposure
* Clear trust boundaries between internal and exposed systems
* Enforcement of deny-by-default inter-zone traffic
* Protection of sensitive internal resources (HR File Server)

---

## Architectural Goals

1. Isolate public-facing services from internal infrastructure
2. Prevent lateral movement between DMZ and internal networks
3. Restrict HR file server access to authorized internal VLANs
4. Enforce explicit security policies at the firewall
5. Validate segmentation through end-to-end testing

---

## Zone Design

### Firewall Zones

| Zone          | VLANs          | Purpose                    |
| ------------- | -------------- | -------------------------- |
| ZONE_INTERNAL | 10, 20, 30, 40 | Internal users and servers |
| ZONE_DMZ      | 50             | Public-facing services     |
| ZONE_WAN      | N/A            | External network / ISP     |

### VLAN Breakdown

| VLAN | Role                   |
| ---- | ---------------------- |
| 10   | IT                     |
| 20   | Management             |
| 30   | HR (File Server)       |
| 40   | Guest                  |
| 50   | DMZ (Nginx Web Server) |

---

## Network Design Model

This implementation follows a **hub-and-spoke security model** with firewall-enforced zone separation.

* Core switch performs inter-VLAN routing
* ACLs restrict internal lateral access where required
* FortiGate firewall enforces zone-to-zone policies
* DMZ connected to a dedicated Layer 2 switch
* Default posture: deny unless explicitly permitted

Trust boundaries are enforced at the firewall, not the switch fabric.

---

## Service Placement Strategy

### DMZ (VLAN 50)

* Hosts public Nginx web server
* Accessible from WAN via controlled firewall policy
* No direct access to internal VLANs

### HR File Server (VLAN 30)

* Hosted within ZONE_INTERNAL
* Accessible only from:

  * VLAN 10 (IT)
  * VLAN 20 (Management)
* Blocked from:

  * VLAN 40 (Guest)
  * ZONE_DMZ
  * ZONE_WAN

Segmentation is VLAN-based and enforced using core ACLs and firewall policies.

---

## Firewall Policy Model

| Source   | Destination | Action                   | Purpose                     |
| -------- | ----------- | ------------------------ | --------------------------- |
| WAN      | DMZ         | Allow (HTTP/HTTPS)       | Public web access           |
| WAN      | INTERNAL    | Deny                     | Protect internal assets     |
| INTERNAL | HR          | Allow (restricted VLANs) | Authorized access           |
| GUEST    | HR          | Deny                     | Prevent unauthorized access |
| DMZ      | INTERNAL    | Deny                     | Prevent lateral movement    |
| INTERNAL | WAN         | Allow (controlled)       | Internet access             |

Implicit deny exists at the bottom of the policy table.

---

## Security Posture

This version enforces:

* Strict inter-zone inspection
* Logging of inter-zone traffic
* Clear separation of exposed and internal services
* Reduced blast radius in case of DMZ compromise

If the DMZ web server is compromised, firewall policy prevents pivoting into internal VLANs.

---

## Verification & Testing

The following scenarios were validated:

✔ WAN → DMZ (HTTP/HTTPS) successful
✔ WAN → INTERNAL blocked
✔ Guest VLAN → HR server denied
✔ IT/Management → HR server successful
✔ DMZ → INTERNAL denied
✔ Firewall logs confirm policy enforcement

All verification evidence is included in the `verification/` directory.

---

## Key Changes from v2

* Added VLAN 30 (HR segment)
* Added VLAN 50 (DMZ)
* Implemented dedicated DMZ switch
* Updated firewall zone policies
* Updated core ACLs
* Introduced public service exposure model

---

## Lessons Learned

* Segmentation without strict firewall policy is incomplete
* Service exposure must be intentional and minimal
* VLAN separation alone is insufficient without enforcement
* Clear trust boundary documentation improves architecture clarity
* DMZ placement significantly reduces internal attack surface

---

## Summary

v3 transitions the lab environment from simple segmentation and redundancy into a perimeter-hardened architecture.

Public services are isolated.
Internal services are protected.
Trust boundaries are clearly defined and enforced.

This phase demonstrates practical service isolation using VLAN segmentation, ACL control, and firewall zone policy enforcement.
