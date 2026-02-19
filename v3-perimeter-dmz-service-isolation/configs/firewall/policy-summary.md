# Firewall Policy Summary

**Project:** v3 – Perimeter Hardening & Service Isolation

---

## Policy Model

The firewall enforces strict zone-based segmentation between:

* **ZONE_INTERNAL** (VLAN 10, 20, 30, 40)
* **ZONE_DMZ** (VLAN 50)
* **ZONE_WAN** (ISP)

Security posture: **explicit allow, implicit deny**.
All inter-zone traffic is logged.

---

## Core Enforcement Rules

### 1. Internal → Internet

* Allow HTTP, HTTPS, DNS, PING
* NAT enabled
* Applies to internal VLANs

Provides controlled outbound access while preventing internal IP exposure.

---

### 2. WAN → DMZ (Web Only)

* Allow HTTP/HTTPS via VIP objects
* No other DMZ services exposed

Publishes the Nginx web server while keeping other services private.

---

### 3. WAN → Internal

* Deny all
* Logged

Prevents direct exposure of internal subnets.

---

### 4. DMZ → Internal

* Deny all
* Logged

Prevents lateral movement if the DMZ server is compromised.

---

### 5. Guest → Internal (IT / Mgmt / HR)

* Deny all
* Logged

Enforces internal segmentation and protects sensitive VLANs from untrusted devices.

---

### 6. DMZ → Internet

* Allow DNS, HTTPS, PING
* NAT enabled

Permits outbound updates and name resolution for the web server.

---

## Security Outcome

This rule set ensures:

* Public services are isolated in the DMZ
* No direct WAN access to internal networks
* Guest VLAN is contained
* HR file server remains internal-only
* Lateral movement between zones is restricted
* All critical deny paths are logged

No broad “any-to-any” trust exists between security zones.

