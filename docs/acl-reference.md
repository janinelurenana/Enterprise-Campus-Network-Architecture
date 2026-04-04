# ACL Reference

## Overview

Extended named ACLs are applied inbound on each core switch SVI. This placement intercepts traffic as it enters the routed domain — before any forwarding decision is made — ensuring that denied traffic is dropped at the earliest possible point and never forwarded toward its intended destination.

ACLs are stateless. They do not track TCP sessions. Return traffic (responses to outbound connections) must be explicitly permitted. Both core switches carry identical ACL definitions because either may be the HSRP-active gateway for any VLAN at any given time.

All ACLs end with an explicit `deny ip any any log`. This is redundant with IOS's implicit deny but serves two purposes: it produces log entries for unexpected traffic (which the implicit deny does not), and it documents intent — a reviewer reading the config knows the deny is deliberate, not an omission.

---

## Application Points

| ACL Name       | Applied To   | Direction | Device         |
|----------------|--------------|-----------|----------------|
| ACL_IT_IN      | Vlan10 SVI   | inbound   | CORE-1, CORE-2 |
| ACL_MGMT_IN    | Vlan20 SVI   | inbound   | CORE-1, CORE-2 |
| ACL_HR_IN      | Vlan30 SVI   | inbound   | CORE-1, CORE-2 |
| ACL_GUEST_IN   | Vlan40 SVI   | inbound   | CORE-1, CORE-2 |

Inbound on SVI means the ACL evaluates traffic arriving *from* the VLAN — i.e., traffic sourced by hosts in that VLAN heading toward the router. This is the correct placement for enforcing source-based policy.

---

## Return Traffic Rules

Every ACL begins with two return-path permits. These are listed first to ensure the lowest possible processing overhead for established session traffic, which is the majority of flows.

```
permit icmp any any echo-reply
permit tcp any any established
```

**`permit icmp any any echo-reply`**
Allows ICMP echo-reply packets back into the VLAN. Without this, a host could initiate a ping but never receive the response. The source is `any` because reply packets arrive from the remote host (internet, another VLAN, etc.) — not from the local subnet.

**`permit tcp any any established`**
Allows TCP packets with the ACK or RST flag set (IOS definition of "established"). This covers return traffic for any TCP connection initiated by a host in this VLAN. The stateless nature of this rule means a crafted packet with the ACK bit set could technically pass — this is a known limitation of stateless ACLs and is documented in the design decisions.

---

## ACL_IT_IN — VLAN 10 (IT)

IT is the highest-trust department. IT hosts have broad internal access (Management and HR) and unrestricted internet access. IT is blocked from Guest.

**Rationale:** IT administers the infrastructure and requires access to Management for operational overlap and to HR for server and file system access. Blocking IT from Guest prevents IT hosts from being used as pivot points into untrusted space (and prevents a compromised IT host from being used to attack Guest devices, which might be easier targets).

```
permit icmp any any echo-reply
permit tcp any any established
deny   ip 10.10.10.0 0.0.0.255 10.10.40.0 0.0.0.255 log
permit ip 10.10.10.0 0.0.0.255 10.10.20.0 0.0.0.255
permit ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255
permit ip 10.10.10.0 0.0.0.255 any
deny   ip any any log
```

| Rule | Match | Action | Reason |
|------|-------|--------|--------|
| 1 | ICMP echo-reply, any source → any dest | Permit | Return pings |
| 2 | TCP established, any → any | Permit | Return TCP sessions |
| 3 | IT → Guest (10.10.40.0/24) | Deny + log | IT must not reach Guest |
| 4 | IT → Management (10.10.20.0/24) | Permit | Operational access |
| 5 | IT → HR (10.10.30.0/24) | Permit | Infrastructure access |
| 6 | IT → any | Permit | Internet and other destinations |
| 7 | Any → any | Deny + log | Explicit default deny |

**Why is Guest denied before the broad `permit any`?**
ACLs are evaluated top-down, first match wins. The `permit ip 10.10.10.0 ... any` at rule 6 would match Guest-destined traffic if Guest weren't denied first. The explicit deny at rule 3 must appear before the broad permit.

---

## ACL_MGMT_IN — VLAN 20 (Management)

Management has access to HR and the internet. Management is blocked from IT and Guest.

**Rationale:** Management (executive/ops) needs visibility into HR data but should not have direct access to IT infrastructure — that access path should go through IT personnel, not be available to management hosts directly. This reduces the impact of a compromised Management host. Guest access is denied for the same reasons as other VLANs.

```
permit icmp any any echo-reply
permit tcp any any established
deny   ip 10.10.20.0 0.0.0.255 10.10.10.0 0.0.0.255 log
deny   ip 10.10.20.0 0.0.0.255 10.10.40.0 0.0.0.255 log
permit ip 10.10.20.0 0.0.0.255 10.10.30.0 0.0.0.255
permit ip 10.10.20.0 0.0.0.255 any
deny   ip any any log
```

| Rule | Match | Action | Reason |
|------|-------|--------|--------|
| 1 | ICMP echo-reply | Permit | Return pings |
| 2 | TCP established | Permit | Return TCP sessions |
| 3 | Management → IT (10.10.10.0/24) | Deny + log | No direct infra access |
| 4 | Management → Guest (10.10.40.0/24) | Deny + log | No access to untrusted segment |
| 5 | Management → HR (10.10.30.0/24) | Permit | Authorized data access |
| 6 | Management → any | Permit | Internet access |
| 7 | Any → any | Deny + log | Explicit default deny |

