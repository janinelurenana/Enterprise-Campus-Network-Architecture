# HR Segmentation Design

**Project:** v3 – Perimeter Hardening & Service Isolation

---

## 1. Overview

The HR File Server hosts sensitive personnel data and must be **strictly protected** from unauthorized access.

This design introduces **VLAN-based segmentation** combined with **firewall and ACL enforcement** to:

* Protect HR assets from untrusted networks (Guest VLAN, DMZ, WAN)
* Allow only authorized internal access (IT and Management VLANs)
* Ensure compliance with data confidentiality principles

---

## 2. Network Placement

| Component      | VLAN        | Zone                                | Description                |
| -------------- | ----------- | ----------------------------------- | -------------------------- |
| HR File Server | 30          | ZONE_INTERNAL                       | Stores sensitive HR data   |
| Core Switch    | 10,20,30,40 | ZONE_INTERNAL                       | Enforces inter-VLAN ACLs   |
| Firewall       | N/A         | ZONE_INTERNAL / ZONE_DMZ / ZONE_WAN | Enforces inter-zone access |

* HR VLAN (30) is **dedicated**, separated from IT, Management, and Guest VLANs.
* All traffic to HR VLAN passes through **core ACLs and firewall rules**, not left open by default.

---

## 3. Security Model

HR VLAN is treated as a **high-trust, sensitive internal segment**:

* Access is **least privilege**: only IT (VLAN 10) and Management (VLAN 20) can reach HR servers
* Guest VLAN (VLAN 40) and DMZ (VLAN 50) have **no access**
* WAN has **no direct access** to HR

This ensures **data confidentiality and segmentation integrity**.

---

## 4. Access Enforcement

| Source               | Destination  | Services      | Action | Notes                            |
| -------------------- | ------------ | ------------- | ------ | -------------------------------- |
| IT VLAN (10)         | HR VLAN (30) | File Services | Allow  | Authorized administrative access |
| Management VLAN (20) | HR VLAN (30) | File Services | Allow  | Authorized HR access             |
| Guest VLAN (40)      | HR VLAN (30) | Any           | Deny   | Prevents unauthorized access     |
| DMZ VLAN (50)        | HR VLAN (30) | Any           | Deny   | Prevents lateral movement        |
| WAN                  | HR VLAN (30) | Any           | Deny   | No external exposure             |

> All denied attempts are logged to provide visibility and support auditing.

---

## 5. Design Rationale

* **Dedicated VLAN** isolates HR traffic from other internal services.
* **ACL enforcement** on the core ensures only approved VLANs can communicate with HR.
* **Firewall rules** prevent unauthorized inter-zone traffic and enforce a **defense-in-depth** model.
* This approach **minimizes risk** while still enabling operational access for authorized users.

Intentional choices highlight that HR segmentation is **not accidental**, but a deliberate protection mechanism.

---

## 6. Integration with DMZ and WAN

* HR VLAN **cannot be reached** from the DMZ, even if the DMZ web server is compromised.
* All HR services remain internal-only; WAN exposure is blocked.
* This creates a clear **trust boundary** between sensitive data and untrusted networks.

---

## 7. Security Outcome

* HR assets are **segmented, monitored, and protected**
* Access is **restricted to authorized personnel**
* Risk of compromise via lateral movement is minimized
* Trust boundaries are clear and enforced