---

## ACL_HR_IN — VLAN 30 (HR)

HR has internet access only. HR is blocked from IT, Management, and Guest.

**Rationale:** HR handles sensitive personnel data. Restricting HR from reaching other internal VLANs limits the blast radius if an HR host is compromised — an attacker cannot pivot laterally from HR to IT or Management. HR staff have no operational need to initiate connections to those VLANs; any required data sharing should be handled by server-side access controls on the HR file server itself, not by allowing HR workstations to reach other subnets.

```
permit icmp any any echo-reply
permit tcp any any established
deny   ip 10.10.30.0 0.0.0.255 10.10.10.0 0.0.0.255 log
deny   ip 10.10.30.0 0.0.0.255 10.10.20.0 0.0.0.255 log
deny   ip 10.10.30.0 0.0.0.255 10.10.40.0 0.0.0.255 log
permit ip 10.10.30.0 0.0.0.255 any
deny   ip any any log
```

| Rule | Match | Action | Reason |
|------|-------|--------|--------|
| 1 | ICMP echo-reply | Permit | Return pings |
| 2 | TCP established | Permit | Return TCP sessions |
| 3 | HR → IT (10.10.10.0/24) | Deny + log | No infra access |
| 4 | HR → Management (10.10.20.0/24) | Deny + log | No cross-department access |
| 5 | HR → Guest (10.10.40.0/24) | Deny + log | No access to untrusted segment |
| 6 | HR → any | Permit | Internet access only |
| 7 | Any → any | Deny + log | Explicit default deny |

---

## ACL_GUEST_IN — VLAN 40 (Guest)

Guest has internet access only. Guest is denied access to all internal subnets (10.10.0.0/16 summary).

**Rationale:** Guest endpoints are untrusted by definition — they may be personal devices, contractor laptops, or attacker-controlled machines. A single deny covering the entire 10.10.0.0/16 range (rather than individual /24 denies) ensures that any new internal subnet added in the future is automatically blocked from Guest without requiring an ACL update. This is the only ACL that uses a summary match for the deny; it is appropriate here because the policy intent is total isolation from anything internal.

```
permit icmp any any echo-reply
permit tcp any any established
deny   ip 10.10.40.0 0.0.0.255 10.10.0.0 0.0.255.255 log
permit ip 10.10.40.0 0.0.0.255 any
deny   ip any any log
```

| Rule | Match | Action | Reason |
|------|-------|--------|--------|
| 1 | ICMP echo-reply | Permit | Return pings |
| 2 | TCP established | Permit | Return TCP sessions |
| 3 | Guest → 10.10.0.0/16 (all internal) | Deny + log | Total internal isolation |
| 4 | Guest → any | Permit | Internet access |
| 5 | Any → any | Deny + log | Explicit default deny |

**Why use 0.0.255.255 (a /16 wildcard) instead of listing each /24?**
The other ACLs deny specific /24 subnets because the policy is nuanced — IT is denied Guest but permitted Management and HR. Guest policy has no nuance: nothing internal is reachable. A /16 summary is cleaner, harder to misconfigure, and future-safe.

---

## East-West Policy Matrix

The following matrix is derived directly from the ACL rules. ✅ = permitted, ❌ = denied.

| Source \ Destination | IT (10) | Mgmt (20) | HR (30) | Guest (40) | Internet |
|----------------------|---------|-----------|---------|------------|----------|
| IT (10)              | —       | ✅         | ✅       | ❌          | ✅        |
| Management (20)      | ❌       | —         | ✅       | ❌          | ✅        |
| HR (30)              | ❌       | ❌         | —       | ❌          | ✅        |
| Guest (40)           | ❌       | ❌         | ❌       | —          | ✅        |

Note: "Internet" means any destination outside 10.10.0.0/16. This includes the default route toward the FortiGate. The FortiGate then applies its own policy (INTERNAL_to_WAN) which further restricts outbound services to HTTP, HTTPS, DNS, and PING.

---

## Operational Notes

**ACL hit counters.** Use `show ip access-lists` on either core switch to view match counts per rule. During validation, clear counters first (`clear ip access-list counters`) then run test traffic and inspect which rules incremented. This is the primary method for confirming that the correct rule matched a given flow.

**Logging.** All deny rules include the `log` keyword. Denied traffic generates syslog messages with source IP, destination IP, protocol, and ports. In production, these should be shipped to a syslog server. In the lab, they are visible via `show logging` or `debug ip packet` (use with caution on a production device).

**Adding a new VLAN.** Any new internal VLAN added to the network requires:
1. A new ACL defining its access policy.
2. The new subnet added to deny rules in existing ACLs where appropriate (e.g., if a new Finance VLAN at 10.10.50.0/24 should be blocked from Guest, `ACL_GUEST_IN` already covers it via the /16 summary — but `ACL_HR_IN` would need an explicit deny for Finance if HR should not reach it).
3. Application of the new ACL inbound on the new SVI on both CORE-1 and CORE-2.

**Modifying existing rules.** IOS named ACLs support `ip access-list extended ACL_NAME` followed by `no <sequence-number>` and insertion of new rules at specific sequence numbers. This avoids the need to delete and recreate the entire ACL during changes.
